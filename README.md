# GSoC 2026 — OWASP FinBot CTF: Final Work Report

**Contributor:** Muhammad Daniyal ([@Deez-Automations](https://github.com/Deez-Automations))
**Program:** Google Summer of Code 2026
**Organization:** OWASP Foundation
**Project:** OWASP GenAI Security — FinBot CTF: The Autonomous Threat Track
**Project repository:** [OWASP-ASI/finbot-ctf](https://github.com/OWASP-ASI/finbot-ctf)
**Mentor:** Rishi Mondal
**Status:** Midterm evaluation passed

---

## Summary

FinBot CTF is a hands-on training platform built around a simulated agentic finance system — vendors, invoices, payments, and a set of LLM-powered agents that process them. The platform teaches AI security by letting players actually exploit realistic vulnerabilities in that system.

My original proposal was to design and ship four advanced red-track CTF challenges covering categories the platform had no coverage for at all: insecure inter-agent communication, cascading autonomous failures, human-trust exploitation, and rogue-agent persistence. I built all four. Live-testing them against the platform's production LLM told a different story than the design docs did: each challenge depended on the model making one specific, risky judgment call under pressure, and that call only landed reliably 30–40% of the time. A CTF challenge that only sometimes works isn't a CTF challenge — it's a coin flip with extra steps. After discussing this with my mentor, none of that work shipped as scored content.

Instead of forcing unreliable challenges out the door, I redirected the second half of the summer toward something with a lower failure surface: auditing FinBot's own codebase directly, and independently verifying candidates from the project's own ~300-issue QA backlog, for **real, structural bugs** — the kind that are true regardless of which LLM the platform happens to be running that week. That work produced:

- **12 independently verified and tested bugfix pull requests**, spanning payment-pipeline correctness, SSRF, broken access control, denial-of-service, and data-integrity issues
- **1 fully built, live-verified CTF challenge** — Cross-Vendor Email, a broken object-level authorization (BOLA) exploit against the platform's chat assistant, built with a purely mechanical detector that doesn't depend on any specific LLM behavior surviving a paraphrase step

Every fix below followed the same process: read and independently verify the actual current source before writing a line of code (several linked GitHub issues turned out to cite stale line numbers, wrong exception types, or propose a "fix" that would itself crash at runtime), write the tests first, get a dedicated security review pass, and locally merge-test against every other open PR from the same batch before pushing — because in an open-source project with dozens of concurrent contributors, a clean diff on your own branch means nothing if it breaks on merge.

All 13 pull requests listed below are open against the upstream repository as of this report.

---

## The Work, PR by PR

Each entry follows the same four-part shape: **what the bug was, what I did, why it mattered, how the fix works.**

### 1. [Cross-Vendor Email — BOLA/IDOR CTF Challenge](https://github.com/GenAI-Security-Project/finbot-ctf/pull/558)
**What it was:** The platform's vendor chat assistant could be asked about a FinMail message by its raw ID, or about another vendor's inbox by ID directly — and nothing checked whether that message or inbox actually belonged to the vendor asking.
**What I did:** Built a complete CTF challenge around this real gap: a YAML challenge definition, a custom event-driven detector, and 28 tests, then live-tested the full exploit path against the deployed LLM before opening the PR.
**Why it mattered:** Broken object-level authorization is one of the most common vulnerability classes in real production APIs, and the platform had almost no hands-on content teaching it.
**How it's fixed / built:** The detector is purely mechanical — it cross-references the tool-call event's vendor identity against the message's real owner at the database layer, with zero dependence on any specific LLM wording surviving a rephrase.

### 2. [FinDrive filename log injection](https://github.com/GenAI-Security-Project/finbot-ctf/pull/561)
**What it was:** File uploads accepted filenames containing raw newline and control characters with no sanitization at all.
**What I did:** Rejected control characters (`\n`, `\r`, `\t`, `\x00`) in filenames before they ever reach storage or the application log.
**Why it mattered:** A crafted filename could inject a fake, convincing-looking line into the application's own audit log — a real log-injection vector, not just cosmetic input hygiene.
**How it's fixed:** A validation check at the upload entry point, with a dedicated test for each control character plus a regression test confirming ordinary filenames are unaffected.

### 3. [Invoice null-date crash in the payment pipeline](https://github.com/GenAI-Security-Project/finbot-ctf/pull/562)
**What it was:** `Invoice.to_dict()` called `.isoformat()` directly on two date fields with no null check, so any invoice missing a date crashed with an opaque, undiagnosable `AttributeError` — inside the payment-processing path itself.
**What I did:** Replaced the crash with a clear, named `ValueError`, matching the exact behavior the linked issue's own acceptance criteria required.
**Why it mattered:** An unhandled crash mid-payment is a worse failure mode than a loud, diagnosable one — a corrupted invoice should fail obviously, not opaquely.
**How it's fixed:** Explicit guards before serialization, with tests that reproduce both original crashes first, then confirm the fix.

### 4. [FinStripe vendor bank account never persisted](https://github.com/GenAI-Security-Project/finbot-ctf/pull/563)
**What it was:** A payment transfer's destination bank account was accepted by the API but silently never written to the database.
**What I did:** Persisted it correctly, including a real Alembic schema migration.
**Why it mattered:** Without it, there was no way to audit which account a historical payment actually went to — a genuine gap in a financial system's audit trail.
**How it's fixed:** Added the missing column and migration, independently verified applying cleanly against a throwaway copy of the database before merging.

### 5. [FinStripe accepts a negative mock balance](https://github.com/GenAI-Security-Project/finbot-ctf/pull/564)
**What it was:** The platform's simulated payment balance could be set to a negative number with zero validation, and autonomous agents made real payment decisions based on that value.
**What I did:** Added a type and range guard rejecting negative or non-numeric balances.
**Why it mattered:** A corrupted balance value could cause an agent to reason incorrectly about whether a payment was affordable — a decision-integrity bug, not just a data one.
**How it's fixed:** A guard at the config-read boundary, with tests covering `None`, negative, and non-numeric inputs.

### 6. [FinStripe accepts a negative pagination limit](https://github.com/GenAI-Security-Project/finbot-ctf/pull/565)
**What it was:** Listing payment transfers accepted a negative `limit` value, producing undefined database query behavior.
**What I did:** Rejected negative values, and moved the guard down into the shared repository method itself rather than just the one MCP tool wrapper that called it.
**Why it mattered:** A second, real caller — the vendor portal's own transaction API — shared that exact repository method and had the identical unguarded gap; fixing only the surface entry point would have left the door open right next to it.
**How it's fixed:** A `ValueError`-raising guard inside `PaymentTransactionRepository.list_for_vendor`, so every current and future caller inherits the protection automatically.

### 7. [Orchestrator confirms a payment to the vendor even when it failed](https://github.com/GenAI-Security-Project/finbot-ctf/pull/574)
**What it was:** The orchestrator agent unconditionally told its own LLM to confirm a payment to the vendor, regardless of whether that payment had actually succeeded.
**What I did:** Gated the confirmation instruction on the payment's real, reported status.
**Why it mattered:** A failed payment was being reported to the vendor as successful — a direct correctness bug with real financial-trust consequences.
**How it's fixed:** A `task_status == "success"` check before the confirmation instruction is generated at all. Along the way, caught that the linked issue's own suggested fix checked for a status value that doesn't exist anywhere in the codebase's schema, and used the correct one instead.

### 8. [Unauthenticated SSRF via the guardrail webhook](https://github.com/GenAI-Security-Project/finbot-ctf/pull/575)
**What it was:** A platform feature let a session register a URL the server would later send an HTTP request to. The existing safety check only validated bare IP addresses typed directly into the URL — it never resolved hostnames via DNS — and separately, every endpoint that registered or triggered a webhook accepted a fully anonymous, unauthenticated session.
**What I did:** Closed both halves: added real DNS resolution to the URL validator (so a hostname that *resolves* to an internal address is caught, not just a literal IP), and required an authenticated session on every state-changing endpoint.
**Why it mattered:** Combined, an anonymous visitor to the platform could register an internal-network URL and immediately trigger the server into making a request to it — textbook unauthenticated Server-Side Request Forgery, one of the OWASP Top 10.
**How it's fixed:** DNS resolution now runs on an async, timeout-bounded worker thread (after a security review caught that my own first pass would have let a malicious DNS target freeze the server's event loop for every concurrent user — a self-introduced denial-of-service, found and closed before it ever shipped), and write endpoints now require a logged-in session.

### 9. [Guardrail payload corruption before signing, and a missing safety hook](https://github.com/GenAI-Security-Project/finbot-ctf/pull/576)
**What it was:** Oversized hook payloads were truncated with a raw byte-slice before being cryptographically signed — which could cut a multi-byte character in half and produce invalid JSON that still carried a technically-valid signature. Separately, a safety hook that should fire after every tool call never fired for one specific, commonly-used tool.
**What I did:** Rewrote the truncation to always produce valid JSON at the field level instead of a blind byte cut, and wired up the missing hook.
**Why it mattered:** A receiver getting an unparseable-but-signed payload is a worse failure mode than an oversized one; a hook silently never firing meant an entire class of tool calls went unmonitored by the platform's own safety system.
**How it's fixed:** Field-aware truncation logic, plus a structural change ensuring the guardrail service can never raise an exception that gets silently misattributed to an unrelated caller.

### 10. [Dark Lab tool override silently dropped half its input](https://github.com/GenAI-Security-Project/finbot-ctf/pull/577)
**What it was:** An admin-facing API lets operators override an AI tool's description *and* its parameter schema — but the code that applies the override only ever read the description half. The parameter-schema half was accepted, stored, and reported back as successfully applied, while doing nothing at all.
**What I did:** Fixed the code to apply both fields, and along the way caught that the linked issue's own suggested one-line fix would have crashed the entire feature at runtime, because it referenced a field name that doesn't exist on the underlying object.
**Why it mattered:** This override mechanism is the platform's own sandbox for teaching tool-poisoning attacks — a silently broken half of it undermined the exact feature it exists to demonstrate.
**How it's fixed:** Verified the correct field name empirically against the actual installed library before writing anything, applied both fields independently, and closed two further crash-causing edge cases (a non-string description, a malformed override entry) found during a second review pass.

### 11. [Profile share-card cache served stale or wrong images](https://github.com/GenAI-Security-Project/finbot-ctf/pull/578)
**What it was:** A public profile card image was cached to disk using a key built from only four fields. Two users with matching stats — common for any brand-new account — collided on the exact same cached file, and any change to an existing user's avatar or bio was invisible to the cache, serving the old image indefinitely.
**What I did:** Rebuilt the cache key to include every field that actually affects what gets rendered, going two steps beyond what the linked issue itself had proposed.
**Why it mattered:** Tracing the actual avatar-rendering code showed that one avatar type resolves its image from the user's *email*, not the field the issue suggested keying on — and a separate, unrelated endpoint lets users change which badges are featured on their card with zero effect on the fields the original proposal covered.
**How it's fixed:** A small, independently-tested pure function computing the key from every relevant field, with the hash upgraded from MD5 to SHA-256.

### 12. [Chat assistant crashes on a missing vendor or invoice ID](https://github.com/GenAI-Security-Project/finbot-ctf/pull/579)
**What it was:** The platform's two chat assistants (vendor-facing and admin-facing) each had five AI-callable tools that forwarded an ID argument straight to the database with no check for a missing value.
**What I did:** Added explicit guards to all of them — and found the real scope was 12 call sites, not the 5 the linked issue described, since the same methods are duplicated across both assistant classes, plus two more with the identical gap that the original report never mentioned.
**Why it mattered:** A missing ID produced a misleading "not found" error, which risked the AI reasoning incorrectly that a real vendor or invoice simply didn't exist, rather than recognizing it just hadn't been given the argument it needed.
**How it's fixed:** Explicit guards returning a clear, correctly-worded error before the call ever reaches the data layer — and corrected the linked issue's claimed crash type after tracing what the code actually does at runtime.

### 13. [Orchestrator context polluted by whitespace-only agent summaries](https://github.com/GenAI-Security-Project/finbot-ctf/pull/580)
**What it was:** The multi-agent orchestrator captures each agent's summary to pass along as context to the next agent, using a simple truthiness check — but a string made of only whitespace is truthy in Python, so it passed the check and was injected as pure noise into every following step.
**What I did:** Tightened the guard to also reject whitespace-only and non-string values.
**Why it mattered:** Beyond the wasted context, a guard that a technically-non-empty-but-meaningless string can satisfy is exactly the kind of soft spot a prompt-injection attempt could exploit to slip content past it unnoticed.
**How it's fixed:** A one-line guard change, backed by 9 new tests — including one added after a review pass flagged that the obvious fix would itself crash on a non-string input instead of failing safely.

---

## What this adds up to

Thirteen pull requests, all independently researched, tested, and reviewed before being opened — not thirteen copy-pasted one-liners. Several of them corrected factual errors in the GitHub issues that described them (a stale line number, a wrong exception type, a suggested fix that would have crashed on its own). Several went further than what was originally reported, because tracing the actual code turned up more of the same bug class than the issue itself had found. Every one of them includes tests written before the fix, confirming the bug first and the fix second.

None of these are individually dramatic. Together, they represent a real, measurable improvement to the correctness, security, and reliability of a platform that other people are actively learning AI security on.

---

*This report was current as of August 2026. For up-to-date PR status, see the links above directly.*
