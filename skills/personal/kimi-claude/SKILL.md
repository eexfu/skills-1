---
name: kimi-claude
description: "Configure and troubleshoot Claude Code running on Kimi (Moonshot) via claude-kimi-env and launcher scripts. Use when switching Kimi models (k3 / k3-256k / k2.7 / k2.7-highspeed), debugging 401 auth or model-not-found errors, adding a new Kimi model id, configuring context window (256k vs 1M), or checking telemetry-disable env vars."
---

# kimi-claude

## How it works

Routing to Kimi is decided entirely by environment variables, not by any proxy: `ANTHROPIC_BASE_URL=https://api.kimi.com/coding/` points Claude Code at the Kimi Code endpoint and `ANTHROPIC_API_KEY` authenticates against it. `~/.local/bin/claude-kimi-env` is the single source of truth for that injection (model ids, alias slots, context-window sizing, telemetry kill switches). The launcher scripts only set `CLAUDE_CODE_KIMI_MODEL`, add the local proxy, and `exec claude`.

## Launcher family

| Command | Model id | Context window |
|---|---|---|
| `claude` (`.bashrc` function) | `k3-256k` (default) | 256k |
| `claude-k3` | `k3` | 1M |
| `claude-k3-256k` | `k3-256k` | 256k |
| `claude-k27` | `k2.7` | 256k |
| `claude-k27-hs` | `k2.7-highspeed` | 256k |
| `claude-kimi` | alias of `claude-k3-256k` | 256k |

Escape hatches in `.bashrc`: `claude()` runs the Kimi default; `claude-anthropic()` unsets every Kimi variable and starts native Anthropic Claude. Ad-hoc switch without remembering script names: `CLAUDE_CODE_KIMI_MODEL=k3 claude`.

## Core pitfalls

- Launcher scripts do not load `.bashrc`: in a non-interactive shell `claude` resolves to the real binary, not the function, so injection logic must live in `claude-kimi-env` and be sourced — never assumed from the interactive shell.
- `settings.json` `env` overrides shell-inherited variables of the same name. Three nevers for `settings.json`: no API key, no `ANTHROPIC_MODEL` (set dynamically from `CLAUDE_CODE_KIMI_MODEL`), no context-window variables (they change per model).
- The 1M model id is `k3`, not `k3[1m]`. Bracket syntax is Anthropic's 1M-context convention; Kimi rejects it with 401 "model id does not exist".
- `k2.7` / `k2.7-highspeed` are undocumented-but-working aliases (official ids: `kimi-for-coding` / `kimi-for-coding-highspeed`). If the endpoint ever stops accepting them, swap the ids back in `claude-kimi-env` and `settings.json`.
- Four config surfaces must stay in sync: launcher scripts, `claude-kimi-env`, `settings.json`, `.bashrc`.

## Telemetry: first layer

`claude-kimi-env` exports `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` and `DISABLE_NON_ESSENTIAL_MODEL_CALLS=1`, so telemetry, logging, and auto-update traffic is never generated at the source. This first layer works standalone. For the second layer — REJECT rules at the local proxy as a safety net — see the mihomo skill.

## Troubleshooting

### 401 / auth errors

1. Bypass the client and test the API directly:

```bash
curl https://api.kimi.com/coding/v1/messages \
  -H "x-api-key: $KIMI_API_KEY" -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"k3-256k","max_tokens":16,"messages":[{"role":"user","content":"hi"}]}'
```

2. Check how `claude-kimi-env` resolves the key: `~/.config/claude-kimi/env` is sourced first, so an `ANTHROPIC_API_KEY` there wins; only when `ANTHROPIC_API_KEY` is still empty is a set `KIMI_API_KEY` copied into it. Confirm which one your entrypoint actually sees — a launcher in a non-interactive shell never sees `.bashrc` exports.
3. Discriminate transient rate-limiting from real failure: rapid successive calls can return transient 401s that clear after a few seconds. If the direct curl also fails consistently, the key or model id is genuinely wrong.

### Model-not-found errors

Check the id against the launcher table above. Using `k3[1m]` instead of `k3` is the usual cause.

### Adding a new Kimi model id

Sync all four config surfaces:

1. `claude-kimi-env` — extend alias defaults and the context-window `case` arm if the new id has a non-256k window.
2. A new `~/.local/bin/claude-<name>` launcher exporting `CLAUDE_CODE_KIMI_MODEL='<id>'` (copy an existing one, change one line, `chmod +x`).
3. `settings.json` `env` — update alias slots (`ANTHROPIC_DEFAULT_*_MODEL`, `CLAUDE_CODE_SUBAGENT_MODEL`) only; still no key, no `ANTHROPIC_MODEL`, no window variables.
4. `.bashrc` — nothing required; keep it key-free.

Then verify with `/model` in-session and a `claude -p` smoke test.

## Verification quick-reference

- `/status` — confirm Base URL is the Kimi endpoint.
- `/model` — switch opus/sonnet/haiku/fable and confirm the resolved ids.
- `claude -p "reply with exactly: ok"` — smoke test; must answer without a login prompt.

## Full setup

See [SETUP.md](SETUP.md) for the complete from-scratch setup with verbatim scripts.
