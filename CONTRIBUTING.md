# Contributing

## Ground rules

1. **No instance URLs, client IDs, secrets, sys_ids, or flow GUIDs.** Use placeholders: `<SN_INSTANCE>`, `<CLIENT_ID>`, `<USER_SYS_ID>`, `<FLOW_ID>`. PRs containing real identifiers will be closed without merge.
2. **No real ticket data.** Example transcripts must be synthetic.
3. **Cite platform behaviour.** Both Copilot Studio and ServiceNow ship changes continuously. When correcting a limit, link the vendor documentation.
4. **Keep the rationale.** This repo explains *why* each choice was made. Changes that strip reasoning will be asked to restore it.

## Pull requests

- One logical change per PR
- Descriptive commit messages, explain the why
- Update the affected README table in the same PR
