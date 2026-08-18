# Metrics

## The Dashboard That Matters

Five numbers. If you cannot show these, you cannot claim the agent is working.

| Metric | Definition | Target |
|---|---|---|
| Deflection rate | `deflected / (deflected + escalated)` | > 40% by month 3 |
| Abandonment rate | `abandoned / total sessions` | < 10% |
| Reopen rate | Same caller + intent within 48 h of a deflection | < 5% |
| Handle time delta | Bot-origin vs phone-origin ticket handle time | > 20% faster |
| Cost per resolution | Credits consumed / resolutions | Trending down |

---

## Application Insights (KQL)

### Deflection funnel

```kusto
customEvents
| where timestamp > ago(30d)
| where name in ("BotSessionStart", "BotDeflected", "BotEscalated")
| extend conversationId = tostring(customDimensions.ConversationId)
| summarize
      started   = countif(name == "BotSessionStart"),
      deflected = countif(name == "BotDeflected"),
      escalated = countif(name == "BotEscalated")
  by bin(timestamp, 1d)
| extend abandoned = started - deflected - escalated
| extend deflectionRate = round(100.0 * deflected / (deflected + escalated), 1)
| extend abandonRate    = round(100.0 * abandoned / started, 1)
| order by timestamp asc
```

### Intents that escalate most

The backlog for KB authoring, in priority order.

```kusto
customEvents
| where timestamp > ago(30d)
| where name == "BotEscalated"
| extend intent = tostring(customDimensions.Intent)
| summarize escalations = count(),
            avgTurns = avg(todouble(customDimensions.Turns))
  by intent
| top 20 by escalations desc
```

### Credit consumption by pattern

```kusto
customEvents
| where timestamp > ago(30d)
| where name in ("GenerativeAnswer", "ToolCall", "ClassicAnswer")
| extend credits = case(
      name == "ClassicAnswer",    1,
      name == "GenerativeAnswer", 2,
      name == "ToolCall",         5,
      0)
| summarize totalCredits = sum(credits), calls = count()
  by name, bin(timestamp, 1d)
| order by timestamp asc
```

> This query is why deflection is cheaper than escalation at every scale: a grounded answer costs **2 credits**, a ticket-creating tool call costs **5**, plus the human handle time behind it.

---

## ServiceNow Reporting

### Bot-origin volume

```sql
-- Report on: incident
-- Condition: contact_type = virtual_agent
-- Group by: assignment_group
-- Trend: created, weekly
```

### Handle-time comparison

The number that proves the transcript is worth writing.

```sql
-- Report on: incident
-- Condition: opened_at >= last 30 days
-- Group by: contact_type
-- Aggregate: AVG(calendar_duration) where state = Closed
```

If bot-origin tickets are **not** faster to handle than phone-origin tickets, the context payload is not being read. Check that `description` is visible in the agent workspace view, and that `u_bot_attempted_kb` is on the form. Context nobody sees is context that does not exist.

---

## Weekly Review Questions

1. Which 5 intents escalated most? Do KB articles exist for them?
2. Did abandonment rise? If so, at which turn number do users leave?
3. Any deflected intents with reopens? Those are false deflections.
4. Is credit spend per resolution trending down as KB coverage grows?
5. Did `u_bot_version` change this week? Correlate against every metric above.
