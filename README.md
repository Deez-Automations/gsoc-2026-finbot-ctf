# GSoC 2026 — OWASP FinBot CTF: Final Work Report

**Contributor:** Muhammad Daniyal ([@Deez-Automations](https://github.com/Deez-Automations))
**Program:** Google Summer of Code 2026
**Organization:** OWASP Foundation
**Project:** OWASP GenAI Security — FinBot CTF: The Autonomous Threat Track
**Upstream repository:** [OWASP-ASI/finbot-ctf](https://github.com/OWASP-ASI/finbot-ctf)
**Mentor:** Rishi Mondal
**Status:** Midterm evaluation passed

---

## Summary

FinBot CTF is a training platform built around a simulated agentic finance system. Vendors, invoices, payments, and a set of LLM-powered agents that process all of it, wired up so players can practice real AI security exploits against something that behaves like production software instead of a toy example.

My proposal was to design and build four advanced red-track CTF challenges in categories the platform had zero coverage for: insecure communication between agents, cascading autonomous failures, human-trust exploitation, and rogue-agent persistence. I built all four. Then I live-tested them against the platform's actual production LLM, and the results didn't match the design docs. Every one of them hinged on the model making a single risky judgment call under the right kind of pressure, and that call only landed reliably somewhere around 30 to 40 percent of the time. A challenge that works less than half the time isn't a challenge, it's a slot machine with a writeup. I talked it through with my mentor and none of that work went out as scored content.

So I redirected. Instead of continuing to chase reliability on judgment-dependent challenges, I spent the second half of the summer auditing FinBot's own code directly, plus independently verifying candidates pulled from the project's own backlog of roughly 300 open GitHub issues, hunting for real, structural weaknesses instead of probabilistic ones. A crash is a crash no matter which model is running that day. That work turned into:

- **12 independently researched, tested, and reviewed security and reliability patches**, covering payment-pipeline correctness, SSRF, broken access control, denial-of-service, and data integrity
- **1 fully built and live-verified CTF challenge**, Cross-Vendor Email, a broken object-level authorization exploit against the chat assistant, detected mechanically instead of depending on any specific model output surviving a paraphrase

Every patch below went through the same loop before it touched a line of production code: read the actual current source and verify every claim in the linked issue against it (several of them cited stale line numbers, the wrong exception type, or a suggested fix that would have crashed the moment it ran), write the tests first, get a dedicated security review, and locally merge-test the branch against every other open PR from the same batch before pushing. On a project with dozens of people committing to the same backlog at once, a clean diff on your own branch doesn't mean much if it can't actually merge.

All 13 pull requests below are open against the upstream repo as of this report.

---

## The work, PR by PR

### 1. [Cross-Vendor Email — a BOLA/IDOR challenge, built and shipped](https://github.com/GenAI-Security-Project/finbot-ctf/pull/558)

The vendor chat assistant let you ask about a FinMail message by its raw ID, or about another vendor's inbox by ID directly, and nothing on the backend checked whether that message or inbox actually belonged to the vendor asking. That's a textbook broken object-level authorization gap, and it was sitting in the code unintentionally, not manufactured for the sake of having something to build. I turned it into a full challenge: a YAML definition, a custom detector, 28 tests, and a live end-to-end run against the deployed model before the PR ever went up. The detector itself doesn't guess at model behavior. It cross-references the tool-call event's vendor identity against the message's real owner directly at the database layer, so it fires the same way regardless of how the exploit gets phrased.

<video src="https://github.com/Deez-Automations/gsoc-2026-finbot-ctf/raw/main/media/cross-vendor-email-demo.mp4" controls width="720">
Video not supported inline, download it directly: https://github.com/Deez-Automations/gsoc-2026-finbot-ctf/raw/main/media/cross-vendor-email-demo.mp4
</video>

*Live demo: the exploit running end to end against the deployed model, detector firing on the real event stream.*

### 2. [FinDrive filename log injection](https://github.com/GenAI-Security-Project/finbot-ctf/pull/561)

File uploads accepted filenames with raw newline and control characters baked in, no sanitization at all. A filename like `invoice.pdf\nFAKE LOG ENTRY: admin login from 1.2.3.4` would land verbatim in the application log, which means an attacker could forge log lines that look completely legitimate. Fixed by rejecting `\n`, `\r`, `\t`, and null bytes at upload time, with a test for each character plus a regression check that normal filenames still work fine.

### 3. [Invoice.to_dict() crashing the payment pipeline on a missing date](https://github.com/GenAI-Security-Project/finbot-ctf/pull/562)

`Invoice.to_dict()` called `.isoformat()` straight on the invoice and due dates with no null check, so any invoice missing either field blew up with a bare `AttributeError` right inside the payment path, with zero information about what actually went wrong. Both linked issues asked for a specific behavior in their own acceptance criteria: raise a clear `ValueError`, not an opaque crash. That's what I built, with tests that reproduce the original crash first and then confirm the fix.

### 4. [FinStripe never persisted the vendor's bank account](https://github.com/GenAI-Security-Project/finbot-ctf/pull/563)

A payment transfer's destination account was accepted by the API and then just never written to the database. That means there was no way to look back at a historical payment and confirm where it actually went, which is a real gap in a system that's supposed to be auditable. Fixed with the missing column plus a proper Alembic migration, and I checked the migration applied cleanly against a throwaway copy of the database before merging, not just that it looked right on paper.

### 5. [FinStripe accepted a negative mock balance](https://github.com/GenAI-Security-Project/finbot-ctf/pull/564)

The simulated account balance could be set to a negative number with no validation whatsoever, and agents were making real payment-affordability decisions based on that value. A corrupted balance isn't just bad data here, it's a decision-integrity problem, since an agent reasoning over a negative balance can end up approving or blocking a payment for the wrong reason entirely. Added a type and range guard, tested against `None`, negative numbers, and non-numeric junk.

### 6. [FinStripe accepted a negative pagination limit](https://github.com/GenAI-Security-Project/finbot-ctf/pull/565)

Listing transfers accepted a negative `limit` and just handed it to the database, producing undefined query behavior. I didn't just patch the one MCP tool that surfaced the bug. I traced it down to the shared repository method underneath, `PaymentTransactionRepository.list_for_vendor`, and found the vendor portal's own transaction API called the exact same method with the exact same hole. Fixing only the entry point I was handed would've left the second one wide open right next to it, so the guard went in at the shared layer instead, where every caller inherits it automatically.

### 7. [Orchestrator told the vendor a payment succeeded even when it failed](https://github.com/GenAI-Security-Project/finbot-ctf/pull/574)

The orchestrator agent unconditionally instructed its own LLM to confirm a payment to the vendor, regardless of whether the payment had actually gone through. That means a failed payment could get reported back as a successful one, which is about as direct a trust-breaking bug as you can have in a financial workflow. Gated the confirmation instruction on the real, reported status instead. Worth noting: the linked issue's own suggested fix checked for a status value that doesn't exist anywhere in this codebase's schema, so applying it verbatim would have silently broken the legitimate success path too. I traced the actual enum first and used the value that's really there.

### 8. [Unauthenticated SSRF through the guardrail webhook](https://github.com/GenAI-Security-Project/finbot-ctf/pull/575)

A Labs feature lets a session register a webhook URL that the server later sends a real HTTP request to. The existing validation only checked bare IP addresses typed directly into the URL and never resolved hostnames through DNS at all, so a hostname that *resolves* to an internal address like `127.0.0.1` sailed straight through. On top of that, every endpoint that registered or triggered the webhook accepted a completely anonymous session. Put those two together and any visitor to the site, logged in or not, could point the server at an internal service and fire it immediately. Fixed both halves: real DNS resolution in the validator, and authentication required on every endpoint that changes state. The DNS fix isn't naive either. My first pass ran the resolution synchronously inside the hot path, and a security review caught that a malicious DNS target could've frozen the event loop for every user on the server at once. Rewrote it to offload onto a worker thread with a hard timeout before it ever shipped.

### 9. [Guardrail payloads could get corrupted before their own signature was computed](https://github.com/GenAI-Security-Project/finbot-ctf/pull/576)

Oversized hook payloads got truncated with a raw byte-slice right before being signed with HMAC. Cut in the wrong spot, that slice can chop a multi-byte character in half and hand back invalid JSON that still carries a technically valid signature, which is worse than useless to whoever's supposed to trust it. Separately, a safety hook meant to fire after every tool call was silently never firing for one specific, frequently-used tool. Rewrote the truncation to work field by field so it always produces valid JSON, and wired up the missing hook. Also restructured the service so it can never raise an exception that gets silently misattributed to some unrelated caller's error handling.

### 10. [Dark Lab's tool override silently ignored half of what you sent it](https://github.com/GenAI-Security-Project/finbot-ctf/pull/577)

There's an admin API that's supposed to let operators override both a tool's description and its parameter schema. In practice, the code applying that override only ever read the description. The parameter schema got accepted, stored, and reported back as successfully applied, while doing nothing at all. I fixed it to actually apply both, and along the way found that the linked issue's own one-line suggested fix would have crashed the feature outright, because it referenced a field name that doesn't exist on the object it's setting it on. I checked the real field name against the installed library directly rather than trusting the suggestion, then applied it correctly and closed two more related crash spots a second review round turned up.

### 11. [Profile share cards served the wrong image, or a stale one](https://github.com/GenAI-Security-Project/finbot-ctf/pull/578)

The public share-card endpoint cached a rendered profile image keyed on just four fields. Any two brand-new users, which is nearly everyone right after signup, had identical stats and collided on the exact same cache entry, so one of them got served the other's card. Existing users had the opposite problem: changing your avatar or bio didn't touch the cache key at all, so the old card kept getting served forever. I rebuilt the key to cover everything that actually affects the render, and traced two gaps the original bug report hadn't caught: one avatar type pulls its image from the user's email rather than the field the issue suggested keying on, and a separate endpoint lets you change which badges are featured with zero effect on any of the fields the simple fix would've covered.

### 12. [Chat assistant crashed the moment an ID argument went missing](https://github.com/GenAI-Security-Project/finbot-ctf/pull/579)

Both chat assistants, the vendor-facing one and the admin-facing one, each had five tool-calling methods that took a vendor or invoice ID straight from the model and forwarded it to the database with no check for whether it was even present. The linked issue described this as a 5-site bug. Tracing the actual code turned up 12, because the same methods are duplicated across both assistant classes and two more had the identical gap that never made it into the original report. Added explicit guards to all twelve, returning a clean, correctly-worded error instead of letting a missing argument masquerade as "record not found," which risks the model concluding a real vendor doesn't exist when it just wasn't given what it needed.

### 13. [Orchestrator context got polluted by whitespace pretending to be a summary](https://github.com/GenAI-Security-Project/finbot-ctf/pull/580)

The multi-agent orchestrator captures each agent's summary and hands it to the next agent as context, gated by a plain truthiness check. A string made of nothing but spaces is truthy in Python, so it sailed past the check and got injected as pure noise into every step downstream. Tightened the guard to reject whitespace-only and non-string values both. A guard that a technically-non-empty, functionally-empty string can slip through is exactly the kind of gap a prompt-injection attempt could exploit to smuggle something past it unnoticed, so this wasn't just cleanup. Nine new tests came with it, including one added after a review pass pointed out that the obvious fix would crash on a non-string input instead of just failing safely.

---

## What this adds up to

None of these thirteen patches are flashy on their own. A missing null check, a byte-slice done wrong, a status check that used the wrong enum value. But every one of them got the same treatment: read the real code first, write a failing test that proves the issue, fix it, get it reviewed, and check it actually merges cleanly next to everything else in flight. Several corrected real mistakes in the reports that described them. Several turned out to be bigger than reported once I actually traced the code instead of taking the write-up at face value.

That's the work. Small individually, and a real, cumulative improvement to a platform other people are actively learning security on.

---

*Current as of August 2026. For live status on any PR, follow the links above directly.*
