# mihomo: from-scratch setup

Complete setup of the local mihomo proxy on a fresh Linux machine. No external documents required.

## 1. Install mihomo

```bash
mkdir -p ~/.local/bin
cd /tmp
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.18.8/mihomo-linux-amd64-v1.18.8.gz
gunzip mihomo-linux-amd64-v1.18.8.gz
mv mihomo-linux-amd64-v1.18.8 ~/.local/bin/mihomo
chmod +x ~/.local/bin/mihomo
mihomo -v   # verify the install
```

Use the current release; adjust the version and platform in the asset name. Ensure `~/.local/bin` is on `PATH` (`export PATH="$HOME/.local/bin:$PATH"` in `.bashrc`).

## 2. Download the base config from your Clash subscription

The config path is fixed at `~/.config/mihomo/config.yaml` (hardcoded in `start.sh` / `supervisor.sh`):

```bash
mkdir -p ~/.config/mihomo
curl -L -o ~/.config/mihomo/config.yaml 'YOUR_CLASH_SUBSCRIPTION_URL'
```

> SENSITIVE: the subscription URL carries your account token, and the downloaded `config.yaml` contains your node credentials (UUIDs, server addresses). Treat both like keys: never commit them, never paste them into docs. The placeholder above must be replaced with the Clash/Clash Meta subscription address from your provider's user panel (it usually looks like `https://<provider>/api/v1/client/subscribe?token=...`).

Your provider may also require the geo data files `GeoSite.dat` / `geoip.metadb` next to the config; download them from the MetaCubeX release if a missing-file error appears. The subscription config is ready to run but needs the manual rules block before it routes claude/codex correctly — see CONFIG.md.

## 3. Start / stop / supervisor scripts

Keep the three scripts below together in one directory (for example `~/.local/bin/mihomo-scripts/`), `chmod +x` each. These are the canonical implementation; the content is published verbatim and is secret-free.

`start.sh`:

```bash
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG="${HOME}/.config/mihomo/config.yaml"
LOG="/tmp/mihomo.log"
PIDFILE="/tmp/mihomo.pid"
SUPERVISOR_PIDFILE="/tmp/mihomo-supervisor.pid"

# 检查是否已经有非 zombie 的 mihomo 在运行
is_alive() {
    local p=$1
    [ -z "$p" ] && return 1
    [ "$p" = "1" ] && return 1
    kill -0 "$p" 2>/dev/null || return 1
    local state=$(cat /proc/$p/stat 2>/dev/null | awk '{print $3}' || true)
    [ "$state" = "Z" ] && return 1
    return 0
}

RUNNING_PID=""
for p in $(pgrep -x mihomo 2>/dev/null || true); do
    if is_alive "$p"; then
        RUNNING_PID=$p
        break
    fi
done

if [ -n "$RUNNING_PID" ]; then
    echo "mihomo already running (PID: $RUNNING_PID)"
    echo "$RUNNING_PID" > "$PIDFILE"
    exit 0
fi

if [ ! -f "$CONFIG" ]; then
    echo "error: config not found: $CONFIG"
    exit 1
fi

# 清理旧的 pid 文件
rm -f "$PIDFILE" "$SUPERVISOR_PIDFILE"

# 用 setsid 启动 supervisor，使其脱离当前 terminal/session，
# 并在 mihomo 崩溃后自动重启；supervisor 会 wait 子进程，避免 zombie。
setsid "${SCRIPT_DIR}/supervisor.sh" >> "$LOG" 2>&1 &
SUPERVISOR_PID=$!
echo "$SUPERVISOR_PID" > "$SUPERVISOR_PIDFILE"

sleep 2

# 确认 mihomo 真的起来了（非 zombie）
NEW_PID=""
for p in $(pgrep -x mihomo 2>/dev/null || true); do
    if is_alive "$p"; then
        NEW_PID=$p
        break
    fi
done

if [ -n "$NEW_PID" ]; then
    echo "$NEW_PID" > "$PIDFILE"
    echo "mihomo started (PID: $NEW_PID, supervisor PID: $SUPERVISOR_PID)"
    echo "log: $LOG"
else
    echo "error: mihomo failed to start"
    rm -f "$PIDFILE" "$SUPERVISOR_PIDFILE"
    kill "$SUPERVISOR_PID" 2>/dev/null || true
    exit 1
fi
```

`stop.sh`:

```bash
#!/bin/bash

PIDFILE="/tmp/mihomo.pid"
SUPERVISOR_PIDFILE="/tmp/mihomo-supervisor.pid"
PID=""
SUPERVISOR_PID=""

# 优先从 PID 文件读取
if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE" 2>/dev/null || true)
fi
if [ -f "$SUPERVISOR_PIDFILE" ]; then
    SUPERVISOR_PID=$(cat "$SUPERVISOR_PIDFILE" 2>/dev/null || true)
fi

# 验证 PID 是否还活着且不是 zombie
is_alive() {
    local p=$1
    [ -z "$p" ] && return 1
    [ "$p" = "1" ] && return 1
    kill -0 "$p" 2>/dev/null || return 1
    local state=$(cat /proc/$p/stat 2>/dev/null | awk '{print $3}' || true)
    [ "$state" = "Z" ] && return 1
    return 0
}

if ! is_alive "$PID"; then
    PID=""
fi
if ! is_alive "$SUPERVISOR_PID"; then
    SUPERVISOR_PID=""
fi

# 回退到 pgrep，但跳过 zombie
if [ -z "$PID" ]; then
    for p in $(pgrep -x mihomo 2>/dev/null || true); do
        if is_alive "$p"; then
            PID=$p
            break
        fi
    done
fi

# 通过 mihomo 的 PPID 找 supervisor（仅当 PPID 是 bash 且不是 1 时）
if [ -z "$SUPERVISOR_PID" ] && [ -n "$PID" ]; then
    SPID=$(cat /proc/$PID/stat 2>/dev/null | awk '{print $4}' || true)
    if [ -n "$SPID" ] && [ "$SPID" != "1" ]; then
        CMD=$(cat /proc/$SPID/comm 2>/dev/null || true)
        if [ "$CMD" = "bash" ] && is_alive "$SPID"; then
            SUPERVISOR_PID=$SPID
        fi
    fi
fi

if [ -z "$PID" ] && [ -z "$SUPERVISOR_PID" ]; then
    echo "mihomo not running"
    rm -f "$PIDFILE" "$SUPERVISOR_PIDFILE"
    exit 0
fi

# 先给 supervisor 发 SIGTERM，让它去优雅地结束并回收 mihomo
if [ -n "$SUPERVISOR_PID" ]; then
    echo "stopping mihomo supervisor (PID: $SUPERVISOR_PID)..."
    kill "$SUPERVISOR_PID" 2>/dev/null || true
fi

# 等待最多 5 秒让 supervisor 完成清理
for i in 1 2 3 4 5; do
    sleep 1
    if ! is_alive "$SUPERVISOR_PID"; then
        SUPERVISOR_PID=""
    fi
    if ! is_alive "$PID"; then
        PID=""
    fi
    [ -z "$SUPERVISOR_PID" ] && [ -z "$PID" ] && break
done

# 如果还有残留，再强制清理
if [ -n "$PID" ] && is_alive "$PID"; then
    echo "force killing mihomo (PID: $PID)..."
    kill -9 "$PID" 2>/dev/null || true
fi
if [ -n "$SUPERVISOR_PID" ] && is_alive "$SUPERVISOR_PID"; then
    echo "force killing supervisor (PID: $SUPERVISOR_PID)..."
    kill -9 "$SUPERVISOR_PID" 2>/dev/null || true
fi

sleep 1

# 最终检查
ALIVE=false
for p in "$PID" "$SUPERVISOR_PID"; do
    [ -z "$p" ] && continue
    if is_alive "$p"; then
        echo "error: failed to stop process (PID: $p)"
        ALIVE=true
    fi
done

rm -f "$PIDFILE" "$SUPERVISOR_PIDFILE"

if [ "$ALIVE" = true ]; then
    exit 1
else
    echo "mihomo stopped"
fi
```

`supervisor.sh`:

```bash
#!/bin/bash
CONFIG="${HOME}/.config/mihomo/config.yaml"
LOG="/tmp/mihomo.log"
PIDFILE="/tmp/mihomo.pid"

if [ ! -f "$CONFIG" ]; then
    echo "error: config not found: $CONFIG" >&2
    exit 1
fi

MIHOMO_PID=""

# 收到终止信号时，先杀掉 mihomo 子进程并 wait 回收，避免产生 zombie
cleanup() {
    if [ -n "$MIHOMO_PID" ] && kill -0 "$MIHOMO_PID" 2>/dev/null; then
        kill "$MIHOMO_PID" 2>/dev/null || true
        wait "$MIHOMO_PID" 2>/dev/null || true
    fi
    rm -f "$PIDFILE"
    exit 0
}

trap cleanup SIGTERM SIGINT

while true; do
    mihomo -f "$CONFIG" >> "$LOG" 2>&1 &
    MIHOMO_PID=$!
    echo "$MIHOMO_PID" > "$PIDFILE"
    wait "$MIHOMO_PID" || true
    MIHOMO_PID=""
    echo "mihomo exited, restarting in 3s..." >> "$LOG"
    rm -f "$PIDFILE"
    sleep 3
done
```

## 4. `.bashrc` proxy-injection functions

Add the key-free versions below to `~/.bashrc`. NEVER paste your real `.bashrc` into docs or commits — it contains plaintext credentials. These functions are the safe, minimal subset:

```bash
# user-level executables (mihomo is installed here)
export PATH="$HOME/.local/bin:$PATH"

# Route Claude Code through the local mihomo proxy
claude() {
    http_proxy=http://127.0.0.1:7890 \
    https_proxy=http://127.0.0.1:7890 \
    HTTP_PROXY=http://127.0.0.1:7890 \
    HTTPS_PROXY=http://127.0.0.1:7890 \
    command claude "$@"
}

# Route Codex through the local mihomo proxy
codex() {
    http_proxy=http://127.0.0.1:7890 \
    https_proxy=http://127.0.0.1:7890 \
    HTTP_PROXY=http://127.0.0.1:7890 \
    HTTPS_PROXY=http://127.0.0.1:7890 \
    command codex "$@"
}
```

Notes: these are shell *functions* that shadow same-named commands; `command claude` bypasses the function to call the real binary, avoiding infinite recursion. Proxy variables only take effect inside the child process, so other terminal commands are unaffected. Extend the pattern to other tools by adding functions. `source ~/.bashrc` or open a new terminal after editing.

## 5. Verification checklist

```bash
# 1. mihomo running and port listening
pgrep -x mihomo
ss -tln | grep 7890

# 2. proxy actually works (returns the exit node IP, or the direct IP per rules)
curl -x http://127.0.0.1:7890 https://api.ipify.org

# 3. functions in effect
type claude    # should print "claude is a function"

# 4. smoke test through the proxy (non-interactive)
claude -p "reply with exactly: ok"
codex exec --help >/dev/null
```

Then use `claude` / `codex` interactively as usual.

## 6. Config backup habit

Before editing rules or refreshing the subscription, back up the current config:

```bash
cp ~/.config/mihomo/config.yaml ~/.config/mihomo/config.yaml.bak.$(date +%s)
```

A refresh overwrites your manual edits (CONFIG.md explains the re-apply flow), so a timestamped backup is what lets you roll back. After any config change, validate syntax with:

```bash
mihomo -d ~/.config/mihomo -f ~/.config/mihomo/config.yaml -t
```

Then reload with `stop.sh && start.sh` (see SKILL.md for the scripts).
