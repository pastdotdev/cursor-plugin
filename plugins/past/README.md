# past.dev for Cursor

Your Cursor chats become memory, and every prompt reaches the model with what past.dev recalls for it.

## Install

In Cursor, open **Customize**, add a marketplace from GitHub with the source
`pastdotdev/cursor-plugin`, and install **past.dev** from it. From a terminal, this adds the same
marketplace:

```bash
agent plugin marketplace add https://github.com/pastdotdev/cursor-plugin
```

Then connect it, in a chat:

```text
/past-connect <project-api-key> <identity> [api-url]
```

The project API key starts with `past_sk_` and comes from **Build › API keys** in the
[console](https://app.past.dev) (not the project's id, and not the organization's `past_mk_`
management key); the identity is who every memory is attributed to, and the API URL is for
self-host only. Cursor asks before it runs the command. A machine another past.dev plugin already
connected needs nothing more: they share `~/.past/config.json`.

## What it does

| Hook | When | What |
|---|---|---|
| `workspaceOpen` | Cursor opens a workspace | sends the chats that went quiet while Cursor was closed |
| `sessionStart` | a chat opens | the same, for a chat opened later |
| `beforeSubmitPrompt` | every prompt | keeps the prompt, recalls against it and puts the result in front of it; the chat's first prompt also gets the project's history |
| `afterAgentResponse` | the agent replies | keeps the reply |
| `stop` | a turn ends | marks the start of the quiet that decides when the chat is sent |

**A chat is sent in sittings.** A new sitting starts at the first prompt after the chat went quiet
for 30 minutes (`sittingMinutes`); a turn the agent is still working on is never quiet, however long
it runs. Each sitting is dated at its own first prompt, and what past.dev remembers takes the date
of the sitting it came from. A chat is sent once it has been quiet that long, by one small process
that watches every chat and leaves when none is waiting. Whatever is said afterwards starts a new
sitting, so a sitting already in past.dev is never sent again.

The plugin also serves `past_recall`, a tool the agent calls to look something up in past.dev, and a
short rule that tells it what the recall block is and when the tool helps. Cursor's Ask mode offers
no tools, so there recall arrives with the prompt alone.

## What leaves this machine

The prompts as you typed them, the agent's reply to each, and the project and branch names. That
is all.

Not sent: tool calls and their output, file contents, diffs, command output, attachments, the
model's thinking, what the agent writes between tool calls, and anything injected into the chat.
Strings that look like keys, tokens or private keys are replaced with `[redacted]` before anything
is sent.

Run `/past-cut` to read exactly what would be sent, before it is.

## Skills

| Skill | Does |
|---|---|
| `/past-connect` | store the key, identity and audience in `~/.past/config.json`, mode `600` |
| `/past-status` | what it is connected to, what it has sent, and the last send that failed |
| `/past-cut [chat]` | print exactly what would be sent from a chat, by default the latest one here |
| `/past-ingest [chat]` | send one chat now, without waiting for it to go quiet |

Without an audience everyone in the project sees what you send. `/past-connect --audience <slug>`
narrows it to an audience from the console; `--audience project` widens it back.

## Configuration

`~/.past/config.json`:

```json
{
  "apiKey": "past_sk_…",
  "identity": "you@example.com",
  "apiUrl": "https://api.past.dev",
  "audience": "developers",
  "recall": true,
  "promptRecall": true,
  "ingest": true,
  "sittingMinutes": 30,
  "idleMinutes": 720,
  "deny": ["/Users/you/clients/acme"]
}
```

- `audience` is an audience slug from the console; leave it out and the whole project sees what
  you send.
- `recall: false` stops the reading and keeps the sending. `ingest: false` does the opposite, and
  nothing is kept on this machine either.
- `promptRecall: false` stops the recall in front of every prompt and keeps `past_recall`. Cursor
  delivers that recall through a field of its prompt hook that its documentation does not
  describe; if a Cursor release stops delivering it, this stops the spend.
- `sittingMinutes` is how long a chat stays quiet before it is sent, and before what follows starts
  a new sitting (30 by default, 5 at least). A chat keeps the length it was first sent with.
- `idleMinutes` is how long a turn that never ended holds its chat back (720, twelve hours): Cursor
  quit in the middle of it.
- `deny` holds path fragments. A chat whose workspace matches one is ignored completely: nothing
  is read and nothing is sent.
- `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file.
- Self-host: point `apiUrl` at your own deployment. Nothing else changes. Use `https`: over plain
  `http` to another machine the key travels unencrypted, and `connect` and `status` warn about it.

## Requirements

Cursor 3.23 or later, and Node 16 or later on the path of your login shell.

## Where it keeps things

Everything lives in `~/.past/`, a folder only you can open: `config.json` (the connection),
`state.json` (what was sent, as content hashes, under `hosts.cursor`) and `cursor/`, which holds one
journal per chat (its prompts and replies, with their times), the sender's lock, and `past.mjs`, the
fixed path the skills run.

## Removing it

Uninstall the plugin in **Customize**, then delete `~/.past/cursor`. Revoking the key in **Build ›
API keys** stops it immediately from the server side. What was already sent stays until you delete
it in the console.

## License

MIT. See [LICENSE](LICENSE).
