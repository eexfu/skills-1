# kimi-claude: from-scratch setup

Complete setup of Claude Code on the Kimi Code API on a fresh Linux machine. No external documents required.

Prerequisites: Node.js (via nvm recommended), `~/.local/bin` on `PATH` (`export PATH="$HOME/.local/bin:$PATH"`), and a Kimi Code API key from the Kimi Code console.

## 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
# or the official native installer: curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

## 2. Store the API key correctly

Put the key in a dedicated env file, never hardcoded in `.bashrc` (`.bashrc` only loads in interactive shells — launcher scripts would never see it — and a plaintext export there leaks into every subshell environment):

```bash
mkdir -p ~/.config/claude-kimi
cat > ~/.config/claude-kimi/env <<'EOF'
ANTHROPIC_API_KEY=YOUR_KIMI_API_KEY
EOF
chmod 600 ~/.config/claude-kimi/env
```

`claude-kimi-env` sources this file first, so an `ANTHROPIC_API_KEY` there wins over any shell-inherited value; only when `ANTHROPIC_API_KEY` is still empty does it fall back to copying a set `KIMI_API_KEY`. The env file is the recommended single place to store the key.

## 3. Shared injection script: `~/.local/bin/claude-kimi-env`

Single source of truth for all Kimi env injection, sourced by every entrypoint:

```bash
#!/usr/bin/env bash
# Kimi Code 环境变量注入(共享逻辑)
# 被 ~/.bashrc 的 claude() 函数和 ~/.local/bin/claude-k* 启动脚本共同 source。
# 调用前可选设置 CLAUDE_CODE_KIMI_MODEL 指定模型,默认 k3-256k。

# 可选:从 env 文件读取(非交互 shell 没有 .bashrc 里的 KIMI_API_KEY 时兜底)
if [[ -f "${HOME}/.config/claude-kimi/env" ]]; then
  set -a
  # shellcheck source=/dev/null
  source "${HOME}/.config/claude-kimi/env"
  set +a
fi

if [[ -z "${ANTHROPIC_API_KEY:-}" && -n "${KIMI_API_KEY:-}" ]]; then
    export ANTHROPIC_API_KEY="${KIMI_API_KEY}"
fi
export ANTHROPIC_BASE_URL="${ANTHROPIC_BASE_URL:-https://api.kimi.com/coding/}"

model="${CLAUDE_CODE_KIMI_MODEL:-k3-256k}"
export ANTHROPIC_MODEL="${model}"
export ANTHROPIC_DEFAULT_FABLE_MODEL="${ANTHROPIC_DEFAULT_FABLE_MODEL:-${model}}"
export ANTHROPIC_DEFAULT_OPUS_MODEL="${ANTHROPIC_DEFAULT_OPUS_MODEL:-k3}"
export ANTHROPIC_DEFAULT_SONNET_MODEL="${ANTHROPIC_DEFAULT_SONNET_MODEL:-k3-256k}"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="${ANTHROPIC_DEFAULT_HAIKU_MODEL:-k2.7}"
export CLAUDE_CODE_SUBAGENT_MODEL="${CLAUDE_CODE_SUBAGENT_MODEL:-k3-256k}"
export CLAUDE_CODE_EFFORT_LEVEL="${CLAUDE_CODE_EFFORT_LEVEL:-high}"

# 掐断遥测/日志/自动更新外发(源头层)
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
export DISABLE_NON_ESSENTIAL_MODEL_CALLS=1

case "${model}" in
    k3|k3-1m)
        export CLAUDE_CODE_AUTO_COMPACT_WINDOW="${CLAUDE_CODE_AUTO_COMPACT_WINDOW:-1048576}"
        export CLAUDE_CODE_MAX_CONTEXT_TOKENS="${CLAUDE_CODE_MAX_CONTEXT_TOKENS:-1048576}"
        ;;
    *)
        export CLAUDE_CODE_AUTO_COMPACT_WINDOW="${CLAUDE_CODE_AUTO_COMPACT_WINDOW:-262144}"
        export CLAUDE_CODE_MAX_CONTEXT_TOKENS="${CLAUDE_CODE_MAX_CONTEXT_TOKENS:-262144}"
        ;;
esac
```

## 4. `.bashrc` functions (key-free)

Add these functions to `~/.bashrc`. They contain no key — authentication comes from the chain in step 2:

```bash
# Default: claude = Kimi Code + local proxy
claude() {
    # shellcheck source=/dev/null
    source "${HOME}/.local/bin/claude-kimi-env"

    http_proxy=http://127.0.0.1:7890 \
    https_proxy=http://127.0.0.1:7890 \
    HTTP_PROXY=http://127.0.0.1:7890 \
    HTTPS_PROXY=http://127.0.0.1:7890 \
    command claude "$@"
}

# Escape hatch: start native Anthropic Claude
claude-anthropic() {
    unset ANTHROPIC_BASE_URL ANTHROPIC_API_KEY ANTHROPIC_MODEL
    unset ANTHROPIC_DEFAULT_FABLE_MODEL ANTHROPIC_DEFAULT_OPUS_MODEL
    unset ANTHROPIC_DEFAULT_SONNET_MODEL ANTHROPIC_DEFAULT_HAIKU_MODEL
    unset CLAUDE_CODE_SUBAGENT_MODEL CLAUDE_CODE_EFFORT_LEVEL
    unset CLAUDE_CODE_AUTO_COMPACT_WINDOW CLAUDE_CODE_MAX_CONTEXT_TOKENS
    unset CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC DISABLE_NON_ESSENTIAL_MODEL_CALLS
    command claude "$@"
}
```

## 5. Launcher family in `~/.local/bin/`

One script per model. Each sources `claude-kimi-env`, so the only difference is the `CLAUDE_CODE_KIMI_MODEL` value. `chmod +x` each file after creating it.

`~/.local/bin/claude-k3` — 1M context:

```bash
#!/usr/bin/env bash
export CLAUDE_CODE_KIMI_MODEL='k3'
# shellcheck source=/dev/null
source "${HOME}/.local/bin/claude-kimi-env"
http_proxy=http://127.0.0.1:7890 \
https_proxy=http://127.0.0.1:7890 \
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
exec claude "$@"
```

`~/.local/bin/claude-k3-256k` — 256k context:

```bash
#!/usr/bin/env bash
export CLAUDE_CODE_KIMI_MODEL='k3-256k'
# shellcheck source=/dev/null
source "${HOME}/.local/bin/claude-kimi-env"
http_proxy=http://127.0.0.1:7890 \
https_proxy=http://127.0.0.1:7890 \
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
exec claude "$@"
```

`~/.local/bin/claude-k27`:

```bash
#!/usr/bin/env bash
export CLAUDE_CODE_KIMI_MODEL='k2.7'
# shellcheck source=/dev/null
source "${HOME}/.local/bin/claude-kimi-env"
http_proxy=http://127.0.0.1:7890 \
https_proxy=http://127.0.0.1:7890 \
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
exec claude "$@"
```

`~/.local/bin/claude-k27-hs` — high-speed variant:

```bash
#!/usr/bin/env bash
export CLAUDE_CODE_KIMI_MODEL='k2.7-highspeed'
# shellcheck source=/dev/null
source "${HOME}/.local/bin/claude-kimi-env"
http_proxy=http://127.0.0.1:7890 \
https_proxy=http://127.0.0.1:7890 \
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
exec claude "$@"
```

`~/.local/bin/claude-kimi` — backwards-compatible alias, equivalent to `claude-k3-256k`:

```bash
#!/usr/bin/env bash
# Claude Code launcher for Kimi Code API（向后兼容别名，等价于 claude-k3-256k）
# 逻辑统一在 ~/.local/bin/claude-kimi-env，可通过 CLAUDE_CODE_KIMI_MODEL 覆盖模型。
# Docs: https://www.kimi.com/code/docs/third-party-tools/claude-code.html

# shellcheck source=/dev/null
source "${HOME}/.local/bin/claude-kimi-env"
http_proxy=http://127.0.0.1:7890 \
https_proxy=http://127.0.0.1:7890 \
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
exec claude "$@"
```

## 6. `~/.claude/settings.json` hardening

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.kimi.com/coding/",
    "ANTHROPIC_DEFAULT_FABLE_MODEL": "k2.7-highspeed",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "k3",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "k3-256k",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "k2.7",
    "CLAUDE_CODE_SUBAGENT_MODEL": "k3-256k",
    "CLAUDE_CODE_EFFORT_LEVEL": "high",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

`settings.json` `env` overrides shell-inherited variables of the same name, so three things never go in here: no API key (comes from the step-2 chain), no `ANTHROPIC_MODEL` (set dynamically per launcher), no context-window variables (they change per model).

## 7. Optional: local proxy

The launchers route traffic through a local proxy at `127.0.0.1:7890`; if you do not run one, delete the four `http_proxy`/`https_proxy` lines from each entrypoint — for the full proxy setup and telemetry REJECT rules, see the mihomo skill.

## 8. Verification checklist

1. Default entrypoint hits Kimi without a login prompt:
   ```bash
   claude -p "reply with exactly: ok"
   ```
2. Every launcher works:
   ```bash
   for l in claude-k3 claude-k3-256k claude-k27 claude-k27-hs; do $l -p ok; done
   ```
3. In a session, `/status` shows the Kimi Base URL and `/model` switches between opus/sonnet/haiku/fable with the expected Kimi model ids.
