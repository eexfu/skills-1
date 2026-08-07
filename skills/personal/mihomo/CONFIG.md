# mihomo: config anatomy and day-to-day reference

`mode: Rule` routes by top-down rule matching: the first rule that matches a domain decides, and anything that matches nothing falls to the final `MATCH` rule (direct). Everything below refers to `~/.config/mihomo/config.yaml`.

## The manual rules block, line by line

A subscription-generated config works out of the box but does not route claude/codex. A hand-edited block must sit at the **very top** of the `rules:` list, before any subscription-provided rule:

```yaml
rules:
  # ===== manually added =====
  # Block Claude Code telemetry/logging (REJECT must be first, ahead of proxy rules)
  - DOMAIN-SUFFIX,datadoghq.com,REJECT
  - DOMAIN-SUFFIX,datadog.com,REJECT
  - DOMAIN-SUFFIX,statsig.com,REJECT
  - DOMAIN-SUFFIX,sentry.io,REJECT
  # ===== claude / codex via proxy =====
  # Anthropic / Claude Code (AI group, see "AI-dedicated group" below)
  - DOMAIN-SUFFIX,anthropic.com,🤖 AI 专用
  - DOMAIN-SUFFIX,claude.ai,🤖 AI 专用
  - DOMAIN-SUFFIX,claude.com,🤖 AI 专用
  - DOMAIN-SUFFIX,platform.claude.com,🤖 AI 专用
  - DOMAIN-SUFFIX,claude-code.com,🤖 AI 专用
  - DOMAIN-KEYWORD,claude,🤖 AI 专用
  # OpenAI / Codex
  - DOMAIN-SUFFIX,openai.com,🤖 AI 专用
  - DOMAIN-SUFFIX,chatgpt.com,🤖 AI 专用
  - DOMAIN-SUFFIX,oaistatic.com,🤖 AI 专用
  - DOMAIN-SUFFIX,oaiusercontent.com,🤖 AI 专用
  - DOMAIN-KEYWORD,codex,🤖 AI 专用
  # GitHub (codex/claude MCP and integrations)
  - DOMAIN-SUFFIX,github.com,🔰 手动选择
  - DOMAIN-SUFFIX,githubusercontent.com,🔰 手动选择
  - DOMAIN-SUFFIX,githubassets.com,🔰 手动选择
  - DOMAIN-KEYWORD,github,🔰 手动选择
  # npm registry (codex installs/updates deps)
  - DOMAIN-SUFFIX,npmjs.org,🔰 手动选择
  # Google SSO / OAuth (Claude login flow)
  - DOMAIN-SUFFIX,accounts.google.com,🔰 手动选择
  - DOMAIN-SUFFIX,googleapis.com,🔰 手动选择
  - DOMAIN-SUFFIX,gstatic.com,🔰 手动选择
  - DOMAIN-SUFFIX,withgoogle.com,🔰 手动选择
  - DOMAIN-SUFFIX,googleusercontent.com,🔰 手动选择
  - DOMAIN-SUFFIX,google.com,🔰 手动选择
  # ===== manually added ends; subscription rules below =====
```

Line-by-line anatomy:

- **REJECT rows** (datadoghq/datadog/statsig/sentry): drop Claude Code telemetry and logging at the network layer. They are the proxy-layer (second) line of defense; the first line is telemetry disabled by env vars, covered by the kimi-claude skill. **They must precede every proxy rule** — rule matching is top-down, so a proxy rule placed first would swallow the telemetry domains and let the traffic out.
- **Anthropic / Claude rows**: `anthropic.com`, `claude.ai`, `claude.com`, `platform.claude.com`, `claude-code.com`, plus the `claude` keyword. Required — this is the API traffic.
- **OpenAI / Codex rows**: `openai.com`, `chatgpt.com`, `oaistatic.com`, `oaiusercontent.com`, plus the `codex` keyword. Required for Codex.
- **GitHub / npm / Google rows**: for login flows and MCP integrations; optional but recommended, and always sent to the manual-select group so you can steer them case by case.
- **Rule targets are group names**: each third field must be the exact `name:` of a group that exists in `proxy-groups:`. A rule pointing at a missing group makes mihomo fail to load the config.

## Adding a proxy group

Claude/Codex API traffic needs a dedicated select group. Add it under `proxy-groups:`:

```yaml
proxy-groups:
  - name: "🤖 AI 专用"
    type: select
    proxies:
      - "YOUR-AI-NODE-1"    # replace with your subscription's actual node names
      - "YOUR-AI-NODE-2"
```

Notes:

- `type: select` gives a manually switched list; the current node is changed via the external-controller API (see SKILL.md).
- Node entries must be real `name:`s from your subscription's `proxies:` list.
- If your subscription has no AI-dedicated nodes, point the AI rules at `🔰 手动选择` instead of adding this group.
- `🔰 手动选择` and `🎯 Direct` are usually already provided by the subscription; confirm the exact names before referencing them, or mihomo fails to load.

## claude/codex domain lists

- **Required** (AI API traffic → `🤖 AI 专用`): the Anthropic/Claude and OpenAI/Codex rows above.
- **Optional** (login & MCP → `🔰 手动选择`): the GitHub, npm, and Google rows above. Trim or extend as your setup needs.

## AI-dedicated group (placeholder node names)

The `🤖 AI 专用` group (defined in "Adding a proxy group") must list your subscription's AI-capable nodes, always shown as placeholders — never publish real node names:

```yaml
proxies:
  - "YOUR-AI-NODE-1"    # replace with your subscription's actual node names
  - "YOUR-AI-NODE-2"
```

## Kimi (Moonshot) traffic

Kimi's API (`api.kimi.com`) is deliberately left out of the manual rules block, so in the default setup its traffic falls to `MATCH` (direct) — unless a subscription-provided rule matches it first. Reason: the domestic API is faster and more stable connected directly than through an overseas exit node, and Kimi is reachable from your region without a proxy.

To route Kimi traffic through the proxy instead, add one rule at the top of the manual block:

```yaml
  - DOMAIN-SUFFIX,api.kimi.com,🤖 AI 专用
```

The client side of Kimi (base URL, model ids, launchers) is the kimi-claude skill's territory — see that skill for the config surface, and keep the two in agreement about whether Kimi goes direct or via the proxy.

## Subscription update: re-apply flow

Re-downloading the subscription **overwrites `config.yaml`**, wiping the manual rules block and the `🤖 AI 专用` group. After every refresh:

1. Back up the current config: `cp ~/.config/mihomo/config.yaml ~/.config/mihomo/config.yaml.bak.$(date +%s)`.
2. Download the fresh config to `~/.config/mihomo/config.yaml`.
3. Re-apply the manual rules block and the AI-dedicated group from this document.
4. Validate: `mihomo -d ~/.config/mihomo -f ~/.config/mihomo/config.yaml -t`.
5. Reload: `stop.sh && start.sh`.

Backing up before step 2 means a botched re-apply is one `cp` away from rolling back.
