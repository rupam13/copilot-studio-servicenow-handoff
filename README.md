<div align="center">

# Copilot Studio to ServiceNow — Handoff & ITSM Automation

**Deflect what you can. Hand off what you can't — with the full transcript intact.**

[![Platform](https://img.shields.io/badge/Platform-Copilot%20Studio%202025--2026-orange?style=flat-square)](https://copilotstudio.microsoft.com)
[![ITSM](https://img.shields.io/badge/ITSM-ServiceNow-81B5A1?style=flat-square)](https://www.servicenow.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

*Companion project to the [Copilot Studio Practitioner Playbook](https://github.com/rupam13/copilot-studio).*

</div>

---

## Why This Exists

Most Copilot Studio to ServiceNow integrations fail in one of two ways:

1. **The bot creates a ticket for everything.** No deflection, so the agent adds a step rather than removing one. Ticket volume goes *up*.
2. **The handoff loses the conversation.** The human agent opens the ticket and sees "User needs help." They ask the user to repeat everything. The user concludes the bot wasted their time.

This repo addresses both: a **deflection-first topic design**, and a **context-preserving handoff** that writes the full transcript, the attempted resolutions, and the confidence trail into the ServiceNow record before a human ever sees it.

---

## The Three Patterns

| Pattern | When to use | Outcome |
|---|---|---|
| **1. Deflect** | Knowledge exists and confidence is high | No ticket. Logged as deflection. |
| **2. Fulfil** | Request is a known, automatable catalog item | Ticket created *and closed* by the agent |
| **3. Hand off** | Novel, sensitive, or repeatedly failed | Live agent transfer with full context |

```
                    User query
                        |
                        v
            [ Generative Answers ]
            grounded on SN Knowledge
                        |
         +--------------+--------------+
         |              |              |
    confidence     catalog match   no match / low
      high              |          confidence / 2 failed turns
         |              |              |
         v              v              v
    [ DEFLECT ]   [ FULFIL ]      [ HAND OFF ]
    log + close   RITM created    incident + transcript
                  and fulfilled   + live agent transfer
```

---

## What Gets Written on Handoff

The single highest-value design decision in this repo. Before the human agent is engaged, the incident already contains:

| Field | Source | Why the human agent needs it |
|---|---|---|
| `short_description` | Agent-summarised intent | Triage without reading the transcript |
| `description` | Full turn-by-turn transcript | The actual context |
| `u_bot_attempted_kb` | KB article numbers already shown | Stops them re-sending the same article |
| `u_bot_confidence` | Final orchestration confidence | Signals whether intent was even understood |
| `u_deflection_attempts` | Count of resolution attempts | Measures user frustration |
| `caller_id` | Entra UPN, resolved via `sys_user` | Never taken from user input |
| `u_conversation_id` | Copilot Studio conversation ID | Joins back to Application Insights |
| `contact_type` | `virtual_agent` | Segments bot-origin tickets in reporting |

> The `u_bot_attempted_kb` field is what stops the most common complaint about bot handoffs: the human re-sending an article the user has already read and rejected.

---

## Repo Structure

```
copilot-studio-servicenow-handoff/
|
+-- topics/
|   +-- DeflectWithKnowledge.topic.yaml
|   +-- CreateIncident.topic.yaml
|   +-- EscalateToLiveAgent.topic.yaml
|
+-- api/
|   +-- servicenow-endpoints.md      # Table API contracts used
|   +-- field-mapping.md             # Copilot Studio -> ServiceNow mapping
|
+-- docs/
    +-- deflection-strategy.md
    +-- authentication.md
    +-- metrics.md
```

---

## Quick Start

1. **Register the app in ServiceNow** — OAuth 2.0, client credentials. See [`docs/authentication.md`](docs/authentication.md).
2. **Create the custom fields** on `incident` per [`api/field-mapping.md`](api/field-mapping.md).
3. **Add ServiceNow Knowledge as a knowledge source** in Copilot Studio (graph connector or indexed export).
4. **Import the topics** from `topics/`.
5. **Set the deflection threshold** — `DeflectionConfidenceThreshold`, default `0.75`.

> Every identifier here is a placeholder: `<SN_INSTANCE>`, `<CLIENT_ID>`, `<SCOPE>`. No credentials are committed.

---

## Design Decisions

**Why not the out-of-box ServiceNow connector for everything?**
The certified connector is fine for simple CRUD. It does not let you control the *transcript payload shape*, and it authenticates as a single service account, which destroys per-user entitlement checks. This repo uses the connector for catalog operations and a custom connector with OAuth on-behalf-of for anything user-scoped.

**Why is `caller_id` resolved server-side?**
If the caller is taken from anything the user typed, any user can raise a ticket as anyone else. The UPN comes from `System.User.PrincipalName`, then is resolved against `sys_user` inside the flow.

**Why count deflection attempts before escalating?**
Escalating on the first miss wastes the agent. Escalating on the fifth infuriates the user. Two failed resolution turns is the empirically reasonable default, exposed as a variable so it can be tuned per queue.

**Why `contact_type = virtual_agent`?**
Without it, bot-originated tickets are indistinguishable from phone tickets and the deflection rate becomes unmeasurable. You cannot improve what you cannot segment.

**Why write the transcript to `description` and not an attachment?**
Attachments are not indexed by ServiceNow's list search and are invisible in most agent workspace views. A transcript nobody reads is not context.

---

## Limits That Shaped This Design

| Constraint | Value | Consequence |
|---|---|---|
| Agent flow sync timeout | 100-120 s | Ticket creation is sync; live-agent transfer is async |
| Connector payload | 5 MB public cloud / 450 KB GCC | Transcripts truncated to last 30 turns in GCC |
| Knowledge sources per agent | 500 | KB is filtered by category, not loaded wholesale |
| Copilot Credits, generative answer | 2 credits | Deflection is cheaper than escalation at every scale |
| Copilot Credits, tool call | 5 credits | Ticket creation costs 2.5x an answer, budget accordingly |
| API calls per connection | 300 / 60 s | Bulk sync jobs throttled to 250/min with backoff |

---

## License

[MIT](LICENSE). ServiceNow and Microsoft product names and marks belong to their respective owners.

---

<div align="center">

**Built by [Rupam Wadibhasme](https://github.com/rupam13)**

*"Deflection is the only ITSM metric that compounds."*

</div>
