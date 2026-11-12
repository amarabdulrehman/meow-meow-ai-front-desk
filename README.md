# Meow Meow Veterinary Hospital: AI Front Desk + PayPal Braintree No-Show Protection

An AI receptionist for a vet clinic, built in **n8n** for **AI Agents Arena 2.0 (CUI Tech Fest 2026)**.
A pet owner sends one message, and the system books, cancels or reschedules on **Google Calendar**, records it in **Google Sheets**, emails a confirmation, and hands anything medical to staff.

> **Core idea:** the AI *understands* the customer; plain code *decides and acts*. Smart, but predictable.

## How it works
1. **Entry:** n8n Form, JSON API (`POST /webhook/mmvh-api`) or the customer website.
2. **Context:** reads clinic rules from a Config sheet (hours, lunch, services, providers, holidays, ARENA_NOW) and the CallLog. A repeated `request_id` replays the saved answer, so nothing is booked twice.
3. **AI agent (Gemini):** works out intent, asks for missing details, and calls one of 5 tools: `check_availability`, `book_appointment`, `cancel_appointment`, `reschedule_appointment`, `request_staff_handoff`.
4. **Engine (deterministic sub-workflow):** validates every rule, reads the calendar, checks overlaps and suggests up to 3 alternatives. Then it creates, moves or deletes the event and writes the sheet, **rolling back the calendar if the sheet write fails**.
5. **Reply:** the final status comes from tool evidence, never from the model's claim. Everything is logged in CallLog.

## Features
- Book / cancel / reschedule (same event moved) / availability / staff handoff
- Rules: 30-min grid, lead time, booking horizon, clinic hours, lunch, closed days & holidays, provider days, service–species match, all-day events count as busy
- Guardrails: no medical advice (symptoms → handoff), prompt-injection safe, never invents times or references, asks when a time zone is unclear
- Reliability: evidence-based status, calendar/sheet rollback, idempotent replay, Gemini → backup Gemini → Groq fallback, timeouts, error-handler workflow
- **Fault injection:** `debug.fault` in Config simulates any failure live
- Extras: Gmail confirmations, replies in the customer's language, website chat memory, accessible website (larger text, high contrast, voice input)

## NEW: No-show protection with PayPal Braintree (PayPal AI Hackathon 2026 update)
Missed vet appointments waste a vet's time and lose the clinic money, and small clinics can't afford a front-desk person to chase deposits.
The AI front desk now **books the slot and protects it with a refundable deposit**, end to end:

1. **AI books** - the Gemini agent understands the request and calls `book_appointment` (unchanged guardrails).
2. **Code requests the deposit** - the new `MMVH Payments (PayPal Braintree)` sub-workflow writes a `PENDING` row to the **Payments** ledger and returns a secure, unguessable pay link. The model never creates links or amounts.
3. **Customer pays** - `pay.html` (static, GitHub Pages) loads the booking from `/webhook/mmvh-pay` and shows a Braintree **Drop-in** checkout (client token from the Braintree GraphQL API). The amount always comes from the server-side ledger, never from the browser.
4. **Charge + confirm** - `/webhook/mmvh-pay-submit` runs `chargePaymentMethod`, marks the deposit `PAID` and emails a receipt.
5. **Smart refunds** - on cancel, a deterministic policy decides: 24h+ before the visit = automatic `reverseTransaction` (void/refund); later = deposit kept (`KEPT_LATE_CANCEL`); never paid = `VOIDED`. The AI only explains the outcome in the customer's language.
6. **Agentic follow-up** - a new agent tool `check_deposit_status` lets customers ask "did my deposit go through?" or "send my link again" (reference + email verified).

**Resilience:** if payments fail, the booking still stands and the customer is told staff will follow up (`Payments Unavailable` path); every money step is logged in the ledger and CallLog.

| Path | What it is |
|---|---|
| `workflows/mmvh-payments.json` | Payments sub-workflow: deposit request, pay-page API, charge, reverse/refund policy, status tool |
| `pay.html` | Customer deposit page (Braintree Drop-in), served by GitHub Pages |

**Extra setup:** create an n8n *Basic Auth* credential "Braintree Sandbox" (User = public key, Password = private key) and attach it to the 3 Braintree nodes. The `Payments` tab is created automatically. Point the `PayPal Deposit & Refunds` and `check_deposit_status` nodes at the imported Payments workflow, and enable GitHub Pages for `pay.html`. Sandbox test card: `4111 1111 1111 1111`.

## Statuses
`BOOKED` · `CANCELED` · `RESCHEDULED` · `ALTERNATIVES` · `NEEDS_INFO` · `HANDOFF` · `DECLINED` · `ERROR` · `VERIFICATION_FAILED`

## Repository
| Path | What it is |
|---|---|
| `workflows/mmvh-ai-front-desk.json` | Main workflow: triggers, agent, tools, logging, email, reply |
| `workflows/mmvh-engine.json` | Deterministic booking engine (rules, calendar, sheet, rollback) |
| `workflows/mmvh-error-handler.json` | Error workflow |
| `workflows/mmvh-website.json` | Serves the customer website from n8n |
| `website/index.html` | The customer website ("Sheru Bhai" theme) |

## Run it yourself
1. Import the 4 JSON files into n8n (Workflows → Import from file).
2. Create credentials: Google Calendar, Google Sheets, Gmail (OAuth2), Google Gemini, and optionally Groq. Attach them where nodes show `REPLACE_ME`.
3. Create a Google Sheet with tabs **Config**, **Appointments** and **CallLog**, and replace `YOUR_SPREADSHEET_ID` in the nodes.
4. In the Front Desk workflow, point the tool nodes at your imported Engine workflow, then publish all workflows.

## Tested
100+ checks: rule boundaries, calendar conflicts, booking/replay, reschedule/cancel, config changes, injected failures, LLM behaviour and model failover.

## Known limitations
- Two customers booking the exact same slot at the same instant is not fully atomic.
- Free-tier LLM quotas (mitigated by fallback tiers).

Built by **Amar**.
