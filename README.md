# AeroDesk — Customer-Facing Airline Disruption Resolution Agent

A prototype customer resolution agent for airline disruptions (cancellations and delays), built for the AIONOS Agentic AI Factory recruitment assignment (Assignment 3).

## Run it

No build step, no dependencies, no server required.

Open `index.html` directly in any modern browser (double-click it, or `start index.html` on Windows). If your browser blocks local scripts, serve the folder with any static file server, e.g. `npx serve .`, and open the printed URL.

Pick a customer on the landing screen (Priya / Arvind / Meher) to open their case.

## What this is

A support agent for a specific customer journey — an airline disruption — that:
- understands what the customer is asking for, including multiple asks in one message
- already knows the customer's own booking and flight status (never re-asks for it)
- applies the airline's stated policy deterministically, not by LLM guesswork
- takes the allowed action, or escalates when it lacks the authority to do more
- replies in plain, empathetic customer-support language
- keeps a separate, evaluator-facing decision trace and audit trail of every reasoning step

## Architecture

```
Customer message
      │
      ▼
Intent Layer (pattern-based NLU)  ──►  one or more intents
      │
      ▼
Policy Engine (deterministic, pure functions)
  • delay compensation thresholds
  • cancellation entitlement
  • fare-difference authority limit
  • refund-method rule
      │
      ├─► ALLOWED  → Action executed (simulated) + Resolution card updated
      └─► NOT ALLOWED → Escalation created + Resolution card updated
      │
      ▼
Audit Trail + Decision Trace (evaluator view)
      │
      ▼
Natural-language response (customer view)
```

The core design principle: **the intent/NLU layer proposes an interpretation of what the customer wants; the policy engine is the only thing authorized to decide what happens.** The LLM/intent layer can misclassify free text, get replaced, or be upgraded — it can never itself grant compensation, waive a fare difference, or decide an escalation. That authority lives entirely in `Policy` (see `index.html`), which is plain deterministic JavaScript with no model in the loop.

This split is what makes hallucinated compensation structurally impossible: even if the intent layer mis-tags a message, the policy engine only ever executes actions that are explicitly defined against the supplied rules.

### Why a single static HTML/CSS/JS file
Given the assignment's time limit, reliability mattered more than infrastructure. A self-contained file with no backend, database, or build step: runs anywhere instantly, has zero deployment risk for a live demo, and keeps the entire decision path (data → policy → action → log) inspectable in one place for the defense.

### Intent layer — an intentional, documented placeholder for an LLM
`IntentEngine.classify(text)` in `index.html` uses keyword/pattern matching tuned to the customer journeys in the Data Pack, returning a list of intent tags (a message can carry more than one, e.g. a refund request and an upgrade request together). It is written as a swappable module: it takes a string and returns intent tags, and nothing downstream cares how those tags were produced. In a production build this function would be replaced by an LLM function-calling step (e.g. Claude) that maps free text to the same fixed intent vocabulary — the policy engine, resolver, and UI would not change at all. This was a deliberate scoping decision for the time available, not a hidden shortcut.

## Source of truth

All customer, booking, and policy data lives in the `DATA` object at the top of `index.html`, transcribed directly from the AIONOS Data Pack:
- 3 customers (Priya Nair / Gold, Arvind Kulkarni / Silver, Meher Kaur / Platinum) with their real PNRs and flights
- Delay compensation tiers, cancellation rebooking/refund rule, fare-difference agent limit (₹1,500), loyalty tier rule
- No policy, price, or customer fact is invented. The Data Pack's sample conversations were used only for tone reference, never as data, per the assignment's explicit instruction.
- Customer contact details (phone/email) from the Data Pack are intentionally **not** surfaced in the UI — the app only shows what's operationally relevant (name, tier, PNR, booking, history summary).

## Agent vs. deterministic logic

| Layer | Responsibility | Can it override policy? |
|---|---|---|
| Intent layer | Parse free text into intent tags; detect frustration for tone | No |
| Policy engine | Evaluate delay/cancellation/fare-diff/refund rules against the known booking | This **is** the authority |
| Resolver | Combine customer + booking + intents + policy result into an action or an escalation | No — it only executes what the policy engine returns |
| Response composer | Turn the resolver's outcome into natural, empathetic language | No — wording only, never facts |

## Escalation conditions implemented
- Any request for compensation beyond the stated policy (e.g. a complimentary upgrade)
- Fare difference above the ₹1,500 agent limit (loyalty tier does not raise this limit)
- Refund requested to a different payment method than the original
- Any mention of legal action or a formal complaint — escalated immediately, overriding the rest of that turn

## Design & Architecture FAQ
**Why a single HTML file?** Given the 6-hour limit, it prioritizes reliability, inspectability, and zero-dependency execution over infrastructure — it opens instantly for an evaluator with no build, server, or deploy step, and every decision path is readable top to bottom.
**Why no runtime LLM/API call?** None is running in this prototype. `IntentEngine` is intentionally kept behind a stable `classify(text) → intent tags` interface so a real LLM/function-calling layer could replace it later without moving policy authority into the model — the policy engine would still gate every action exactly as it does today.
**Why is the policy engine deterministic?** Thresholds and entitlements (delay tiers, the ₹1,500 fare-diff limit, refund rules) are facts from the Data Pack, not judgment calls — encoding them as plain functions means the same input always produces the same, checkable decision, rather than relying on a model to get a number right.
**How would this become a production system?** It would connect to real customer/booking systems and governed action APIs (rebooking, refunds, vouchers) instead of simulated actions, add persistence and observability for the audit trail, and swap the intent layer for an LLM call behind the same policy gate.

## Evaluator view
Click **Decision Trace** in the header, or the **Decision Trace** tab in the right-hand panel, to see the internal reasoning for each turn (request → booking state → rule → decision → action) plus a running audit trail. This is kept separate from the customer-facing chat by design — the customer never sees rule names, thresholds, or internal terminology.

## Scenarios covered (from the Data Pack)
1. **Priya (Gold, SK4821X)** — flight cancelled; asks for a full refund *and* a free business-class upgrade in the same message. Refund is granted (original payment method, 7 business days); the upgrade is declined and escalated. A legal-threat message is also included as a robustness check.
2. **Arvind (Silver, TR1190B)** — 4-hour delay; asks for hotel accommodation. Declined (hotel requires >5h) — meal voucher and lounge access are confirmed instead.
3. **Meher (Platinum, WL7742)** — 6-hour delay; asks for a full night's hotel stay (capped at delayed-hours coverage) and to move to a higher-fare flight with a ₹2,000 fare difference (exceeds the ₹1,500 agent limit — escalated to a supervisor; Platinum tier does not extend this limit).

## AI tools used and how
- **Claude Code** was used as the primary build tool for this assignment: reading and structuring the Data Pack, designing the propose/validate architecture, writing the HTML/CSS/JS application, and testing all three scenarios live in a browser.
- No AI model call happens at runtime in this prototype — the "agent" in this submission is the deterministic pipeline described above, with the intent-classification step explicitly documented as the seam where a real LLM call would plug in.

## Known scope limits (deliberate)
- No backend, database, authentication, or real payment/airline integration — all actions are simulated and logged in-memory.
- Only the three customers in the Data Pack are modeled.
- The Data Pack does not specify behavior exactly at the 3h/5h delay boundaries; this build treats "more than 3/5 hours" literally (strictly greater than), matching the given wording. The two delay scenarios in this exercise (4h, 6h) are unaffected by that edge case.
