---
name: past-connect
description: Connect this machine to a past.dev project, with a project API key and the identity every memory is attributed to. Run it only when the person asks for it.
disable-model-invocation: true
---

# Connect past.dev

Run this command in the terminal, with the person's own values in place of the placeholders. Show what it prints, word for word.

```bash
node "$HOME/.past/cursor/past.mjs" connect <project-api-key> <identity> [api-url] [--audience <slug>]
```

- The project API key starts with `past_sk_`. It comes from the past.dev console, under Build › API keys.
- The identity is who every memory is attributed to: an email or an id.
- `--audience <slug>` with no key changes who sees what is sent from now on. `--audience project` gives it back to the whole project.
- Never repeat the key in a reply, and never write it into a file of the repository.

If the command cannot reach the network from where it runs, ask the person to allow it, then run it again.
