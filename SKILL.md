---
name: babata-browser
description: |
  巴巴塔浏览器控制技能 v2.1 — 基于 Playwright 的轻量浏览器自动化，自然语言控制，Accessibility Tree优先，零额外AI依赖。
  Use when: JS渲染页面抓取、政府网站动态政策采集、表单自动填报、截图取证、需要交互的网页。
  NOT for: 静态页面内容（用web_fetch）、API查询（用CLI）、简单文本搜索（用web_search）。
metadata:
  openclaw:
    emoji: 🦞
    requires:
      bins: [playwright]
      env: []
    primaryEnv: ''
---

# Babata Browser 🦞 v2.1

> Playwright驱动的轻量浏览器自动化。先扫描再操作，先文字再截图。

---

## When to Use ✅

- 政府网站动态政策抓取（中纪委/卫健委/医保局 — 常需JS渲染）
- JS渲染的SPA页面数据采集（GitHub Trending、ClawHub等）
- 表单自动填报、按钮点击、搜索交互
- 截图取证（视觉验证需要的场景）
- web_fetch返回空白/乱码/<500字符的页面
- 微信公众号文章（需JS渲染+登录态）

## When NOT to Use ❌

- 静态内容页面 → 用 `web_fetch`（更快更轻）
- API/结构化数据查询 → 用 `gh CLI` 或直接 `fetch()`
- 纯文本搜索 → 用 `web_search`（Tavily）
- 批量数据下载 → 用脚本比浏览器快

---

## 安装

```bash
pip install playwright
python -m playwright install chromium
cd skills/babata-browser && pip install -e .
```

---

## 核心设计原则

### 1. 先扫描再操作（来自 smart-browser 最佳实践）

不要盲目snapshot整个页面。先用JS片段找到可交互元素：

```python
# 扫描页面可交互元素（只返回可见的按钮/链接/输入框）
result = browser.execute_js(page, """
  (() => {
    const els = document.querySelectorAll('a[href], button, input, select, textarea, [role=button], [onclick]');
    return [...els]
      .filter(el => { const r = el.getBoundingClientRect(); return r.width > 0 && r.height > 0 && r.top < window.innerHeight; })
      .map((el, i) => ({
        i, tag: el.tagName.toLowerCase(),
        text: (el.innerText || el.value || el.placeholder || '').trim().slice(0, 50),
        id: el.id, href: el.href?.slice(0, 80)
      }));
  })()
""")
```

### 2. 按文字点击，不猜选择器

```python
# ✅ 用文字匹配点击（最稳定）
browser.click(page, text='最新政策')

# ❌ 不要硬编码CSS选择器
browser.click(page, selector='#content > div:nth-child(3) > a')
```

### 3. 条件等待，不固定sleep

```python
# ✅ 等特定文字出现
browser.execute_js(page, """
  new Promise(resolve => {
    let tries = 0;
    const timer = setInterval(() => {
      if (document.body.innerText.includes('目标文字') || ++tries > 30) {
        clearInterval(timer);
        resolve(tries < 30 ? 'found' : 'timeout');
      }
    }, 500);
  })
""")
```

### 4. 分层提取：先区域再细节

```
Accessibility Snapshot → 找到目标区域
  → get_text(selector=区域) → 提取文字
  → 还是不清晰 → screenshot（最后手段）
```

---

## 使用方式

### Python API（推荐）

```python
from scripts.babata_browser import execute_task

# 一句话控制
result = execute_task('打开 https://www.nhc.gov.cn，搜索 医疗政策，提取前5条标题')
# result = {'task': '...', 'steps': ['OK: ...', 'Searched: 医疗政策'], 'data': '...'}
```

### 底层API（精确控制）

```python
from scripts.babata_browser import BabataBrowser

browser = BabataBrowser(headless=True)
browser.start()
page = browser.new_page()

# 导航
browser.goto(page, 'https://example.com')
# 等待条件满足
browser.wait(page, 2000)
# 获取可交互元素列表（用JS扫描）
browser.get_links(page, limit=30)
# 按文字点击
browser.click(page, text='同意')
# 填表单
browser.fill(page, 'input[name="username"]', 'value')
# 提取正文
text = browser.get_text(page)
# 截图（最后手段）
browser.screenshot(page, path='evidence.png')

browser.stop()  # ⚠️ 必须关闭，否则进程残留
```

### CLI模式

```bash
babata-browser '打开 GitHub Trending，提取热门项目' --json
```

---

## 内置能力

| 动作 | 说明 | 典型场景 |
|------|------|---------|
| `goto(url)` | 导航到URL | 打开目标页面 |
| `get_text(sel?)` | 提取文字（可选限定区域） | 提取页面正文 |
| `get_links(limit)` | 提取所有链接 | 获取导航菜单、搜索结果 |
| `click(text=, sel=)` | 点击（文字或CSS匹配） | 翻页、提交、导航 |
| `fill(sel, val)` | 填写输入框 | 搜索框、登录表单 |
| `screenshot(path)` | 全页截图 | 取证、视觉验证 |
| `scroll(n)` | 滚动页面 | 懒加载内容 |
| `execute_js(code)` | 执行JS | 扫描元素、条件等待 |
| `extract_table(sel)` | 提取表格数据 | 数据表格 |

---

## 错误处理

| 错误 | 原因 | 处理 |
|:----|:-----|:-----|
| `Page.goto: net::ERR_TIMED_OUT` | 网络超时 | 增加timeout: `goto(page, url, timeout=60000)` |
| CloudFlare "Just a moment..." | 被CF拦截 | 该站点无法用浏览器绕过，改用其他数据源 |
| `Element not found` | 元素不存在 | 先scan再用text匹配，不用CSS选择器 |
| `page.click: Timeout` | JS未加载完 | 用条件等待替代固定sleep |
| 浏览器进程残留 | 未调用stop() | 使用try/finally确保stop() |

---

## 安全规则

- **不要在不可信网站上输入真实凭证** — 浏览器操作可能被JS窃取
- **截图前检查页面内容** — 避免截图保存敏感信息
- **headless=True（默认）** — 减少系统资源占用
- **使用后必须stop()** — 防止浏览器进程残留

---

## 对比Playwright MCP

| | Playwright MCP | babata-browser v2.1 |
|---|---|---|
| 依赖 | Node.js + npx + Chromium | **仅 Python + Playwright + Chromium** |
| 安装 | npx自动 | pip install |
| AI决策 | MCP客户端决定 | **巴巴塔LLM直接决策** |
| Token效率 | MCP协议开销 | **CLI直接调用，零协议开销** |
| 持久状态 | ✅ 支持 | 按需创建/销毁 |
| 适用 | 长周期自动化 | **高频交互、采样取证** |

---

## 变更日志

| 版本 | 日期 | 改动 |
|:----:|:----:|------|
| v2.1 | 2026-05-11 | 新增When NOT to Use、错误处理表、安全规则、智能扫描JS片段（来自smart-browser）、条件等待模式、分层提取策略。从GitHub成熟技能学习Did/Don't模式 |
| v2.0 | 2026-05-07 | Accessibility Tree优先策略、CLI/MCP双模式。来源：Playwright MCP |
| v1.0 | — | 初始版本 |
