---
name: past-ingest
description: Send one Cursor chat to past.dev now, without waiting for it to go quiet. Run it only when the person asks for it.
disable-model-invocation: true
---

# Send a chat to past.dev now

Run this command in the terminal. Give the chat's id, or its first characters, when the person named one. Without an id it sends the chat most recently active in this workspace. Show what it prints, word for word.

```bash
node "$HOME/.past/cursor/past.mjs" ingest [chat]
```

If the command cannot reach the network from where it runs, ask the person to allow it, then run it again.
