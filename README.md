# Meow Meow Veterinary Hospital: AI Front Desk

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
