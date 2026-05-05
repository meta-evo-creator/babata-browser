---
name: babata-browser
description: 巴巴塔浏览器控制技能 — 基于 Playwright 的轻量浏览器自动化，自然语言控制，零额外AI依赖
---

# Babata Browser 🦞

> 轻量浏览器自动化技能。给巴巴塔装一双"网页上的手"——打开网页、填写表单、点击按钮、提取数据、截图保存。

## 对比 browser-use

| | browser-use | babata-browser |
|---|---|---|
| 依赖 | 50+包 | **仅 Playwright** |
| 安装 | 300MB+/20min | 100MB/2min |
| 控浏览器 | ✅ | ✅ |
| AI决策 | 内置LLM | **巴巴塔LLM直接决策** |
| 中文任务 | 一般 | ✅ 原生中文 |

## 安装

```bash
pip install playwright
python -m playwright install chromium
```

## 使用

```python
from scripts.babata_browser import execute_task

# 一句话操控浏览器
execute_task("打开卫健委官网，搜索最新政策，提取前5条标题")
execute_task("打开 https://example.com，搜索 医疗AI，提取结果")
execute_task("打开登录页，填表提交，截图保存")
```

## 内置能力

| 动作 | 说明 |
|------|------|
| `goto` | 导航到URL |
| `get_text` | 提取页面文字 |
| `get_html` | 获取HTML |
| `click` | 点击元素(文本/CSS) |
| `fill` | 填写表单 |
| `get_links` | 提取所有链接 |
| `screenshot` | 全页截图 |
| `scroll` | 滚动页面 |
| `execute_js` | 执行JavaScript |
| `extract_table` | 智能提取表格 |
| `search_and_extract` | 搜索+提取 |
| `login_if_needed` | 自动登录 |

## 应用场景

- 卫健委/医保局/中纪委官网动态政策抓取
- 政府监管系统自动填报
- JS渲染页面数据采集
- 网页内容变化监控
- 自动化表单提交
