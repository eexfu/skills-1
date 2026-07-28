---
name: web-reader
description: Read any URL and extract its main content (including JS-rendered pages that curl/wget can't handle) for the agent to read and analyse. Use when the user shares a link and asks to look at it, summarise it, or asks what an article/post says.
---

# Web Reader

通用网页阅读工具。给定任意 URL，提取其正文内容供 AI 阅读和分析。

## 何时使用

- 用户分享了一个链接说"看看这个"、"总结一下"、"这篇文章说了什么"
- 需要获取网页上的文章、文档、帖子内容
- 需要阅读 JS 渲染的页面（普通 curl/wget 拿不到内容的情况）

## 前置条件

- OpenCLI 已安装且 Chrome 扩展连接正常
- 依赖 `opencli-browser` skill

## 使用方法

### 单页阅读

```bash
# 打开页面并提取主要内容
opencli browser default open <url>
opencli browser default state
```

根据 `state` 返回的 DOM 结构，选择合适的选择器提取正文：

```bash
# 通用正文提取（尝试常见文章容器）
opencli browser default eval "(() => {
  const selectors = ['article', 'main', '.article', '.post-content', '.content', '[role=main]'];
  for (const s of selectors) {
    const el = document.querySelector(s);
    if (el && el.innerText.length > 200) return el.innerText.slice(0, 8000);
  }
  return document.body.innerText.slice(0, 8000);
})()"
```

### 简化流程

如果页面结构简单，直接用 `extract`：

```bash
opencli browser default open <url>
opencli browser default extract "article,main,.content" --attr text
```

### 长文处理

如果文章很长，需要分页/分段提取：

```bash
# 先滚动到底部确保懒加载内容已加载
opencli browser default scroll bottom

# 再次提取
opencli browser default eval "document.querySelector('article')?.innerText || document.body.innerText"
```

## 输出格式

提取到的文本直接返回给用户或放入上下文。如果用户要求总结，先完整提取再总结；如果用户要求引用，记录原文片段和 URL。

## 注意事项

- 对于需要登录才能查看的内容，确保用户在 Chrome 中已登录该网站
- 某些网站有反爬机制，如果提取失败，尝试先 `state` 查看页面实际加载的内容
- 如果页面是 PDF，OpenCLI 浏览器可能无法直接提取文本，需要告知用户
- 视频页面主要提取标题、描述、评论区，而非视频本身
