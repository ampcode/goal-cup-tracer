# Amp harness facts

Connector id: `amp-cli` (proposed; no connector exists, see section 9) · Facts for version: `0.0.1791676897-g77c125` · Written by: Goal Cup team (first draft and Linux probe), corrected by the Amp team · Last checked: 2026-10-12
Public disclosure: partly. The Amp team left out internal server routes, server-side key handling and other details that a client does not need. Nothing here needs a credential, and no tokens, keys, account data or personal paths are included.
Status: facts only

How to read this sheet. Every fact names its source:

- "Amp source" means the Amp team read it in Amp's source code at the commit this release was built from.
- "measured (macOS)" means the Amp team ran the released npm binary on macOS 26.6, arm64, on 2026-10-12. Each run had an empty environment except `PATH=/usr/bin:/bin:/usr/sbin:/sbin`, a private `HOME` and `TMPDIR`. The execute runs also had an invented `AMP_API_KEY` and an `HTTPS_PROXY` that either refused connections or logged each `CONNECT` line and answered `403`, so no request left the machine.
- "measured (Linux)" means Goal Cup's own key-free probe on Linux x86_64 in an offline network namespace. The full record is in the [Amp readiness record](../qualification/m2-amp-readiness.md). The Goal Cup changes made so far are in [amp-core-prep](../research/amp-core-prep.md).
- "Amp docs" means [ampcode.com/docs](https://ampcode.com/docs) and the pages it links.

No authenticated run against Amp's servers was made for this sheet.

## 1. Run one prompt, headless

```
amp --execute=<goal> --stream-json
```

- `-x, --execute [message]` runs one prompt with no user interface. `--execute=<goal>` binds the whole goal, also a goal that starts with `-`. `-x <goal>` as two argv elements also works, but not when the goal starts with `-` (Amp source; measured (Linux)).
- Execute mode also turns on when stdout is not a terminal. `--stream-json` without `--execute` is refused with `--stream-json flag requires --execute mode` (Amp source).
- stdin is read even when the goal is in argv. Its text is added after the goal, up to 1 MiB, with a 30 s timeout. With stdin `/dev/null` it reads end of file at once (Amp source).
- `--stream-json` prints one JSON object per line on stdout, in the Claude Code stream JSON format (Amp source):
  - first one `{"type":"system","subtype":"init","cwd":…,"session_id":…,"tools":[…],"mcp_servers":[…],"agent_mode":…}`;
  - then `{"type":"user"|"assistant","message":{…},"parent_tool_use_id":null,"session_id":…}` lines;
  - last one `{"type":"result","subtype":"success","result":…,"is_error":false,…}` or `{"type":"result","subtype":"error_during_execution","error":…,"is_error":true,…}`.
  `--stream-json-thinking` adds thinking blocks. Subagent messages are not printed.
- The process runs one turn, then exits. The exit code is 0 every time the run reaches the agent loop, also after an `error_during_execution` result and after a handled SIGINT or SIGTERM. Read the `result` line, not the exit code. The exit code is 1 when the run fails before the agent loop (rejected key, no connection to the server, executor handshake failure, archived thread) and after an uncaught error (Amp source).
- Measured (macOS): with the invented key and no route out, exit 1 after about 1 s, empty stdout, and stderr `Error: Couldn't connect to the Amp server at https://ampcode.com/.`
- Tool approvals: with empty settings no tool asks for approval. When settings turn on the permissions rules, an "ask" verdict refuses that tool call and the agent continues; a guarded-file prompt exits with code 1. The hidden flag `--dangerously-allow-all` allows every tool call. If the server queues a tool approval, stream JSON mode refuses it and ends with an error result (Amp source).
- After the run, a new thread is archived on Amp's server, unless `--no-archive-after-execute` is given (Amp source).
- `-m, --mode <key>`: default `medium`. Built-in modes are `low`, `medium`, `high` and `ultra`; plugin modes are matched by key or label; other keys go to the server, which accepts or refuses them. The Claude Code external agent is `--mode external-agent:claude-code` (Amp source).
- With no key, stdout prints `No API key found. Starting login flow...` and a login URL, then waits up to 5 minutes for a browser callback or a pasted key (Amp source; measured (Linux)).
- `AMP_THREAD_ID`, when it holds a valid thread id, makes the new thread a child of that thread. Amp sets it inside its own sandboxes; a launcher should not pass it (Amp source).

## 2. Install and pin a version

- `npm install @ampcode/cli@0.0.1791676897-g77c125`. `@sourcegraph/amp` is the old package name. It is published at the same version as an alias whose only dependency is `@ampcode/cli`, and its README says the old name "still works for backwards compatibility for the time being". Use `@ampcode/cli` (npm metadata; Amp source).
- `@ampcode/cli` has `postinstall: node install.cjs`, which hard-links (or copies) the platform package's `amp` to `bin/amp` and makes `bin/amp.exe` a symlink to it. Platform packages: `@ampcode/cli-{darwin-arm64,darwin-x64,linux-arm64,linux-x64,win32-x64}`. There is no Windows arm64 package (npm metadata; Amp source).
- The executable is one native Bun 1.4.1 file: Mach-O on macOS, ELF on Linux, `amp.exe` on Windows. No `node` is needed at run time (measured (macOS): `Mach-O 64-bit executable arm64`).
- Where it lands (measured (macOS), same as measured (Linux)): `<prefix>/node_modules/@ampcode/cli/bin/amp` is a regular file with link count 2, the same inode as `@ampcode/cli-darwin-arm64/amp`. `<prefix>/node_modules/.bin/amp` links to `bin/amp.exe`, which links to `amp`. `program = "amp"` resolves to that regular file.
- Print the version: `amp --version` prints `0.0.1791676897-g77c125 (released 2026-10-11T00:01:37.000Z, …)`; `amp version` adds `npm package: @ampcode/cli` (measured (macOS)).
- Goal Cup records the build instead of pinning it (D53). For a connector, `version_args = ["--version"]` prints one line.
- Executable bytes for `0.0.1791676897-g77c125`:

  | Platform | Size (bytes) | SHA-256 | Source |
  |---|---|---|---|
  | darwin-arm64 `amp` | 109,666,576 | `107c8b2c312e97447bd20d9f7d678d477195bd330753de81749b547bc7477f44` | measured (macOS): tarball, platform package and installed `bin/amp` are identical |
  | darwin-x64 `amp` | 87,444,976 | `7e99fec7bfd68613cd73fe08f65abc4a9893ca5fff96e3a0d0d07905124dcc06` | npm tarball; not installed |
  | linux-x64 `amp` | 134,157,792 | `6c1b791d8e2c4661460b96276860c289d50ff0cacadaf1061ccb401295792036` | measured (Linux): tarball, platform package and installed `bin/amp` are identical |
  | linux-arm64 `amp` | unknown | unknown | not measured |
  | win32-x64 `amp.exe` | 104,117,072 | `5b45b230455440d092ea159a44095f40df8b4f2c431b51ce79a1698dba2b50f7` | npm tarball; installed bytes not measured |

- Other ways to install (Amp docs; Amp source): the install script `curl -fsSL https://ampcode.com/install.sh | bash` puts the binary in `$AMP_HOME/bin/amp`, default `$HOME/.amp/bin/amp`, and links it from `$HOME/.local/bin/amp`; on Windows `install.ps1` puts it in `%USERPROFILE%\.amp\bin\amp.exe`. Homebrew: `ampcode/tap/ampcode`.
- `amp update` finds how Amp was installed: a Homebrew install runs `brew upgrade ampcode/tap/ampcode`; a global npm, pnpm, yarn or bun install runs that package manager's global install of `@ampcode/cli`; an install-script binary downloads the new binary from `static.ampcode.com`, checks its SHA-256 and replaces `$AMP_HOME/bin/amp`. Execute mode never checks for updates and never updates itself (Amp source).
- Releases are frequent. On 2026-10-12 the npm dist-tag `latest` already pointed to `0.0.1791806555-g425e34`; the dist-tag `next` is an older line from 2026-05-14 (npm metadata).

## 3. Where it keeps things, per OS

| | macOS | Linux | Windows |
|---|---|---|---|
| Config dir | User settings `$HOME/.config/amp/settings.json`; `settings.jsonc` is read when `settings.json` does not exist (Amp source; measured (macOS): `$HOME/.config/amp` created, mode 0700, empty). Managed settings `/Library/Application Support/ampcode/managed-settings.json` (Amp source). Workspace settings `.amp/settings.json`; Goal Cup measured the workspace root as the nearest ancestor holding `.git`, else the cwd (measured (Linux)) | Same as macOS, and the user settings follow `XDG_CONFIG_HOME` (Amp source; measured (Linux)). Managed settings `/etc/ampcode/managed-settings.json` (Amp source; measured (Linux): read attempt) | User settings `%USERPROFILE%\.config\amp\settings.json`; follows `XDG_CONFIG_HOME` when it is set; `%APPDATA%` is not used. Managed settings `%ProgramData%\ampcode\managed-settings.json` (Amp source) |
| Session / transcript store | None on this machine. The thread (messages, tool calls, tool results, subagents) is stored on Amp's servers. After the run, `amp threads export <thread id>` prints it as JSON and `amp threads markdown <thread id>` as Markdown, both fetched from the server (Amp source). The only local per-thread file is a diagnostic log, not a transcript: `$HOME/.cache/amp/logs/threads/<thread id>.log` (section 4) | Same as macOS | Same as macOS; the diagnostic log is `%LOCALAPPDATA%\amp\logs\threads\<thread id>.log` (Amp source) |
| Logs / cache | Cache `$HOME/.cache/amp` (Amp source; measured (macOS)): `logs/cli.log` (JSON lines, rotated at 10 MiB, two files; measured mode 0644 in a 0700 directory), `logs/threads/<thread id>.log`, `bin/rg` (ripgrep, downloaded only when `rg` is not on `PATH`), `global-plugins/` and `global-skills/` (plugins and skills synced from the server). Data `$HOME/.local/share/amp` (Amp source; measured (macOS)): `device-id.json` and `.legacy-state-owner.json` (both mode 0600) on every run; `secrets.json`, `accounts.json`, `session.json`, `last-threads/` and `workspace-managed-settings-<hash>.json` when they apply (section 5). Bun extracts a native add-on (the keyring library) to `$TMPDIR/.bun-<uid>-<hash>.node`, mode 0600, and reuses it on later runs (measured (macOS)) | Same as macOS, with the cache following `XDG_CACHE_HOME` and the data following `XDG_DATA_HOME` (Amp source; measured (Linux)) | Cache and logs `%LOCALAPPDATA%\amp` (same files as macOS). Data `%USERPROFILE%\.local\share\amp` (Amp source). Where Bun extracts its add-on on Windows: unknown |
| Env vars that move these | `HOME` moves all of them (measured (macOS)). `XDG_CONFIG_HOME` moves settings. `XDG_CACHE_HOME` moves the cache and logs; an empty value makes the cache the relative path `amp/` under the cwd, so unset it rather than set it empty. `XDG_DATA_HOME` moves `secrets.json`, `accounts.json`, `device-id.json` and the managed-settings cache, but on macOS not `.legacy-state-owner.json`, `session.json` or `last-threads/`. `AMP_SETTINGS_FILE` or `--settings-file` names the settings file (absolute, or relative to the cwd). `AMP_LOG_FILE` or `--log-file` moves `cli.log`. `AMP_RIPGREP_PATH` names an `rg` binary. `TMPDIR` moves the Bun add-on. `AMP_HOME` moves only the install-script binary (Amp source) | Same as macOS, except that `XDG_DATA_HOME` moves every data file (Amp source; measured (Linux): settings and log follow XDG; `AMP_HOME` did not move them) | `USERPROFILE` (home), `LOCALAPPDATA` (cache and logs); `XDG_CONFIG_HOME` and `XDG_DATA_HOME` as on macOS; `XDG_CACHE_HOME` is ignored (Amp source) |

Guidance files: Amp reads one of `AGENTS.md`, `Agents.md`, `agents.md`, `AGENT.md`, `Agent.md`, `agent.md` or `CLAUDE.md` (in that order) from the cwd and every ancestor, stopping before `/`, `/Users` or `/home`. It also reads them from `$HOME/.config/amp/`, `$HOME/.config/` and the managed-settings directory. `AMP_IGNORE_GUIDANCE_FILES` takes `:`-separated glob patterns matched against absolute paths; `*` turns all guidance files off (Amp source).

Skills: Amp loads skills from `$HOME/.config/agents/skills`, `$HOME/.agents/skills`, `$HOME/.config/amp/skills`, `.agents/skills` and `.claude/skills` in the cwd and its ancestors (stopping at `$HOME`), `$HOME/.claude/skills`, `$HOME/.claude/plugins/cache`, and the `amp.skills.path` setting. `amp.skills.disableClaudeCodeSkills` turns off the Claude Code directories (Amp source; measured (Linux) for the scan).

## 4. Session format and identity

- **No local own-session file.** The thread lives in a thread actor on Amp's servers. The CLI holds only a short tail of it in memory during the run (Amp source). There is nothing on disk for Goal Cup to read as a native store.
- **The diagnostic thread log is not a transcript.** `logs/threads/<thread id>.log` in the cache directory is written in execute mode. It is append-only JSON lines: operational log records (for example connection and transport state) that carry the thread id, plus process id and trace ids. It does not hold the conversation, and its record shape is not a stable format. A later execute or interactive run deletes thread logs older than 7 days (Amp source).
- **Id.** `T-` followed by a UUIDv7, for example `T-019a…`. Amp's server mints it when the CLI creates the thread, after authentication and plugin and skill sync. A caller cannot choose the id of a new thread: a requested id must already exist (Amp source). Each thread has its own id. The thread page is `https://ampcode.com/threads/<thread id>`.
- **Where the id first appears locally.** In `--stream-json` mode, as `session_id` on the first stdout line, `{"type":"system","subtype":"init",…}`. Every later line carries the same `session_id`. The CLI prints the init line when the server starts the first inference turn: after the server has built the prompt and tool list, just before it calls the model. The CLI holds back any user or assistant line until the init line is printed, so the init line always comes before any model text on stdout (Amp source). The model request itself is made by Amp's server (section 6), not by the local process.
- **When no init line appears.** A run that fails before the agent loop (rejected key, no connection) prints no JSON at all, only an error on stderr (Amp source; measured (macOS)). A run that ends early with an error result still prints the init line first.
- **Other places the id appears.** The diagnostic log's file name, the files under `last-threads/` in the data directory (the last thread id for a workspace), and the thread URL (Amp source).

## 5. Auth

- **API key.** `AMP_API_KEY` holds an Amp access token. A non-empty value takes precedence over stored credentials, and a key from the environment is never written to `secrets.json` or a keychain. An OAuth token (JWT shape) in `AMP_API_KEY` is refused at startup (Amp source).
- **Login.** `amp login` signs in through the browser or a pasted key and stores the key (Amp source).
- **Token grammar.** Amp keys start with `sgamp_` and use only `[A-Za-z0-9_]`. A user key is `sgamp_user_`, 26 upper-case letters and digits, `_`, and 64 lower-case hex digits: 102 characters. A service-account key is `sgamp_sa_` and 64 hex digits: 73 characters (Amp source). This fits Goal Cup's assumption of 20 to 256 bytes of `[A-Za-z0-9_-]`.
- **Where credentials are stored.** On every OS the default is a plain file, `secrets.json` in the data directory (`$HOME/.local/share/amp/secrets.json` on macOS and Linux, `%USERPROFILE%\.local\share\amp\secrets.json` on Windows), mode 0600, one entry per secret and server URL. The OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service, through the `@napi-rs/keyring` library) is used only when the setting `experimental.cli.nativeSecretsStorage.enabled` is true, which is off by default (Amp source; measured (macOS): the log says `using file-based secrets storage`). `accounts.json` is written by login and account switching, not in a run with `AMP_API_KEY` (Amp source).
- **A rejected or revoked key.** stderr prints `Error: Invalid or missing API key. Run 'amp login' to authenticate.` (HTTP 401) or `Error: Access token expired. Run 'amp login' to refresh.` (HTTP 403); exit code 1; nothing on stdout. These messages do not contain the key. Execute mode does not start a login when `AMP_API_KEY` is set (Amp source). The CLI does not redact its own log files, and the error object is written to `cli.log` (Amp source). Whether a server error body can contain the key: unknown. Measured (macOS): after a failed run, no part of the invented key was found in any file the run wrote.
- **Claude Code mode** (Amp docs; Amp source): a Claude subscription is used through the local `claude` binary and its own login. Amp removes `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX` and `CLAUDE_CODE_USE_FOUNDRY` from the child's environment and needs Claude Code 2.1.276 or later.
- **Files derived from the key** (Amp source):
  - `.legacy-state-owner.json` holds `identityKey`: the first 16 hex digits of the SHA-256 of the normalized server URL, a NUL byte and the user id. For an `sgamp_` key the user id is the part between `sgamp_` and the last `_`, so `identityKey` does not depend on the key's secret part. For any other credential the user id is derived from the SHA-256 of the credential. It records which account owns the local state directories that are not namespaced by account. No setting stops it from being written in a run with `AMP_API_KEY`.
  - `workspace-managed-settings-<hash>.json` (only when the workspace has server-managed settings) holds the full, unsalted SHA-256 of the key as `credentialHash`, so a cached copy is used only with the same credential.

## 6. Network

- **One server.** `https://ampcode.com/` (default of `AMP_URL` and the `amp.url` setting). In a normal execute run every request goes to this host (Amp source; measured (macOS): the logging proxy saw only `CONNECT ampcode.com:443`, twice, before the run failed):
  - HTTPS requests with JSON bodies: authentication and account status, plugin and skill sync, thread creation, telemetry and the archive call after the run. Each carries `AMP_API_KEY` in an `Authorization: Bearer` header.
  - One WebSocket under `https://ampcode.com/actors/`, which carries the thread: messages, tool calls and tool results. Frames are JSON-RPC 2.0 text frames, plus binary frames for some executor relays. The `rivet_encoding` subprotocol label is present, but the frames are not CBOR. A short-lived per-thread token rides in the `Sec-WebSocket-Protocol` header. There is no SSE. The HTTP version: unknown.
- **No model or provider host from the CLI.** Inference runs on Amp's servers, in the thread actor. That includes bring-your-own-key: `amp config model-providers` stores the provider connection on Amp's server, and the server calls the provider with it (Amp source). The local process never sends a model request. A local proxy sees the thread protocol to `ampcode.com`, not the model traffic. The one exception is Claude Code mode, where the local `claude` child talks to Anthropic itself; its hosts are Claude Code's, not Amp's.
- **Other hosts.**
  - `storage.googleapis.com` (bucket `amp-public-assets-prod-0`), only to download ripgrep when `rg` is not on `PATH` and `AMP_RIPGREP_PATH` is unset; the download is checked against a SHA-256 and, on macOS, Amp's code signature (Amp source).
  - Assets that a tool uploads may go to a signed storage URL that the server returns (Amp source).
  - MCP servers configured in settings, and plugins (local or synced from the server), can contact any host.
- **Update check.** Not in execute mode. Elsewhere `https://static.ampcode.com/cli/cli-version.txt` (or the npm registry for an npm install). `AMP_SKIP_UPDATE_CHECK=1` or `amp.updates.mode: "disabled"` turns it off (Amp source).
- **Telemetry.** `POST https://ampcode.com/api/telemetry` with feature-usage events, for example MCP connections, managed-settings use and use of `--dangerously-allow-all`. This version has no client setting to turn it off. There is no third-party analytics, and no OpenTelemetry exporter (Amp source).
- **`HTTPS_PROXY`.** The HTTPS requests use Bun's `fetch`, which honours `HTTPS_PROXY`, `https_proxy`, `HTTP_PROXY`, `http_proxy` and `NO_PROXY` (measured: the logging proxy above). The WebSocket uses a bundled undici WebSocket with an environment proxy agent, which reads the same variables (lower case first) once at startup (Amp source; tested by the Amp team with a bundle built the same way as the release, not in an authenticated run). `AMP_USE_NATIVE_WEBSOCKET=1` switches to Bun's own WebSocket, which ignores the proxy; do not set it. The `amp.proxy` setting is copied into `HTTP_PROXY`/`HTTPS_PROXY` only when they are unset, and too late to reach the WebSocket. Uploads to a signed storage URL may not use `HTTPS_PROXY` (Amp source; not tested).
- **Custom CA.** `NODE_EXTRA_CA_CERTS` adds to Bun's bundled Mozilla roots, for both the HTTPS requests and the WebSocket. Without it, a certificate from an unknown CA fails with `UNABLE_TO_VERIFY_LEAF_SIGNATURE`; there is no fallback that accepts it. The OS trust store is added only with `NODE_USE_SYSTEM_CA=1` or `--use-system-ca`. On Windows, Amp also loads the Windows root certificates. Amp has no CA setting of its own (Amp source; tested by the Amp team with a private CA and a bundle built the same way as the release, not in an authenticated run).
- **Base-URL and endpoint overrides (Goal Cup never sets these):** `AMP_URL` and the `amp.url` setting (the server), `RIVET_PUBLIC_ENDPOINT` (the thread WebSocket), `AMP_STORAGE_BASE` (update downloads) and `AMP_PORTAL_DOMAIN` (links only) (Amp source). Goal Cup's name rules refuse `AMP_URL` (`_URL`), `AMP_STORAGE_BASE` (`_BASE`) and `RIVET_PUBLIC_ENDPOINT` (`ENDPOINT`); no rule matches `AMP_PORTAL_DOMAIN`.

## 7. Process model

- **One process is the executor.** With `--executor local` (the default) this process is the thread's executor: tool calls run here, in the cwd. The agent loop, inference, transcript and thread state are on Amp's server. With `--executor orb` or `runner:<id>` the process only watches a thread that runs elsewhere (Amp source).
- **Children** (Amp source; measured (Linux): offline only `git rev-parse --show-toplevel`):
  - `git`, for workspace status and snapshots.
  - Shell tool commands, through `/bin/bash`. They inherit the full environment, including `AMP_API_KEY`, `HTTPS_PROXY` and `NODE_EXTRA_CA_CERTS`. Their output goes through Amp's secret redaction before it reaches the thread.
  - `rg` (ripgrep).
  - MCP servers that run over stdio, with `AMP_API_KEY` removed from their environment. Untrusted workspace MCP servers are not started.
  - Plugins: Amp runs its own binary again as Bun (`BUN_BE_BUN=1`).
  - `claude`, in Claude Code mode only. The server asks for the executable `claude`; the CLI searches `$HOME/.local/bin`, `$HOME/bin` and then `PATH`. There is no setting for its path. With Goal Cup's `PATH` and a private home, `claude` is found only in the private home's `.local/bin` or `bin`. The child inherits the full environment, including `HTTPS_PROXY` and `NODE_EXTRA_CA_CERTS`, except the variables listed in section 5. It runs with `permissionMode: bypassPermissions` and does not use `persistSession` or `sessionStore`: the CLI streams its events to Amp's server, which adds them to the Amp thread.
- **Argv pre-scan.** Before its option parser runs, Amp scans the raw argv for `--log-file <value>` and `--log-level <value>` (as two elements), `--headless` or `--headless=…`, `--no-tui`, `--color`, `--no-color`, the token `-ox` (rewritten to `--orb-execute -x` before a `--`), and whole elements that equal a subcommand name such as `login` or `update`. A goal passed as the one element `--execute=<goal>` is not matched by any of these (Amp source).
- **Subagents** run on Amp's server; their tool calls run in this process. Stream JSON mode does not print their messages (Amp source).
- **SIGINT and SIGTERM** (Amp source):
  - Before the turn starts, there is no handler, so the process ends at once.
  - During the turn, the CLI asks the server to cancel the agent loop without waiting for an answer, prints `{"type":"result","subtype":"error_during_execution","is_error":true,"error":"User cancelled (SIGINT/SIGTERM)",…}`, archives the thread and exits with code 0.
  - After SIGKILL the server is not told to stop. An inference in progress can finish, and tool calls wait for an executor that does not come back. Whether the server ends such a turn after some time: unknown.
- **Exit codes.** 0 and 1 as in section 1; a closed stdout pipe also gives 0 (Amp source).

## 8. Things Goal Cup cannot isolate

- **Machine-wide managed settings** at the managed-settings paths in section 3, read outside any private home. Guidance files are also read from the managed-settings directory (Amp source; measured (Linux) for the Linux path).
- **Server-side state.** The thread itself; the system prompt and tool list, which Amp changes on its servers without a new CLI version; workspace managed settings; global plugins and skills, which are downloaded and then run locally; bring-your-own-key provider connections; custom agents and modes; account feature flags (Amp source).
- **The model traffic.** It leaves from Amp's servers, so a local proxy cannot see it (section 6).
- **Guidance files and skills** from ancestors of the cwd and from the Claude Code directories (section 3).
- **The Bun add-on** reused from `$TMPDIR`: a file planted there before the run would be loaded (measured (Linux): reused after only a `stat`).
- **Shell tool commands** inherit `AMP_API_KEY`; a tool could read it, and only Amp's redaction keeps it out of the thread.
- **The `claude` child** in Claude Code mode: it is not pinned, and it has its own login and its own state in the (private) `$HOME/.claude`.

## 9. Quirks and open questions

### Quirks

- Execute mode starts an interactive login when no key is found, and prints the login text on stdout before any JSON.
- The exit code is 0 after an error result; only the `result` line tells success from failure.
- A new thread is archived on the server after the run.
- An empty `XDG_CACHE_HOME` puts the cache, logs and the ripgrep download in `amp/` under the cwd. Unset XDG variables are fine.
- An error that contains "403" is reported as `Error: Access token expired. Run 'amp login' to refresh.`, also when a proxy answers `CONNECT` with `403` (measured (macOS)). An error that contains "401" is reported as an invalid key (Amp source).
- If the directory `TMPDIR` names does not exist, Amp exits at startup with `ERR_DLOPEN_FAILED` for its keyring add-on (measured (macOS)).
- An inherited `AMP_THREAD_ID` makes the new thread a child of another thread.

### Can Goal Cup add an Amp connector today?

No. The identity fits, but the native store does not:

- **Identity fits (decision table row 4, `stdout_json`).** The init line has `subtype` `init` and the id in `session_id`, and it comes before any model text. Rows 1 to 3 fit with `AMP_API_KEY` (D49, D50).
- **No local own-session store (row 0, "No local session file", and row 9).** The thread lives only on Amp's servers. The diagnostic thread log has the right file name and is append-only, but it holds operational records, not the session, so a trace built on it would claim a transcript it does not contain.
- **No local model traffic.** The proxy would record the thread protocol to `ampcode.com`, not model requests (section 6). Goal Cup should decide whether that WebSocket counts as the run's model traffic. Every HTTPS request to `ampcode.com` also carries the key in an `Authorization` header.

### What Amp could add to make capture possible

These are options for discussion, not commitments:

1. **A local transcript file behind a flag or setting.** During an execute run, Amp appends each thread event to one JSON-lines file per thread, for example `<data dir>/threads/<thread id>.jsonl`, in the stream JSON format that stdout already uses, and includes subagent messages with their `parent_tool_use_id`. This gives Goal Cup one append-only native row (rows 8 and 9) under the private home, bound by the `stdout_json` init line it already prints.
2. **The init line before the first turn.** Amp prints the init line as soon as the server returns the thread id, before it sends the prompt, so the id is bound before any thread traffic.
3. **A non-zero exit code after an error result,** so a launcher can tell success from failure without reading stdout.
4. **A treatment of an empty `XDG_CACHE_HOME` as unset,** and a clear error when `TMPDIR` does not exist.

Two options need a Goal Cup core change instead: accept the stream JSON on stdout as the native record, or run `amp threads export <thread id>` after the run and seal its output as a snapshot.

### Open questions for Amp, answered

1. **Where is a thread kept?** On Amp's servers only. There is no local own-session file and no supported local record today; `amp threads export` reads it from the server after the run. See sections 3 and 4, and the options above.
2. **Identity carrier?** Yes: the stdout `system`/`init` line with `session_id` exists in `--stream-json` execute mode. It is printed when the server starts the first inference turn, before any model text on stdout, and the model request is the server's. The id is `T-<UUIDv7>`, minted by the server, stable for the thread and unique.
3. **Transport?** HTTPS JSON requests and one WebSocket with JSON-RPC text frames (not CBOR, no SSE), all to `ampcode.com`. The requests and the WebSocket honour `HTTPS_PROXY` in the release build; uploads to a signed storage URL may not. HTTP version: unknown.
4. **`NODE_EXTRA_CA_CERTS`?** Yes, for the requests and the WebSocket, and a certificate from an unknown CA fails with no fallback. Tested with a bundle built the same way as the release, not in an authenticated run.
5. **Token grammar?** `sgamp_` and then `[A-Za-z0-9_]`: 102 characters for user keys, 73 for service-account keys. The rejection messages do not echo the key; see section 5 for logs.
6. **`identityKey`?** The first 16 hex digits of SHA-256(normalized server URL, NUL, user id); for an `sgamp_` key the user id comes from the key's public part. No setting stops it in a run with `AMP_API_KEY`.
7. **Keychain, `accounts.json`, `secrets.json`?** By default `secrets.json` in the data directory on every OS; the OS keychain only with `experimental.cli.nativeSecretsStorage.enabled`. A key in `AMP_API_KEY` is never stored. `accounts.json` is written only by login and account switching.
8. **macOS and Windows locations?** See section 3. Full macOS managed-settings path: `/Library/Application Support/ampcode/managed-settings.json`.
9. **Claude Code in execute mode?** `--mode external-agent:claude-code`. Creating such a thread needs no experimental feature, but a workspace policy can block it. `amp config external-agents` manages its settings, and its help says that command needs the "External agents" experimental feature.
10. **Claude Code child?** Found in `$HOME/.local/bin`, `$HOME/bin`, then `PATH`. It inherits `HTTPS_PROXY` and `NODE_EXTRA_CA_CERTS`. No `persistSession` or `sessionStore`: Amp streams its events to Amp's server, which keeps them as the Amp thread.
11. **Update check?** It does not run in execute mode. `AMP_SKIP_UPDATE_CHECK=1` suppresses it elsewhere. On an npm install, `amp update` runs the package manager's global install of `@ampcode/cli`.
12. **`--executor local` and `AMP_HOME`?** Local keeps tool execution on this machine; the thread, inference and transcript stay on Amp's server. `AMP_HOME` only sets the install-script binary location (default `$HOME/.amp`); it moves no settings, data, cache or logs.
13. **Guidance files in ancestors?** Yes, up to but not including `/`, `/Users` or `/home`. See section 3.
14. **Bring-your-own-key?** Through Amp's server: the server calls the provider with the stored key, not the CLI.
15. **Installed bytes?** macOS arm64: the installed `bin/amp` is the same as the tarball (measured). macOS x64 and Windows hashes are from the tarballs; installed bytes there are not measured, but `install.cjs` links or copies the file without changing it (Amp source).
