---
name: mihomo
description: "Operate the local mihomo proxy for Claude Code / Codex (start/stop/supervisor, Rule-mode config, proxy groups). Use when the proxy is down or unreachable, a node is slow or banned by a site, adding routing rules or proxy groups for new domains, updating the subscription config, switching nodes via the external-controller API, or blocking Claude telemetry at the proxy layer."
---

# mihomo

Local proxy operations for Claude Code / Codex.

## Architecture

mihomo only opens local ports (HTTP 7890 / SOCKS 7891) and never hijacks traffic; whether a program uses it is decided by `http_proxy` / `https_proxy` injection from the `.bashrc` `claude()` / `codex()` functions, which apply the proxy only for those two commands. In Rule mode only matched domains are proxied; everything else falls to the final `MATCH` rule (direct). No TUN mode, no root.

## Start / stop / supervisor

Three scripts live together in one directory (see SETUP.md): `start.sh` launches `supervisor.sh` via `setsid`, detaching it from the terminal; `supervisor.sh` restarts mihomo 3s after any crash and `wait`s on the child so no zombie is left behind; `stop.sh` SIGTERMs the supervisor first (which reaps mihomo), then force-kills any leftover. PIDs: `/tmp/mihomo.pid` and `/tmp/mihomo-supervisor.pid`. Log: `/tmp/mihomo.log`.

Verification quick-reference:

```bash
pgrep -x mihomo                    # running?
ss -tln | grep 7890                # port listening?
curl -x http://127.0.0.1:7890 https://api.ipify.org   # proxy works
tail -f /tmp/mihomo.log            # what is it doing?
```

## API operations

The external-controller listens on `127.0.0.1:9091`. Group names in the path must be URL-encoded:

```bash
GROUP=$(python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' '🤖 AI 专用')
curl -s http://127.0.0.1:9091/proxies/"$GROUP"                                                  # current node
curl -X PUT http://127.0.0.1:9091/proxies/"$GROUP" -d '{"name":"<node-name>"}'                  # switch node
```

Node-ban troubleshooting (generalized from the archive.org case): a site returns 502 / TLS errors or throttles traffic through one exit node → confirm the group's current node with the API → switch to another node in the same group → retest. If a site bans every node, add a routing rule for that domain (see CONFIG.md).

## Telemetry: second layer

REJECT rules for `datadoghq.com`, `datadog.com`, `statsig.com`, and `sentry.io` block Claude Code telemetry at the proxy layer. This is the second, fallback layer: the first layer — telemetry disabled by env vars at the source — belongs to the kimi-claude skill. Run both; removing the proxy rules alone leaves telemetry falling to DIRECT and stalling. The REJECT rules must come before any proxy rule (CONFIG.md).

## Kimi traffic

Kimi (Moonshot) API traffic is deliberately DIRECT — the manual block adds no rule for it, so by default it falls to `MATCH`. Why, and how to route it through a proxy instead, is in CONFIG.md.

## Companion files

- [SETUP.md](SETUP.md) — from-scratch install, subscription download, the three scripts, `.bashrc` functions, verification, backup habit.
- [CONFIG.md](CONFIG.md) — rules block anatomy, proxy groups, claude/codex domain lists, Kimi traffic, subscription re-apply flow.
