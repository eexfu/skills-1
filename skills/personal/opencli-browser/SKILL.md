---
name: opencli-browser
description: Drive the user's logged-in Chrome via OpenCLI for page navigation, element interaction, content extraction, screenshots, and automation. Use when the user wants to open or read a web page, click/type/fill forms on a site, extract data from a page, operate sites they're logged into (Zhihu, Bilibili, Twitter, Gmail, etc.), or take a page screenshot.
---

# OpenCLI Browser

通过 OpenCLI 操控用户已登录的 Chrome 浏览器，实现网页导航、元素交互、内容提取和自动化操作。

## 何时使用

当用户需要以下操作时：
- 打开并阅读某个网页内容
- 在网页上点击按钮、链接、填写表单
- 提取网页上的特定数据（标题、列表、表格等）
- 操作用户已登录的网站（如知乎、B站、Twitter、Gmail 等）
- 截取网页截图
- 等待页面加载或特定元素出现

## 前提条件

- 用户已安装 OpenCLI：`npm install -g @jackwener/opencli`
- Chrome 扩展已安装且 `opencli doctor` 显示连接正常
- 使用 `default` 作为默认 session 名称，除非用户指定了其他 session

## 核心命令

所有命令格式：`opencli browser <session> <command> [args] [flags]`

Session 通常用 `default`。

### 导航与状态

```bash
# 打开网页（返回 targetId）
opencli browser default open <url>

# 查看当前页面结构化的 DOM 快照
opencli browser default state

# 后退
opencli browser default back

# 刷新
opencli browser default eval "location.reload()"
```

### 内容提取

```bash
# 用 CSS 选择器提取元素文本或属性
opencli browser default extract <selector> --attr text
opencli browser default extract <selector> --attr href

# 查找元素位置（返回可交互的坐标/索引）
opencli browser default find <selector>

# 获取当前 URL、标题
opencli browser default eval "document.title"
opencli browser default eval "location.href"
```

### 交互操作

```bash
# 点击元素（可用 selector 或 find 返回的索引）
opencli browser default click <selector>

# 在输入框中输入文本
opencli browser default type <selector> "文本内容"

# 填充表单（多个字段）
opencli browser default fill <selector> '{"field1":"value1","field2":"value2"}'

# 选择下拉框
opencli browser default select <selector> "选项值"

# 发送键盘按键
opencli browser default keys <selector> "Enter"
opencli browser default keys <selector> "Escape"
```

### 等待与滚动

```bash
# 等待特定选择器出现
opencli browser default wait <selector> --timeout 5000

# 等待特定文本出现
opencli browser default wait "文本内容" --type text

# 滚动页面
opencli browser default scroll down
opencli browser default scroll up
opencli browser default scroll bottom
```

### 标签页管理

```bash
# 列出所有标签页
opencli browser default tab list

# 新建标签页
opencli browser default tab new [url]

# 切换到指定标签页
opencli browser default tab select <targetId>

# 关闭标签页
opencli browser default tab close <targetId>
```

### 高级

```bash
# 执行任意 JavaScript
opencli browser default eval "document.querySelector('h1').innerText"

# 截取页面截图（保存到文件）
opencli browser default screenshot --output /tmp/page.png

# 监听/查看网络请求（需要先开启）
opencli browser default network --list

# 查看 iframe 列表
opencli browser default frames
```

## 典型工作流

### 阅读一个网页
1. `opencli browser default open <url>`
2. `opencli browser default state` 或 `opencli browser default extract "article,main,.content" --attr text`
3. 如果需要滚动加载：`opencli browser default scroll bottom`
4. 再次提取内容

### 填写并提交表单
1. `opencli browser default open <url>`
2. `opencli browser default fill "#username" '{"value":"myuser"}'`
3. `opencli browser default type "#password" "mypass"`
4. `opencli browser default click "button[type=submit]"`
5. `opencli browser default wait ".success-message"`

### 在已登录网站操作
1. 直接用 `opencli browser default open <目标页面>`（复用 Chrome 登录态，无需再次登录）
2. 提取需要的数据

## 注意事项

- 优先使用 `state` 命令查看页面结构，它比截图更轻量且信息更丰富
- 选择器尽量用稳定的 id、class，避免过于复杂的路径
- 操作后如果页面变化，需要重新 `state` 获取最新 DOM 结构
- 如果命令失败，检查 `opencli doctor` 确认浏览器扩展连接正常
