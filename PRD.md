## Problem Statement

Companies need a fast, consistent way to triage customer-reported issues into a tracker, and management needs visibility into what's happening across all open issues — without digging through a tracker UI or waiting on a human triager. Manual triage is slow and inconsistent; aggregate visibility usually means someone manually compiling a status report.

This is also a portfolio piece: it needs to demonstrate an agent that makes autonomous tool-call decisions (not just answers questions), and that its decision quality can be measured, not just anecdotally observed.

## Solution

An AI ops agent, backed by Linear (free tier) as the real issue tracker, usable in two ways:

1. **Customer-facing chat**: a customer describes an issue conversationally; the agent decides whether to create a new ticket, update an existing one, or ask a clarifying question, then acts via Linear tool calls.
2. **Management-facing chat**: a manager asks natural-language questions about issue trends/status ("how many P1s open", "what's trending this week") and the agent answers using aggregate Linear-backed tools.
3. **Autonomous ingest**: a synthetic `/ingest` endpoint accepts alert/log/complaint-style payloads and the agent decides create/update/ignore/escalate with no human prompting per action — this is the "ops AI agent" story, not just a chatbot.
4. **Eval harness**: every agent decision is logged as a structured trace (input → tool calls made → resulting state). A scenario library (input → expected end-state) is scored against actual outcomes. This trace format is also the data contract a future fine-tuning project will consume.

Reuses uul_chat_ai's proven skeleton (FastAPI + hand-rolled tool-calling loop + shared LiteLLM gateway + Supabase auth) rather than adopting a new agent framework — the point of this portfolio piece is demonstrating the decision loop and eval methodology, not framework integration.

## User Stories

1. As a customer, I want to describe my issue in plain language, so that I don't need to know how to fill out a ticket form.
2. As a customer, I want the agent to recognize when my message is about an issue I already reported, so that it updates the existing ticket instead of creating a duplicate.
3. As a customer, I want to ask for the status of my previously reported issue, so that I know where it stands without contacting support.
4. As a customer, I want the agent to ask a clarifying question when my report is ambiguous, so that it doesn't file a wrong or low-quality ticket.
5. As a customer, I want to only see/affect my own tickets, so that I can't view or modify other customers' issues.
6. As a manager, I want to ask "how many P1 issues are open" and get a direct answer, so that I don't have to open the tracker myself.
7. As a manager, I want to ask about trends ("is volume up or down this week"), so that I can spot emerging problems early.
8. As a manager, I want to ask which issues are escalated or overdue, so that I can intervene before they become bigger problems.
9. As a manager, I want a view across all customers' issues (not just my own), so that I have full company visibility.
10. As a portfolio reviewer (recruiter/interviewer), I want to log in and try both the customer and management experience, so that I can evaluate the range of the system.
11. As a portfolio reviewer, I want to see the agent handle an autonomous alert payload live, so that I can evaluate its decision-making without needing to role-play a customer conversation.
12. As a portfolio reviewer, I want to see the eval harness's scoring output, so that I can evaluate whether the agent's decisions are actually measured, not just claimed.
13. As the system, when an ingested alert/log/complaint payload arrives, I want to decide between create/update/ignore/escalate, so that noise doesn't flood the tracker and real issues aren't missed.
14. As the system, I want every tool-call decision persisted as a structured trace (input, reasoning context, tool calls, resulting state), so that decision quality can be scored and so the trace log is reusable as training data for a future fine-tuned model.
15. As the developer, I want the data model to carry a `company_id` on tickets/tools/users even though only one demo company is seeded, so that the design reads as SaaS-shaped without demo complexity.
16. As the developer, I want to define scenario fixtures (input payload + expected end-state) and a scorer that diffs actual vs expected state, so that I have an objective, repeatable measure of agent decision quality (tau-bench-style).
17. As the developer, I want to reuse the existing STB LiteLLM gateway and Supabase auth setup, so that I'm not duplicating infra across portfolio projects.
18. As the developer, I want role-based tool exposure (customer vs management tool subsets), so that the same agent core safely serves two different trust levels.

## Implementation Decisions

- **Agent Core**: reuse uul_chat_ai's `run_turn`/`run_turn_stream` tool-calling loop shape. Two persona system prompts (customer vs management), selected by the authenticated user's `role`.
- **Linear adapter**: wraps Linear's official MCP server (GraphQL under the hood) exposing `create_ticket`, `update_ticket`, `get_ticket_status`, `list_tickets`, `escalate_ticket`.
- **Ingest endpoint**: `POST /ingest` accepts a synthetic alert/log/complaint-style JSON payload (no real external webhook integration in v1), routes into the agent's autonomous decision path (no chat turn required), returns the decision + resulting Linear action.
- **Analytics tools**: `trend_summary`, `escalation_view`, `list_by_priority` — Linear-API-backed, exposed only to the `management` role.
- **Role gate**: reuse Supabase auth (same JWKS pattern as uul_chat_ai). `role` enum (`customer` | `management`) stored on the user record, determines which tool schemas + system prompt are used. Two seeded demo accounts (one per role) for live portfolio demo.
- **Data model**: `company_id` FK present on tickets/users from day one (multi-tenant-ready schema), but only one demo company is seeded and shown live.
- **Eval harness**: `eval/` directory — scenario fixtures (input payload + expected end-state: ticket created y/n, priority, action taken), a scorer comparing actual vs expected state (tau-bench-style diff), and a structured decision-trace logger (JSON per agent turn: input, tool calls made, final state). Trace schema is deliberately designed to double as future fine-tune training data.
- **Gateway**: same STB-hosted LiteLLM instance already serving uul_chat_ai — no new LLM infra, `LITELLM_BASE_URL` points at the existing gateway.
- **Deploy**: same Tencent VPS as uul_chat_ai, new subdomain, own fe/be/caddy (or shared caddy instance, TBD at implementation time) — confirmed sufficient headroom after disk cleanup (8.7G avail, 77% used, dangling docker images pruned).
- **FE**: reuse uul_chat_ai's chat-window pattern (single chat UI), role-aware suggested questions, same Supabase auth flow.

## Testing Decisions

- Good tests here assert externally observable behavior — the tool call(s) the agent chooses to make and their resulting state — not internal implementation details of the loop.
- **Agent Core**: tool-selection correctness given scripted inputs, mirroring uul_chat_ai's `test_agent_core.py` pattern (fake gateway client, assert which tool got called and with what args).
- **Linear adapter**: unit tests against a mocked Linear API/MCP response, no live network calls in CI.
- **Eval scorer**: pure-function unit tests — given actual vs expected state, assert correct pass/fail scoring.
- **Ingest endpoint**: integration tests — POST a synthetic payload, assert the resulting decision and Linear tool call.
- **FE**: light/no automated tests, matching uul_chat_ai's actual investment (this portfolio's test signal lives in the backend decision logic).

## Out of Scope

- Real external webhook integrations (GitHub/UptimeRobot/etc.) — synthetic `/ingest` payloads only in v1.
- Dashboard UI with charts/graphs — management visibility is chat-only in v1.
- Full RBAC system — a single `role` enum field is sufficient, no granular permissions.
- Live multi-company demo — schema is multi-tenant-ready but only one company is seeded/shown.
- Fine-tuning an open-source model on this agent's decision traces — that is a separate, later portfolio repo that consumes this repo's trace logs as training data.

## Further Notes

- Build order across the broader portfolio effort: this repo first, then a Python e-commerce shopping assistant (`shop-ai-assistant` — separate PRD), then the fine-tuning repo last (consumes this repo's eval trace data).
- VPS disk was cleaned before starting this project (pruned ~5G of dangling docker images from past deploys, freeing 4.1G → 8.7G available) to ensure headroom for a second app stack.
