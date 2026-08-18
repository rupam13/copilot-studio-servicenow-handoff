# ServiceNow API Contracts

Every call uses the **Table API** over OAuth 2.0. No basic auth, no embedded credentials.

---

## 1. Resolve caller

Converts an Entra UPN to a ServiceNow `sys_user` sys_id. Run this **server-side in the flow**, never trust a caller value from the conversation.

```http
GET /api/now/table/sys_user
  ?sysparm_query=email=<UPN>^active=true
  &sysparm_fields=sys_id,name,email,department
  &sysparm_limit=1
Authorization: Bearer <ACCESS_TOKEN>
Accept: application/json
```

**Response**

```json
{
  "result": [
    {
      "sys_id": "<USER_SYS_ID>",
      "name": "Example User",
      "email": "user@<DOMAIN>",
      "department": "<DEPT_SYS_ID>"
    }
  ]
}
```

**Empty result** means the user exists in Entra but not in ServiceNow. Do not fall back to a generic service account, that silently corrupts caller attribution. Fail loudly and raise the ticket against the IT-Unmapped queue.

---

## 2. Create incident with context

```http
POST /api/now/table/incident
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
```

```json
{
  "caller_id": "<USER_SYS_ID>",
  "short_description": "[Copilot] VPN disconnects every 10 minutes",
  "description": "--- Copilot Studio transcript ---\nuser: ...\nagent: ...",
  "contact_type": "virtual_agent",
  "impact": 2,
  "urgency": 2,
  "u_conversation_id": "<CONVERSATION_ID>",
  "u_bot_attempted_kb": "KB0010023;KB0010451",
  "u_bot_confidence": "0.42",
  "u_deflection_attempts": "2"
}
```

**Response** returns `number` (e.g. `INC0012345`) and `sys_id`. Surface the `number` to the user, keep the `sys_id` for follow-up calls.

---

## 3. Search knowledge

Used when Knowledge is queried live rather than pre-indexed as a Copilot Studio knowledge source.

```http
GET /api/now/table/kb_knowledge
  ?sysparm_query=workflow_state=published^active=true^short_descriptionLIKE<TERM>
  &sysparm_fields=number,short_description,text,sys_id
  &sysparm_limit=5
```

> Prefer indexing Knowledge as a Copilot Studio knowledge source. Live search costs a **5-credit tool call** per turn; a grounded generative answer costs **2 credits**. At scale the difference dominates the bill.

---

## 4. Log a deflection

Deflections must be recorded explicitly. "Tickets not created" is not a measurement, it is an absence.

```http
POST /api/now/table/u_bot_deflection
```

```json
{
  "u_conversation_id": "<CONVERSATION_ID>",
  "u_caller": "<USER_SYS_ID>",
  "u_intent": "vpn disconnects",
  "u_kb_articles": "KB0010023",
  "u_outcome": "Deflected",
  "u_turns": 3
}
```

---

## 5. Create a catalog request (Pattern 2: Fulfil)

```http
POST /api/sn_sc/servicecatalog/items/<ITEM_SYS_ID>/order_now
```

```json
{
  "sysparm_quantity": "1",
  "variables": {
    "requested_for": "<USER_SYS_ID>",
    "software_title": "<TITLE>",
    "business_justification": "<JUSTIFICATION>"
  }
}
```

Only wire catalog items that are **fully automated end-to-end**. A catalog item that still needs manual fulfilment is a ticket with extra steps, and users learn to bypass the bot.

---

## Error Handling

| Status | Meaning | Flow behaviour |
|---|---|---|
| `401` | Token expired | Refresh once, then fail |
| `403` | ACL denies the operation | Fail loudly, do not silently downgrade the caller |
| `404` | Table or record missing | Fail, alert, do not create in a fallback table |
| `429` | Rate limited | Exponential backoff, 4 attempts, PT10S base |
| `500` | Instance error | Retry twice, then queue for replay |

**Never** swallow a `403` by retrying as the integration service account. That converts an access-control failure into a privilege escalation.
