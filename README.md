# Pi Simple Web UI

> **A minimal check-in companion for an active [Pi](https://github.com/badlogic/pi-mono)
> session—not a complete or hardened web UI.**

Pi Simple Web UI gives you a narrow browser view of the session currently open
in Pi. It is useful for checking progress away from the terminal and for sending,
steering, or queueing a short message. Its objective is to stay small,
understandable, and close to Pi's own HTML-export presentation rather than grow
into a general remote-control dashboard.

## What it is—and is not

It renders the active branch as a single-column transcript: user and assistant
messages, thinking, Markdown, images, built-in tool calls, generic custom tools,
compactions, branch summaries, model changes, and custom messages. A sticky
composer can submit a prompt, steer a running turn, or queue a follow-up.

It is **not** a session browser, a replacement for Pi's TUI, a public web
service, a multi-user application, or a security-hardened administration
surface. It has no sidebar, session tree, provider/model controls, slash-command
dispatch, pagination, virtualization, or public-internet deployment story.

The browser presentation is visually descended from Pi's HTML exporter. That
lineage keeps transcripts familiar and intentionally constrains the design. The
active Pi theme is converted to CSS variables at runtime; automatic light/dark
theme pairs follow the browser color scheme.

## Usage

A git Pi package needs no build step. To run only this package, without loading
auto-discovered extensions, use exactly:

```sh
pi --no-extensions -e git:github.com/tudoroancea/pi-simple-web-ui
```

From a local checkout, the equivalent bare invocation is:

```sh
pi --no-extensions -e .
```

The extension starts in TUI and RPC modes, binds an ephemeral port on
`127.0.0.1`, and reports the local base URL. In the TUI, run:

```text
/copy-url
```

This copies a complete bootstrap link. Open it in a browser on the same machine.
In RPC mode, the command returns the link in a notification instead of using the
host clipboard.

### Optional Tailscale Serve

Remote serving is disabled by default. Opt in explicitly:

```sh
pi --no-extensions -e git:github.com/tudoroancea/pi-simple-web-ui --tailscale
```

With `--tailscale`, the extension starts the system `tailscale serve` command,
waits for its tailnet HTTPS origin, and stops that process with the session. Use:

```text
/copy-remote-url
```

to copy the remote bootstrap link. Without `--tailscale`, that command clearly
reports that the flag is required. If Tailscale is unavailable or disconnected,
the loopback UI continues to work.

Enabling this flag lets devices allowed by your tailnet policy reach a page that
can read the current transcript and send messages to Pi. Review Tailscale ACLs
and grants before enabling it; membership in a tailnet is not automatically the
right authorization boundary for an agent session.

## Composer and display controls

Enter inserts a newline. `Option+Enter` sends while idle or steers while Pi is
running. `Ctrl+Enter`/`Cmd+Enter` sends while idle or queues a follow-up while
running. Right-click or long-press the send button for explicit choices. Plain
`i` focuses the editor, and `@` opens file completion.

Plain `t`, `e`, `s`, `m`, and `p` toggle thinking, expanded tool output,
timestamps, model/thinking changes, and the effective system prompt.
`Cmd/Ctrl+K` opens the same persisted display settings in a command palette.

Slash commands are deliberately not accepted by the composer; see
[`docs/SLASH_COMMAND_DISPATCH.md`](docs/SLASH_COMMAND_DISPATCH.md).

## Architecture

- **No build:** Pi loads `src/index.ts` directly from the package-root
  `pi.extensions` manifest. Node serves `src/client/` from disk. The browser
  imports pinned Preact, HTM, and Marked ES modules from esm.sh.
- **HTTP and SSE:** a loopback Node HTTP server serves static files and sends a
  fresh full active-branch snapshot over Server-Sent Events on connection and
  relevant Pi updates. There is intentionally no reducer or custom incremental
  wire protocol.
- **Authentication:** each copied URL contains a random, single-use bootstrap
  code in its fragment. The browser exchanges it for a random, path-scoped,
  `HttpOnly; SameSite=Strict` cookie. Codes expire after two minutes and never
  reach the server in the initial HTTP request. Authenticated same-origin POSTs
  provide input and completion.
- **Composer:** input uses Pi's public message API with explicit immediate,
  steer, and follow-up delivery. Pending messages remain visible until Pi begins
  delivering them.
- **Lifecycle:** the local server, optional proxy process, timers, and streams are
  session-scoped and close on session shutdown.

## Tool rendering and extension points

Built-in `bash`, `read`, `write`, `edit`, and `ls` calls have compact renderers.
Every other tool uses a generic fallback that shows the tool name, JSON
arguments, text output, and output images. A newly installed custom Pi tool is
therefore usable without integration work.

For a richer custom renderer, add a narrowly matched branch in `ToolCall` in
`src/client/app.js` before the generic fallback. Keep the fallback intact, avoid
assuming extension-specific result shapes globally, and add a Playwright fixture
covering arguments, text, errors, partial output, and images as applicable.

Transport evolution should remain independent of rendering. If transcript size
becomes a problem, replace full snapshots with versioned deltas and reconnect
recovery rather than coupling renderers to event ordering. See
[`docs/SSE_TRAFFIC_ANALYSIS.md`](docs/SSE_TRAFFIC_ANALYSIS.md).

## Security model and stronger deployments

The random path, one-time bootstrap code, cookie, loopback default, and
same-origin checks reduce accidental access; they do not make this a hardened
remote administration service. The authenticated page exposes session content
(which may contain secrets) and can drive the composer. Local HTTP has no TLS,
the client has no comprehensive Content Security Policy, browser modules come
from a third-party CDN, and there are no users, roles, audit log, rate limits, or
independent confirmation for consequential prompts.

Do not expose the Node server directly to a LAN or the public internet. For a
stronger deployment, keep it on loopback and put a reviewed reverse proxy or SSH
forward in front of it with TLS, strong user authentication, authorization,
request and connection limits, and audit logging. Vendor browser dependencies,
apply a restrictive CSP, consider per-action authorization or read-only mode,
rotate/revoke session credentials, and threat-model the agent's host permissions
and the sensitivity of transcript data. Tailscale transport encryption and ACLs
are useful layers, not substitutes for that work.

## Limitations

- Only the currently active branch is shown; there is no session/tree navigation.
- Full snapshots can become expensive for long or rapidly updating sessions.
- Large transcripts are not virtualized.
- Code blocks have no syntax highlighting.
- The browser needs internet access to load the pinned CDN modules.
- Image attachment and model/provider controls are not implemented.
- The UI and authentication model are intended for personal, short-lived
  check-ins—not unattended or multi-user operation.

## Development

```sh
nub install
nubx playwright install chromium
nub run format:check
nub run lint
nub run typecheck
nub run test:e2e
```

The Playwright suite exercises the real authenticated HTTP/SSE server and the
production browser assets. The project is MIT licensed.
