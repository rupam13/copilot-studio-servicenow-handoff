# Deflection Strategy

## The Core Metric

```
                       deflected
Deflection rate = ----------------------
                  deflected + escalated
```

Exclude `Abandoned` from **both** numerator and denominator. A user who walked away was not helped, and counting them as a success is the single most common way this metric is quietly gamed.

## What Good Looks Like

| Maturity | Deflection rate | Typical characteristic |
|---|---|---|
| Launch | 15-25% | KB is stale, intents are narrow |
| Established | 40-55% | Top 20 intents covered, KB actively maintained |
| Mature | 60-70% | Catalog fulfilment automated, KB written *for* the agent |
| Suspicious | > 85% | Almost always a measurement error, check for abandoned-as-deflected |

Above 85%, verify before celebrating. Common causes: abandoned sessions counted as deflected, the agent claiming resolution without confirming, or escalations routed outside the tracked path.

## Why the KB Is the Bottleneck

Most "the bot is bad" complaints resolve to missing or badly scoped knowledge wearing a model costume. Before tuning instructions, check:

1. **Is the article published and active?** Draft articles are invisible to the connector.
2. **Is it written for a reader or for a search engine?** Articles that open with 200 words of policy preamble ground badly. The answer must be near the top.
3. **Does it assume screenshots?** Images do not ground. An article whose entire procedure lives in a screenshot contributes nothing.
4. **Is it scoped to a category the agent can reach?** 500 knowledge sources is the ceiling, so KB is filtered by category, not loaded wholesale.

## The Two-Attempt Rule

`MaxDeflectionAttempts` defaults to **2**.

| Setting | Effect |
|---|---|
| 1 | Escalates on first miss, wastes human capacity on solvable issues |
| **2** | One retry with reframing, then hand off |
| 3+ | Measurable user frustration, abandonment rises sharply |

Exposed as an environment variable so it can be tuned per queue. A password-reset queue tolerates 3; a payroll queue should escalate at 1.

## Writing KB Articles That Ground Well

| Do | Don't |
|---|---|
| Lead with the resolution steps | Open with scope and policy preamble |
| Use numbered, imperative steps | Describe the fix in prose |
| State the symptom in the user's words | Use only internal jargon in the title |
| Keep one problem per article | Bundle 6 unrelated fixes into a mega-article |
| Put the answer in text | Put the answer only in a screenshot |

## Measuring Honestly

Track these together. Any one alone is misleading:

| Metric | Healthy direction | Warns you about |
|---|---|---|
| Deflection rate | Up | Headline effectiveness |
| Abandonment rate | Down | Users giving up rather than escalating |
| Reopen rate on deflected intents | Down | False deflections, the user came back |
| Escalated ticket handle time | Down | Whether the transcript context is actually useful |
| CSAT on deflected sessions | Up | Whether "resolved" meant resolved |

**Reopen rate is the honesty check.** A deflection that produces a ticket for the same intent 48 hours later was never a deflection.
