# past.dev for Cursor: working on the plugin

The past.dev plugin for Cursor, and the marketplace that carries it. It runs on the developer's
machine and talks to past.dev's public Memory API. Nothing here is privileged: anyone can write the
same connector against the same endpoints with the same key. What a user reads is in `README.md`,
and in full in `plugins/past/README.md`; this file is the rules the code keeps, for whoever changes
it.

## How it fits together

```
Cursor
  hooks/hooks.json   five hooks
  mcp.json           past_recall, served by the same script
  rules/past.mdc     what the recall block is, and when to call past_recall
  skills/*           four skills, run through ~/.past/cursor/past.mjs
            |
  bin/past-hook.mjs <mode>
            |
  workspace opens   wake      writes the launcher; starts the sender for what went overdue
  chat opens        start     the same
  every prompt      prompt    journals the prompt; recalls; returns the block as additional_context
  every reply       reply     journals the reply
  end of a turn     stop      journals the end; makes sure the sender runs
  detached          send      the sender: sends each chat once it is quiet for its sitting length
  MCP server        mcp       past_recall over stdio
            |
  ~/.past/config.json            the connection, shared with every past.dev plugin
  ~/.past/cursor/<chat>.jsonl    one journal per chat: prompts, replies, ends of turns, timed
  ~/.past/cursor/past.mjs        the launcher the skills run
  ~/.past/state.json             what was sent, under hosts.cursor
  ~/.past/hosts/cursor.json      when the hooks last ran, for status
            |
  POST /api/v1/recall · POST /api/v1/ingest/batch · GET /api/v1/audiences/{slug}
```

```
.cursor-plugin/marketplace.json   the catalog: this repository is the marketplace
plugins/past/
  .cursor-plugin/plugin.json      the manifest: names the hooks, the MCP server, the rules and the skills
  hooks/hooks.json                workspaceOpen, sessionStart, beforeSubmitPrompt, afterAgentResponse, stop
  mcp.json                        past_recall: the script in its mcp mode
  rules/past.mdc                  the always-applied rule
  skills/                         past-connect, past-status, past-cut, past-ingest
  bin/past-hook.mjs               the whole plugin, one file, no dependencies
  assets/logo.svg                 the logo the manifest names
  LICENSE                         the same license, for an install that copies this folder alone
.github/workflows/check.yml       every pull request: the script parses and runs, the manifests hold together
```

## The rules this code keeps

1. **A hook never breaks the chat.** Every mode exits 0, prints valid JSON or nothing, and swallows
   its own failures. The prompt hook always answers `{"continue": true}`, with the recall block
   beside it when there is one. `/past-status` is where a person learns something is wrong, so a
   failed send is kept in `state.json` (`lastError`) for it.
2. **Only prose travels.** The journal holds three kinds of entry: the prompt as the person typed
   it (`beforeSubmitPrompt.prompt`), the reply as Cursor shows it (`afterAgentResponse.text`, the
   turn's final message) and the end of a turn (`stop`). Tool calls and their output, file
   contents, attachments, the model's thinking and injected context never reach it, and past.dev's
   own block is dropped by its header. A prompt that only runs one of the plugin's skills
   (`/past-…`) is not conversation, and neither is its answer: Cursor gives a prompt and its reply
   the same `generation_id`, and a reply whose prompt was never recorded is left out (a reply with
   no id counts only while a turn is open). A chat whose journal records no prompt (a subagent's)
   is never sent, and a chat from a window with no folder open is labelled `no folder`.
3. **The journal is the conversation, timed by this machine's clock.** Every entry carries the
   time it arrived. Cursor's transcript file is not read: Cursor does not describe its format as
   stable, it holds no time per line, and with some models it merges the reasoning into the reply.
4. **A chat is sent once it is quiet, by one sender.** Cursor never says a chat is over: its
   `sessionEnd` fires only when a window closes, too late to start a command. So `stop`, `start`
   and `wake` make sure one detached sender runs (`send` mode). Once a minute, on the wall clock,
   it reads the journals written since their last send and sends a chat once it has been quiet for
   its sitting length. Quiet starts at the last entry of a finished turn; a turn whose prompt has
   no reply or end yet holds its chat for `idleMinutes` (Cursor quit in the middle). The sender
   leaves when nothing waits, and the next hook starts it again.
   - One sender at a time: it holds `~/.past/cursor/sender.lock` (a directory, since mkdir is
     atomic) for its life and names itself in `sender.json`. A lock whose holder is gone is taken
     over; one younger than ten seconds belongs to a sender still starting.
   - `SENDER_VERSION` is bumped whenever the sender changes: a sender that an older version
     started is replaced, because it is a long-lived process running the code it started with.
   - A chat whose send failed is left for the next sender run, so a revoked key never keeps a
     process asking.
5. **A sitting already sent never changes.** A sitting starts at the first prompt after the chat
   was quiet for `sittingMinutes` (30 by default, 5 at least), and a chat is sent only once it has
   been that quiet, so whatever is said afterwards starts a new sitting and nothing sent is paid
   for twice. `ingest` sends a chat at once, so it is the one way a sitting can be sent again.
   `state.json` keeps one hash per sitting and pins `sittingMinutes` per chat at its first send,
   so a changed value never moves the boundaries of a chat already in past.dev. Every new sitting of
   a chat goes in one `/api/v1/ingest/batch` call.
6. **The recall rides on an undocumented field.** Cursor's hooks reference gives
   `beforeSubmitPrompt` two outputs, `continue` and `user_message`, yet Cursor delivers the hook's
   `additional_context` to the model before it answers. That is what puts recall in front of every
   prompt, and the chat's first prompt also carries the project's brief, since `sessionStart`'s
   documented context is lost to a race. If a release stops delivering it, nothing says so: the hook
   still recalls and spends. So the check after each Cursor release (below) is part of the job, and
   `promptRecall: false` turns that recall off while `past_recall` keeps working.
7. **A recall query is redacted like the content it would send.** It crosses the network exactly
   as ingested content does, and both go through `redact()`.
8. **`state.json` changes only through `updateState`.** Hooks, the sender and commands are
   separate processes. `updateState` takes a lock (`~/.past/state.lock`, a directory, taken over
   after 5 s, skipped after 3 s so a hook never hangs), re-reads the file, applies the change and
   renames the new file into place. A file that does not parse is read again a few times and, if it
   still does not, left as it is.
9. **No dependencies.** Node 16 or later, standard library only, so installing is a clone. `fetch`
   is used where Node has it and the `http` modules where it does not: Cursor started from the Dock
   finds the `node` of a login shell, which is often an old one. On Node 16 a stalled `http` call
   is destroyed when its time is up. Nothing in the script is newer than Node 14 can parse (no
   `||=`), so on an older Node the hooks stay silent rather than fail.
10. **What a chat renders does not change by a byte.** The heading and the speaker names that
    `renderSitting` prints are part of each sitting's hash, and so are `SOURCE`, `AGENT` and
    `SPEAKER` at the top of the script: a changed byte sends every sitting of a chat that grows
    afterwards again, as a paid revision. `sittingId` and `sittingsOf` are fixed for a related
    reason: changed, they move the boundaries of every chat that grows afterwards.
11. **`~/.past` is the person's alone.** The folder is `700` and every file in it is created `600`,
    the journals included. A plain-http `apiUrl` on another machine sends the key unencrypted:
    `connect` warns, and `status` says so on its first line.
12. **The product is written past.dev.** Every sentence a person reads says past.dev: the READMEs,
    the manifest's display name, the skills, the rule, what a mode prints. Identifiers keep `past`:
    the plugin's name, the skills (`/past-status`), the tool (`past_recall`), `~/.past`, the key
    prefix `past_sk_`, the data point ids and the recall block's header,
    `=== past · recalled from memory ===`, which the journal reader drops by that exact text.

## Beside other past.dev plugins

`~/.past` can be shared with another past.dev plugin on the same machine. `config.json` is then one
connection for all of them. This plugin keeps its chats under `hosts.cursor` in `state.json`, and
everything else of its own in `~/.past/cursor/` and `~/.past/hosts/cursor.json`. The rest of
`state.json` is never read here, and every write puts it back as it was (rule 8).

## Cursor

- **Hooks run in the plugin's folder.** So the project and the branch come from the workspace
  Cursor names (`CURSOR_PROJECT_DIR`, or the first of `workspace_roots`), never from the hook's
  working directory. The branch is read from the workspace's `.git`, a worktree's `.git` file
  included.
- **A skill's command learns nothing about the plugin's folder.** So every hook writes
  `~/.past/cursor/past.mjs`, a two-line launcher that imports the installed script, and the skills
  run `node "$HOME/.past/cursor/past.mjs" <mode>`. A skill's own prompt runs the prompt hook first,
  so the launcher exists before its command does.
- **Ask mode removes MCP tools.** There the prompt hook is the only way recall reaches the model.
- **Cursor shows the agent only tool names until it opens one.** The always-applied rule says what
  the recall block is and when `past_recall` helps.
- **The two skills that take a key or send data run only when named** (`past-connect`,
  `past-ingest`: `disable-model-invocation: true`).
- **Windows** runs hooks through PowerShell, and their input can open with a byte-order mark,
  which `readStdin` drops. Nothing else here has been run on Windows.

## Contracts this depends on

- `POST /api/v1/ingest/batch`: one item per sitting. `id` gives replacement (`cursor:<chat id>`
  for the first sitting, `cursor:<chat id>:<n>` for the others), `timestamp` dates every memory
  drawn from the item, `label` names the chat (`Cursor · <project> · <branch>`, or `Cursor · no folder`), `metadata` carries
  `conversationId`, `project`, `cwd`, `gitBranch`, `sitting`, `turns` and `cursorVersion`.
- `POST /api/v1/recall`: **`identity` is required**. Recall stays off until `/past-connect` sets
  one.
- `audience` on ingest: a slug; absent means the whole project. `connect --audience` checks it with
  `GET /api/v1/audiences/{slug}`.
- Cursor gives every agent hook `conversation_id` (and `session_id`, the same value),
  `generation_id`, `cursor_version` and `workspace_roots`, and sets `CURSOR_PROJECT_DIR` and
  `CURSOR_VERSION` in its environment. `beforeSubmitPrompt` gives `prompt` and `composer_mode`,
  and takes `continue` and `additional_context`; `afterAgentResponse` gives `text`; `stop` gives
  `status`. `${CURSOR_PLUGIN_ROOT}` is expanded in `hooks.json` and `mcp.json`.

If any of these contracts changes, this file changes with it.

## Testing

By hand: there is no test suite. CI parses the script on Node 14, runs it on Node 16 and 22, and
checks that the manifests hold together.

For the hooks and the sender, point a throwaway home at a local stand-in for the API, so nothing
touches your real `~/.past` or a real project: `HOME=/tmp/past-test` with a `config.json` whose
`apiUrl` is `http://127.0.0.1:<port>`, a few lines of Node answering `POST /api/v1/recall` and
`POST /api/v1/ingest/batch` and logging what arrives, and each hook mode fed its stdin JSON by hand:

```bash
echo '{"conversation_id":"…","generation_id":"g1","prompt":"…","workspace_roots":["…"]}' \
  | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs prompt
```

A journal written with times in the past makes a chat due at once, so `send` can be watched
sending without waiting half an hour. `mcp` reads JSON-RPC lines on its input: `initialize`,
`tools/list` and `tools/call` with `past_recall`.

On a real Cursor, the plugin loads from a folder: copy `plugins/past` to
`~/.cursor/plugins/local/past` and reload the window. Cursor's **Hooks** output channel shows each
hook as it runs.

**After each Cursor release**, check that the recall still reaches the model: a local plugin whose
`beforeSubmitPrompt` returns `{"continue": true, "additional_context": "The probe word is …"}`,
then a prompt that asks the model for the probe word. If the model does not know it, the release
stopped delivering the field (rule 6).
