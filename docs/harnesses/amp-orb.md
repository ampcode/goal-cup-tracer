# Amp (orb threads) harness facts

Connector id: `amp-orb-cli` (proposed; no connector exists, see section 9) · Facts for version: `0.0.1791676897-g77c125` · Written by: the Amp team · Last checked: 2026-10-15
Public disclosure: partly. The Amp team left out internal server routes, server-side credential handling and other details that a client does not need. No tokens, keys, account data or personal paths are included.
Status: facts only

This sheet is a variant of [amp.md](amp.md). That sheet covers Amp with the default local executor, where tool calls run on the participant's Mac. This sheet covers an **orb thread**: the agent's tools run in an orb, a fresh cloud machine that Amp starts for the thread, and the Mac only starts the thread and watches it. Facts that are the same in both modes (install, paths on the Mac, auth, proxy and CA behaviour) are in amp.md and are not repeated here. Section 9 shows how Goal Cup tracing could work for orb threads and what is missing today.

Source words: "Amp source" means the Amp team read it in Amp's source code at the commit this release was built from. "Amp docs" means [ampcode.com/docs/orbs](https://ampcode.com/docs/orbs) and [Spawning Orbs](https://ampcode.com/docs/cli/spawning-orbs). No orb run was measured for this sheet.

## 1. Run one prompt, headless

```
amp --execute=<goal> --executor orb --stream-json --no-archive-after-execute --project <project>
```

- `amp -ox <goal>` and `amp --orb-execute -x <goal>` are the same as `--execute=<goal> --executor orb`. `-ox` is rewritten only before a `--` (Amp source; Amp docs).
- The CLI creates a new thread on Amp's server with an orb as its executor and with the goal as the first message. The server starts the orb and the agent begins. The local process then joins the thread as an observer only: it runs no tools (Amp source).
- `--stream-json` keeps the process attached until the turn ends and prints the thread's events on stdout in the same format as local execute mode (amp.md, section 1). Without `--stream-json`, the CLI prints the thread URL and exits at once, before the turn runs (Amp docs).
- The process exits after the first assistant message that ends the turn, when the agent is idle. For Goal Cup that is the end of the run: the agent has finished its one prompt (Amp source).
- `--no-archive-after-execute` is required for Goal Cup. Without it, the CLI archives the new thread when the stream ends, also after a signal. Archiving cancels the agent's turn and pauses the orb (Amp source; Amp docs).
- `--project <project>` names the Amp project (`namespace/name`, `owner/repo` or a repository URL) whose repository the orb clones. Without it, the CLI matches the cwd's git remotes against the user's projects; with no match it prints a warning to stderr and starts an orb with no repository (Amp source; Amp docs).
- Nothing from the Mac's working tree goes to the orb. The orb clones the project's repository at its base branch; local commits that are not pushed, and uncommitted changes, are not in the orb. Only files given with `--attach <path>` and the text on stdin are sent (Amp source).
- Other accepted options: `--orb-size`, `-m/--mode`, `--title`, `--label`, `--attach`, `--stream-json-thinking`. `--stream-json-input` and `--output-tool` are refused with an orb (Amp source).
- `AMP_API_KEY` must hold an access token that starts with `sgamp_`; the short-lived session token from `amp login` is refused (Amp docs).
- Exit codes are as in amp.md: 0 when the run reaches the agent loop, also after an error result; 1 for failures before it (Amp source).

## 2. Install and pin a version

Same as amp.md, section 2: the Mac needs the `amp` CLI from `curl -fsSL https://ampcode.com/install.sh | AMP_VERSION=<version> bash`.

The orb runs its own copy of Amp, `/home/user/.amp/bin/amp`, as a headless executor service (Amp source). Amp chooses that version, not the participant, and the Mac's binary hash says nothing about it. Which version an orb ran is not shown to the CLI today: unknown.

## 3. Where it keeps things, per OS

The Mac columns are the same as amp.md, section 3; in orb mode the Mac writes only `cli.log`, an observer-side `logs/threads/<thread id>.log`, a `last-threads/` entry and the caches (Amp source). The orb is a Linux machine on Amp's side:

| | macOS (the participant's Mac) | Linux (the orb) | Windows |
|---|---|---|---|
| Config dir | As amp.md. Local settings, skills, plugins and MCP servers are not sent to the orb (Amp source) | `HOME` is `/home/user`. The orb gets settings, skills and plugins from the project's repository (`.amp/`, `.agents/`), and the skills and plugins that Amp's server syncs for the workspace and the user (Amp source) | As amp.md; nothing runs on Windows in orb mode |
| Session / transcript store | None | None. The thread is stored on Amp's servers, as in local mode. After the run `amp threads export <thread id>` prints it as JSON (Amp source) | None |
| Logs / cache | As amp.md, observer records only | `/home/user/.cache/amp/logs/`: `headless.log`, `setup.log`, `cli.log` and `threads/<thread id>.log` (diagnostic, not a transcript) (Amp source) | As amp.md |
| Workspace | Not used | `/home/user/workspace/repo` holds the project's repository; more repositories go in `/home/user/workspace/repos/<name>` (Amp source) | Not used |
| Env vars that move these | As amp.md | Set by Amp, not by the participant | As amp.md |

The product is in the orb's workspace. It can be reviewed in the thread's Changes view, browsed in the Files pane, mirrored to a local checkout with `amp sync <thread>`, or pushed to a repository by the agent (Amp docs).

## 4. Session format and identity

- **Id.** `T-<UUIDv7>`, minted by Amp's server when the CLI creates the thread, as in local mode (Amp source).
- **Where the id first appears locally.** In the stdout `{"type":"system","subtype":"init",…,"session_id":…}` line. In orb mode it is printed when the first inference turn starts, which is after the orb has started and run its setup; that can take minutes (Amp source). User and assistant lines are held back until the init line is printed.
- **The init line describes the Mac, not the orb.** Its `cwd`, `tools` and `mcp_servers` come from the local process (Amp source). Use only `session_id` from it.
- **No local own-session file**, on the Mac or in the orb. The transcript lives on Amp's servers (Amp source).
- **What the server keeps.** Every message with its time; user messages record who sent them (user id and device) and from which client; assistant messages record the model and token usage (Amp source). `amp threads export <thread id>` returns the top-level messages, tool calls and tool results with that metadata. Subagent transcripts are not in it. `amp threads raw`, which includes subagents, is limited to Amp staff (Amp source).
- **Reconnects.** If the Mac sleeps or loses the network, the observer reconnects and resumes from the last thread version it saw. The orb keeps working in the meantime (Amp source).

## 5. Auth

- **On the Mac.** `AMP_API_KEY`, as in amp.md, section 5.
- **In the orb.** Amp gives the orb Amp credentials for the thread's owner, so the agent can call Amp as the participant, for example start more threads. The orb also gets a GitHub credential helper for the project, and the workspace, project and personal environment variables and secrets that the participant configured in Amp (Amp source). None of these pass through the Mac.
- **Claude Code mode.** With `--mode external-agent:claude-code`, the `claude` process runs in the orb with the participant's Claude login, which Amp passes to the orb (Amp source).
- **Workload identity.** Inside an orb, `amp orb id-token --audience <audience>` mints a signed token (RS256) that names the user, thread, workspace and project. Anyone can verify it against Amp's published keys (Amp source). It proves that code ran in a given thread's orb; it does not attest the transcript.

## 6. Network

- **From the Mac.** Only `ampcode.com`: the HTTPS JSON calls that create the thread and resolve the project, and the observer WebSocket under `https://ampcode.com/actors/` (Amp source). Proxy and CA behaviour are as in amp.md, section 6. A local proxy sees the thread's events as JSON-RPC text frames, not model requests and not the orb's traffic.
- **Model traffic.** Inference runs on Amp's servers, in the thread actor, as in local mode. In Claude Code mode the `claude` process in the orb calls Anthropic from the orb (Amp source).
- **From the orb.** Tool calls run in the orb and can reach the internet: package registries, GitHub, any host the agent chooses (Amp docs: the agent "can install dependencies, run your app, spawn browsers"). None of it passes through the Mac.

## 7. Process model

- **On the Mac.** One observer process. It runs no tools and starts no children for the agent (Amp source).
- **In the orb.** A headless Amp executor service runs the tool calls: shell commands, git, ripgrep, MCP servers, plugins, browsers, and `claude` in Claude Code mode (Amp source).
- **SIGINT and SIGTERM on the Mac** detach. The CLI prints `{"type":"result","subtype":"error_during_execution","is_error":true,"error":"Detached on SIGINT/SIGTERM; the thread continues on its remote executor",…}` and does not cancel the turn. With `--no-archive-after-execute` the agent keeps working; without it the CLI then archives the thread, which stops it (Amp source).
- **Stopping the agent.** `amp threads archive <thread id>` cancels the turn and pauses the orb. There is no separate cancel command (Amp source).
- **Lifetime.** An orb pauses when the agent has done no work for 5 minutes and nobody has interacted with it for 20 minutes. These are idle timeouts, not a run limit (Amp docs). A low credit balance also stops the orb from being kept awake (Amp source). A hard maximum run time: unknown.
- **Start pacing.** Each user can start 20 orbs in a burst; after that Amp starts one new orb every five minutes, and later starts wait (Amp docs).

## 8. Things Goal Cup cannot isolate

- **People with access to the thread.** The owner, and contributors while multiplayer is on, can act on the orb through channels that are **not** recorded in the thread (Amp source):
  - the Terminal tab, a shared tmux session in the orb (input is passed to the orb with no thread event);
  - the Desktop tab;
  - file uploads and deletes through the Files pane, and pasting a file into the terminal;
  - git commands and service starts from the web app;
  - portal pages served from the orb.
  Messages are recorded: a message from the web app, the native apps, `amp threads continue` or a portal review comment is stored in the thread with its sender and device. Editing an earlier message cuts the thread at that message and reruns it, so the server's current transcript no longer holds the old tail. Multiplayer is off unless the owner turns it on.
- **The orb's setup.** The project's pre-setup script, `.agents/setup` (which can come from a project snapshot up to 72 hours old) and `.agents/resume` run before and around the agent (Amp docs).
- **Credentials and secrets** that the participant configured in Amp reach the orb (section 5).
- **Server-side state**, as in amp.md, section 8, plus the orb's Amp version.
- **Everything in the orb.** Goal Cup's private home, `PATH` and process checks apply only to the Mac's observer process.

## 9. Quirks and open questions

### Quirks

- Without `--no-archive-after-execute`, the end of the stream or a signal archives the thread and stops the agent.
- The init line comes only after the orb has started and set up, and its `cwd`, `tools` and `mcp_servers` describe the Mac.
- An orb with no matching project starts with no repository; the warning goes to stderr.
- Local changes that are not pushed are not in the orb.

### Can Goal Cup trace an orb thread today?

No, not with the schema-1 connector model. Row 0 ("No local session file") stops it, as for local Amp, and more of the run happens off the Mac:

```
 Participant's Mac                      Amp's servers                 Orb (Linux)
┌────────────────────────┐            ┌──────────────────────┐      ┌──────────────────────┐
│ goalcup start          │            │ thread actor         │      │ headless Amp executor│
│  └ amp -x … --executor │  JSON-RPC  │  transcript          │ tool │  shell, git, files,  │
│     orb --stream-json  │◀──────────▶│  inference ─▶ models │◀────▶│  browsers, claude    │
│    (observer only)     │  over WSS  │                      │ calls│                      │
│  recording proxy sees  │            │                      │      │ Terminal, Desktop,   │
│  this WebSocket only   │            │                      │      │ Files pane ◀─ people │
└────────────────────────┘            └──────────────────────┘      └──────────────────────┘
```

What a local tracer can record today, and what it cannot:

| Goal Cup wants | Orb thread today |
|---|---|
| The harness's own session file | None locally. The server holds the transcript; `amp threads export` gives top-level messages after the run, not subagents |
| A session id before model text | Yes: `session_id` on the stdout init line |
| The model traffic | Not visible: inference runs on Amp's servers. The local proxy can record the observer WebSocket, which carries the thread's events |
| A private home and controlled env for the agent | Only for the observer on the Mac. The agent's tools run in the orb, with Amp's environment and the participant's Amp secrets |
| Proof that no human helped | Messages are recorded with sender and device. Terminal, Desktop, Files pane and git actions in the orb are not |
| Finish and kill | The CLI stops at the end of the first turn; `amp threads archive` stops the agent; a signal only detaches |

### How tracing could work for orb threads

A possible design, for discussion. It needs both Amp changes and Goal Cup core changes; none of it is a commitment.

1. `goalcup start` launches `amp --execute=<goal> --executor orb --stream-json --no-archive-after-execute --project <project>` on the Mac, binds `session_id` from the init line, and records stdout and the observer WebSocket as the live record. Laptop sleep is safe: the orb keeps working and the observer resumes.
2. When the result line arrives, or at the 168-hour limit (then `amp threads archive <thread id>`), the tracer fetches the server's record of the thread and seals it with the live record.
3. `goalcup-verify` checks the server record's signature against Amp's published keys and compares it with the live stdout record.

What Amp could add for this:

1. **A signed thread export.** A server-signed record of the whole thread (all messages including subagents, tool calls and results, model and usage per inference, sender and device per user message, and message edits), verifiable offline against Amp's published keys, like the workload identity tokens.
2. **Records of the side channels.** Thread events (or audit events) for Terminal sessions, Desktop sessions, Files pane writes and deletes, and git commands from the web app, with the user and device.
3. **An unattended mode for a thread.** A thread setting, fixed at creation, that refuses new messages, message edits, multiplayer, Terminal, Desktop and Files pane writes after the first prompt, and is shown in the signed export.
4. **The orb's Amp version and setup in the record.** The executor's version and the hashes of the setup scripts and snapshot that ran.
5. **A local transcript file behind a flag**, as in amp.md, section 9, so the stdout stream lands in an append-only file the tracer can record.

### Open questions for Amp

1. Is there a hard maximum run time for an orb, separate from the idle timeouts?
2. Which Amp version runs in the orb, and can a client read it?
3. Does `amp threads export` work for an archived thread, and does it include message edits?
4. Can a workspace or project turn off the Terminal, Desktop and Files pane for an orb thread today?
