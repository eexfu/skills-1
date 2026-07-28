---
name: opencli-usage
description: Query which sites, commands, and parameters OpenCLI supports. Use when the user asks what sites OpenCLI can operate, what subcommands a site has, what arguments a command takes, or when you need to discover available adapters.
---

# OpenCLI Usage

查询 OpenCLI 支持的站点、命令及其用法。

## 何时使用

- 用户问"OpenCLI 能操作哪些网站？"
- 用户想知道某个站点有哪些子命令
- 不确定某个命令的具体参数时
- 需要发现可用的适配器

## 命令

```bash
# 列出所有已安装的命令/适配器
opencli list

# 查看某个站点/命令的帮助
opencli <site> --help
opencli <site> <command> --help

# 诊断 OpenCLI 环境（浏览器扩展连接状态）
opencli doctor

# 查看已连接的 Chrome 配置（profile）
opencli profile list
opencli profile use <profile-name>

# 查看内置适配器列表（官方维护）
# 目前直接运行 opencli list 即可看到所有注册命令
```

## 常见站点速查

以下站点通常开箱即用（需已登录 Chrome）：

| 站点 | 常见命令 |
|------|---------|
| B站 | `opencli bilibili hot`, `opencli bilibili search <关键词>`, `opencli bilibili dynamic` |
| 知乎 | `opencli zhihu hot`, `opencli zhihu search <关键词>`, `opencli zhihu question <id>` |
| 小红书 | `opencli xiaohongshu search <关键词>`, `opencli xiaohongshu feed` |
| Twitter/X | `opencli twitter trending`, `opencli twitter timeline`, `opencli twitter search <关键词>` |
| Reddit | `opencli reddit hot`, `opencli reddit frontpage`, `opencli reddit subreddit <name>` |
| HackerNews | `opencli hackernews top`, `opencli hackernews new`, `opencli hackernews best` |
| GitHub | `opencli github repo <user>/<repo>`, `opencli github pr list` |

## 用法示例

```bash
# 获取 HackerNews 热榜，限制 5 条，JSON 输出
opencli hackernews top --limit 5 -f json

# 知乎热榜，表格输出（默认）
opencli zhihu hot --limit 10

# B站热门视频
opencli bilibili hot --limit 5
```

## 自定义适配器

如果发现某个站点没有内置适配器，可以使用 `opencli-browser` skill 里的命令直接操作浏览器，或者考虑用 `opencli adapter` 相关命令创建自定义适配器（详见 OpenCLI 官方文档）。
