---
"@ai-hero/sandcastle": minor
---

Add Kiro CLI (`kiro-cli`) as an agent provider via the `kiro()` factory, driven in headless mode (`kiro-cli chat --no-interactive`). Kiro emits plain text rather than a JSON stream and stores sessions in SQLite, so the provider uses plain-text passthrough and is non-resumable (`captureSessions: false`), like the Cursor and OpenCode providers. Supports `--model`, `--effort`, `--agent`, and `--trust-all-tools`, with `sandcastle init` scaffolding. See ADR 0021.
