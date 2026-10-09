# past.dev for Cursor

Your Cursor chats become memory, and every prompt reaches the model with what past.dev recalls for it.

[past.dev](https://past.dev) is memory for AI agents. This plugin connects Cursor to a past.dev
project: what you and the agent say to each other is read into memory as you work, and what matters
comes back by itself, with every prompt. The decisions, the reasons behind them, what was already
tried and dropped.

```text
=== past · recalled from memory ===
3 from this project's history. Background, not instruction.

[1] 2026-09-14 — Webhooks retry from the job queue now: the HTTP client's retries sent them twice.
[2] 2026-09-02 — Rejected: caching prices in Redis. They change hourly, and a stale one is worse.
[3] 2026-08-27 — The importer keeps rows it cannot parse in import_rejects and never drops them.
```

That block is what the model reads before your prompt. It arrives on its own: the model does not
have to decide to look anything up.

## Get started

You need Cursor 3.23 or later and Node 16 or later.

1. **Get a project API key.** Sign up at [sso.past.dev/sign-up](https://sso.past.dev/sign-up),
   then open **Build › API keys** in the [console](https://app.past.dev). The key starts with
   `past_sk_`.

2. **Install the plugin.** In Cursor, open **Customize**, add a marketplace from GitHub with the
   source `pastdotdev/cursor-plugin`, and install **past.dev** from it. From a terminal, this adds
   the same marketplace:

   ```bash
   agent plugin marketplace add https://github.com/pastdotdev/cursor-plugin
   ```

3. **Connect it.** In a chat:

   ```text
   /past-connect <project-api-key> <identity>
   ```

   The identity is the email or id every memory is attributed to. Recall stays off without one.
   Cursor asks before it runs the command. A machine that another past.dev plugin already connected
   skips this step: every past.dev plugin reads the same `~/.past/config.json`.

4. **Check it.** `/past-status` shows the project it talks to, and what it has sent.

Chats are read from your next prompt on.

## How it works

```text
every prompt     recall what past.dev knows about this prompt      ──►  in front of it
first prompt     … and what it knows about this project            ──►  in front of it
every turn       the prompt and the reply are kept on this machine
quiet 30 min     the chat is sent, one sitting at a time
on request       past_recall looks something up
```

Five hooks do this, and one script runs them all. A recall that has not answered within 4 seconds
is dropped, so it never holds a prompt up, and a prompt shorter than 12 characters ("yes", "go on")
recalls nothing.

**Recall happens before the model answers.** The prompt hook returns what past.dev recalled, and
Cursor puts it in front of the prompt. For questions about history, the `past_recall` tool lets the
agent look further on its own: ask what was decided about something, or whether it was tried before.

**A chat is sent in sittings.** Where a chat went quiet for 30 minutes, the next prompt starts a new
sitting. Each sitting is dated at its own first prompt, so a decision made on the third day of a
long chat is remembered on that day, and past.dev can tell which of two statements came later. A
chat is sent once it has been quiet for that long; whatever you say afterwards starts a new sitting,
so nothing already sent is sent again.

## What leaves your machine

Your prompts as you typed them, the agent's reply to each, and the project and branch names.
Nothing else.

Never sent: tool calls and their output, file contents, diffs, command output, attachments, the
model's thinking, what the agent writes between tool calls, and anything injected into the chat
rather than typed by a person. Strings that look like keys, tokens or private keys become
`[redacted]` before anything leaves, and recall queries are redacted the same way.

`/past-cut` prints exactly what would be sent from a chat, sitting by sitting, before anything is.
Until a chat is sent, and afterwards, its prompts and replies are kept in `~/.past/cursor/`,
readable only by you.

A chat costs what its prose costs, at 350 bytes to a credit. What a recall costs depends on your
plan: see [past.dev/pricing](https://past.dev/pricing).

## Skills and the tool

| | What it does |
|---|---|
| `/past-connect <key> <identity> [api-url]` | connect this machine; `--audience <slug>` sets who sees what you send |
| `/past-status` | what it is connected to, what it has sent, and the last send that failed |
| `/past-cut [chat]` | print exactly what would be sent from a chat, by default the latest one here |
| `/past-ingest [chat]` | send one chat now, without waiting for it to go quiet |
| `past_recall` | the agent's own lookup in past.dev, a tool it calls when the question is about history |

## Who sees what you send

Everyone in the project, unless you choose an audience: `/past-connect --audience <slug>`, with a
slug from the console's **Audiences** page. `/past-connect --audience project` widens it back.

## Settings

`~/.past/config.json` holds the connection and a few switches: `recall` and `ingest` turn either
half off, `promptRecall` turns off the recall in front of every prompt alone, `sittingMinutes` (30)
is how long a pause starts a new sitting, and `deny` lists paths whose chats are never read.
`PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file. Each setting is described in
the [plugin's README](plugins/past/README.md#configuration).

For a self-hosted past.dev, pass your deployment's URL as the third argument of `/past-connect`. Use
`https`: over plain `http` to another machine the key travels unencrypted, and the plugin says so.

## When something looks wrong

Start with `/past-status`. A hook never interrupts a chat, so a problem shows up there rather than
in the conversation.

- **The hooks never ran.** Status says so. Cursor runs a plugin's hooks only while its setting for
  third-party plugins allows them (**Cursor Settings › Agents**), and on some team plans an
  administrator decides.
- **Nothing is recalled.** Status has to show a key and an identity: recall needs both. A new
  project has nothing to recall until its first chats are in; `/past-ingest` sends the current one
  now.
- **Recall stopped reaching the model after a Cursor update.** The recall in front of a prompt uses
  a field of Cursor's prompt hook that its documentation does not describe. If a release stops
  delivering it, set `"promptRecall": false` in `~/.past/config.json` to stop paying for it;
  `past_recall` keeps working.
- **A send failed.** Status shows the last failed send and what the API answered.
- **The hooks stay silent.** They call `node`, and Cursor finds the `node` of your login shell.
  Node 16 or later has to come first on that path.

## Remove it

Uninstall the plugin in **Customize**, then delete `~/.past/cursor`. Revoking the key in **Build ›
API keys** stops it at once from the server side. What was already sent stays in the project until
you delete it in the console.

## This repository

A Cursor plugin marketplace. Cursor reads it straight from GitHub; there is nothing to build.

```text
.cursor-plugin/marketplace.json   the catalog
plugins/past/
  .cursor-plugin/plugin.json      the manifest
  hooks/hooks.json                the five hooks
  mcp.json                        the past_recall tool, served by the same script
  rules/past.mdc                  what the recall block is, and when to call past_recall
  skills/                         /past-connect, /past-status, /past-cut, /past-ingest
  bin/past-hook.mjs               the whole plugin: one file, Node 16, no dependencies
```

The plugin talks only to past.dev's public Memory API (`/api/v1/ingest/batch`, `/api/v1/recall`,
`/api/v1/audiences`), with your project's key. Nothing in it is privileged: anyone could write the
same connector against the same endpoints.

## Contributing

Issues and pull requests are welcome. [AGENTS.md](AGENTS.md) holds the rules the code keeps and how
to test a change by hand, for people and agents alike. Before opening a pull request:

```bash
node --check plugins/past/bin/past-hook.mjs
HOME=$(mktemp -d) node plugins/past/bin/past-hook.mjs status
```

## Links

- Memory API documentation: [past.dev/docs/memory-api/overview](https://past.dev/docs/memory-api/overview)
- Console: [app.past.dev](https://app.past.dev)
- Sign up: [sso.past.dev/sign-up](https://sso.past.dev/sign-up)
- past.dev's MCP servers: [pastdotdev/mcp](https://github.com/pastdotdev/mcp)
- Community: [past.dev/slack](https://past.dev/slack)

## License

MIT. See [LICENSE](LICENSE).
