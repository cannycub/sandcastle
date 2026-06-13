# Plain-text passthrough and a non-resumable Kiro provider

## Context

This fork adds [Kiro CLI](https://kiro.dev) (`kiro-cli`) as a built-in **agent
provider**, driven in its **headless** mode (`kiro-cli chat --no-interactive`).
Evaluating Kiro against the questionnaire in
[`adding-an-agent-provider.md`](../agents/adding-an-agent-provider.md) surfaces
two collisions with what that guide and our ADRs treat as hard requirements:

1. **No structured JSON stream.** The guide lists "structured (JSON) stream
   events" as required — `parseStreamLine` is documented to consume
   line-delimited JSON, "without it, tool calls and partial text cannot render."
   Kiro headless emits **plain text / markdown** to stdout. Its only structured
   output (`-f json`) applies to `--list-models` / `--list-sessions`, never to
   the chat response.

2. **Database-backed, non-resumable sessions.** Resume is a hard requirement per
   [ADR 0012](./0012-agent-provider-owned-session-storage.md), and
   [ADR 0016](./0016-resume-requires-filesystem-backed-sessions.md) requires the
   session record to be **filesystem-backed**. Kiro stores conversation state in
   a SQLite database (`~/Library/Application Support/kiro-cli/data.sqlite3`,
   selectable via `--session-source v1|v2`) — the exact case ADR 0016 names as
   not resumable. Kiro does expose `--resume-id <SESSION_ID>`, but the record
   behind it is a private DB schema, not a transferable file.

Mainline's contract would therefore reject Kiro outright ("the agent likely
cannot be supported until its CLI changes"). Supporting it anyway is the
deliberate reason this is a fork.

Empirical headless output was captured before deciding. It is clean — no ANSI
color, no spinner redraws — for both a plain question and a tool-using prompt:

```
All tools are now trusted (!). ...
Reading directory: /path (using tool: read, max depth: 0, ...)
 ✓ Successfully read directory /path (7 entries)
 - Completed in 0.0s

> Here are the files and directories:
- RESOURCES.md
...
 ▸ Credits: 0.06 • Time: 6s
```

With `--trust-all-tools`, Kiro auto-executes tools with no confirmation prompt —
required for unattended sandbox runs (a prompt would hang the agent). The
`AgentProvider` contract does not actually require JSON; `parseStreamLine` may
return any `ParsedStreamEvent`s, and `Orchestrator` already returns
`resultText || execResult.stdout` on success — so a provider that emits the raw
stdout as `text` events captures the full result without any structured stream.

## Decision

Support agent providers whose headless output is **unstructured text**, via a
plain-text passthrough parser, and ship Kiro as the first such provider —
**non-resumable**, using the escape hatch ADR 0016 already defines.

- **Passthrough parsing.** `parseKiroStreamLine` strips ANSI escapes from each
  stdout line and emits a single `{ type: "text" }` event for non-blank lines,
  `[]` for blank lines. It emits no `tool_call`, `session_id`, or `usage`
  events. Live output streams to `Display` as text, and the full stdout becomes
  the iteration result via the orchestrator's existing fallback.
- **Non-resumable, by ADR 0016.** Kiro ships with `captureSessions: false` and
  no `sessionStorage`, so `RunResult.resume` is typed `never` for it — no
  special-casing, the same shape OpenCode and Cursor already use. We do not wire
  `--resume-id`. This is ADR 0016 applied, not overridden: a DB-only session
  store does not qualify for resume.
- **Headless command.** `kiro-cli chat --no-interactive --model <m>
  [--effort <e>] [--agent <a>] [--trust-all-tools] -- <prompt>`. The prompt is a
  positional argv argument (required by `--no-interactive`); we guard it at
  120 KiB like the Cursor/Copilot providers, since Kiro headless does not accept
  the prompt on stdin. `--trust-all-tools` is appended when
  `dangerouslySkipPermissions` is set. Auth is `KIRO_API_KEY` via `env`.
- **Interactive mode.** `buildInteractiveArgs` returns
  `kiro-cli chat --model <m> [--effort] [--agent] [<prompt>]` for `interactive()`.

We accept that this departs from two documented must-haves *for this provider*.
The departure is deliberate, fork-scoped, and recorded here and in the guide; it
does not relax the requirements for other providers.

## Considered Options

1. **Heuristic structured parsing** — detect tool lines
   (`(using tool: …)`, `✓ Successfully …`), the `>` response marker, and the
   `▸ Credits • Time` footer to synthesize `tool_call` / `result` / `usage`
   events. Rejected: that wording is undocumented and changes between Kiro
   releases; coupling the parser to it trades robustness for cosmetic chrome we
   can add later, additively, without breaking the passthrough.
2. **Export sessions out of `data.sqlite3` to satisfy resume** — rejected for
   exactly the reasons in ADR 0016: reaching into another tool's private,
   versioned DB schema to extract and re-insert a session subgraph is a far
   heavier and more brittle problem than file transfer.
3. **Don't support Kiro until its CLI adds JSON + file-backed sessions** — the
   mainline stance. Rejected here because the whole point of the fork is to use
   Kiro now; the captured output shows the degraded experience is acceptable.
4. **Plain-text passthrough, non-resumable** (chosen) — minimal, robust, and
   reuses the orchestrator's stdout fallback and ADR 0016's `captureSessions:
   false` path with no new machinery.

## Consequences

- Kiro works for the core flow: run a prompt headlessly in a **sandbox**, stream
  output live, capture the result, and iterate — continuity across **iterations**
  comes from the git worktree on disk, as it already does for non-resumable
  providers.
- Three capabilities are intentionally absent for Kiro: structured tool-call
  rendering in the UI (Kiro's own inline text shows instead), per-iteration token
  usage (Kiro reports *credits*, not tokens), and cross-session resume.
- The provider contract is unchanged. `parseStreamLine` returning only `text`
  events is valid; no orchestrator or `Display` changes are needed.
- [`adding-an-agent-provider.md`](../agents/adding-an-agent-provider.md) gains a
  "Fork deviations" note recording that this fork relaxes the JSON-stream and
  resume must-haves for text-only providers; the upstream requirements stand for
  every other provider.
- Reversible and additive: if Kiro ships a JSON stream or a documented session
  export, a future provider revision can emit `tool_call` / `session_id` /
  `usage` and opt into resume without changing the passthrough default.
