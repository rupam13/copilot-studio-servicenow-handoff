# Field Mapping — Copilot Studio to ServiceNow

## Standard `incident` fields

| ServiceNow field | Source | Notes |
|---|---|---|
| `caller_id` | `System.User.PrincipalName` resolved via `sys_user` | **Server-side only.** Never from conversation text |
| `short_description` | Agent-summarised intent, prefixed `[Copilot]` | Capped at 160 chars |
| `description` | Full transcript, last N turns | N = `TranscriptMaxTurns`, default 30 |
| `contact_type` | Literal `virtual_agent` | Required for deflection reporting |
| `impact` | Deterministic map from user answer | 1 = site, 2 = team, 3 = individual |
| `urgency` | Derived from deflection attempts | 2 when attempts >= 2, else 3 |
| `category` | Mapped from matched KB category | Falls back to `inquiry` |
| `assignment_group` | Mapped from category | Never left null, unassigned tickets age silently |

## Custom fields to create

Create these on the `incident` table before first use. Without them the payload silently drops the context and the whole point of the integration is lost.

| Field | Type | Max length | Purpose |
|---|---|---|---|
| `u_conversation_id` | String | 64 | Joins the ticket to Application Insights telemetry |
| `u_bot_attempted_kb` | String | 500 | Semicolon-delimited KB numbers already shown |
| `u_bot_confidence` | Decimal | — | Final orchestration confidence, 0.00 to 1.00 |
| `u_deflection_attempts` | Integer | — | Resolution attempts before escalation |
| `u_bot_version` | String | 32 | Agent solution version, for regression attribution |

### Why `u_bot_version` matters

When deflection rate drops, the first question is always "what changed?". Without the agent version on the ticket you are correlating a metric against a deployment log by timestamp. With it, the regression is a single grouped query.

## Impact and urgency mapping

| User phrase | `impact` | Rationale |
|---|---|---|
| "whole site", "everyone", "all of us" | 1 | Site-wide |
| "my team", "our department" | 2 | Group-level |
| anything else | 3 | Individual |

| Condition | `urgency` |
|---|---|
| `DeflectionAttempts >= 2` | 2 |
| otherwise | 3 |

Both mappings are **deterministic**. Priority drives SLA clocks and on-call paging. A model must never decide who gets woken up at 3am.

## Deflection table — `u_bot_deflection`

| Field | Type | Purpose |
|---|---|---|
| `u_conversation_id` | String | Correlation key |
| `u_caller` | Reference to `sys_user` | Who was deflected |
| `u_intent` | String | Raw user utterance |
| `u_kb_articles` | String | Articles that resolved it |
| `u_outcome` | Choice | `Deflected` / `Escalated` / `Abandoned` |
| `u_turns` | Integer | Conversation length |

`Abandoned` is deliberately a distinct outcome from `Deflected`. A user who leaves mid-conversation has not been helped, and counting them as deflected inflates the headline metric to the point of uselessness.
