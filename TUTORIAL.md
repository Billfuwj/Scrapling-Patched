# Scrapling 安装与使用教程（Patched 版本）

> **基于 [D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling) v0.4.2，已修复 P0 级无限循环风险**
> 适用于：与 OpenClaw 配合使用的反检测网页数据抓取

---

## 目录

1. [项目简介](#1-项目简介)
2. [安装指南](#2-安装指南)
3. [核心概念：三种引擎](#3-核心概念三种引擎)
4. [基础用法](#4-基础用法)
5. [进阶用法：Session 模式](#5-进阶用法session-模式)
6. [实战案例：电商评论抓取](#6-实战案例电商评论抓取)
7. [与 OpenClaw 集成](#7-与-openclaw-集成)
8. [代理配置](#8-代理配置)
9. [Adaptive 智能选择器](#9-adaptive-智能选择器)
10. [常见问题与排错](#10-常见问题与排错)
11. [补丁说明](#11-补丁说明)

---

## 1. 项目简介

Scrapling 是一个"三引擎合一"的 Python 反检测爬虫框架：

| 引擎 | 底层技术 | 适用场景 |
|------|---------|---------|
| **Fetcher** | curl_cffi (TLS 指纹伪装) | 纯 HTTP 请求，不需要 JS 渲染 |
| **DynamicFetcher** | Playwright | 需要 JS 渲染，但反检测要求不高 |
| **StealthyFetcher** | Patchright (反检测 Playwright 分支) | 需要绕过 WAF/Cloudflare |

**核心优势**：
- TLS 指纹伪装（JA3/JA4）
- 60+ Chrome stealth 启动参数
- 内置 Cloudflare Turnstile 自动破解
- Adaptive 智能选择器（网站改版后自动修复）
- 三种引擎返回统一的 Response 对象

---

## 2. 安装指南

### 2.1 环境要求

- Python >= 3.10
- macOS / Linux / Windows
- 建议使用虚拟环境

### 2.2 从 Patched 版本安装（推荐）

```bash
# 克隆 Patched 版本
git clone https://github.com/<your-username>/Scrapling-Patched.git
cd Scrapling-Patched

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate   # Windows

# 安装（含所有引擎依赖）
pip install -e ".[all]"

# 安装浏览器（StealthyFetcher/DynamicFetcher 需要）
playwright install chromium
```

### 2.3 从 PyPI 安装（原版，未修补）

```bash
pip install scrapling[all]
scrapling install
```

### 2.4 验证安装

```python
python -c "
from scrapling import Fetcher, StealthyFetcher, DynamicFetcher
print('Fetcher (HTTP):', Fetcher)
print('StealthyFetcher (Stealth):', StealthyFetcher)
print('DynamicFetcher (Dynamic):', DynamicFetcher)
print('All engines loaded successfully!')
"
```

### 2.5 可选依赖说明

```toml
# pyproject.toml 中的可选依赖组
[project.optional-dependencies]
fetchers = [...]    # HTTP + 浏览器引擎
spiders  = [...]    # Spider 爬虫框架
ai       = [...]    # MCP/AI 集成
all      = [...]    # 全部安装
```

---

## 3. 核心概念：三种引擎

### 3.1 Fetcher — 纯 HTTP 请求

```
请求 → curl_cffi (TLS伪装) → HTML → lxml 解析 → Response
```

- **速度最快**，资源消耗最低
- 适合：静态页面、API 接口、服务端渲染页面
- 不适合：需要 JS 渲染的 SPA 页面

### 3.2 DynamicFetcher — 标准浏览器

```
请求 → Playwright (Chromium) → JS渲染 → 完整DOM → Response
```

- 能执行 JavaScript
- 适合：React/Vue 等 SPA 页面
- 不适合：有严格反爬检测的网站

### 3.3 StealthyFetcher — 隐身浏览器

```
请求 → Patchright (反检测Chromium) → Stealth参数 → CF破解 → Response
```

- 最强反检测能力
- 适合：Amazon、Cloudflare 防护站点、韩国电商
- 代价：速度最慢，资源消耗最大

### 如何选择？

```
                    需要JS渲染？
                   /          \
                 否             是
                 |              |
              Fetcher      有反爬检测？
             (最快)       /          \
                        否             是
                        |              |
                  DynamicFetcher  StealthyFetcher
                    (中等)          (最强)
```

---

## 4. 基础用法

### 4.1 Fetcher（HTTP 模式）

```python
from scrapling import Fetcher

# 创建实例
fetcher = Fetcher(auto_match=False)

# GET 请求
response = fetcher.get("https://httpbin.org/get")
print(response.status)   # 200
print(response.url)      # 请求URL

# 解析 HTML
response = fetcher.get("https://quotes.toscrape.com/")
quotes = response.css("div.quote")
for quote in quotes:
    text = quote.css("span.text::text").get()
    author = quote.css("small.author::text").get()
    print(f"{author}: {text}")
```

### 4.2 StealthyFetcher（隐身模式）

```python
from scrapling import StealthyFetcher

# 基本用法
fetcher = StealthyFetcher(
    headless=True,           # 无头模式
    network_idle=True,       # 等待网络空闲
    block_webrtc=True,       # 防止 WebRTC IP 泄露
)

response = fetcher.fetch(
    "https://example.com",
    wait=2000,               # 额外等待 2 秒
    disable_resources=True,  # 屏蔽图片/字体/CSS（加速）
)

# 解析数据（和 Fetcher 完全一样的 API）
title = response.css("title::text").get()
```

### 4.3 DynamicFetcher（动态模式）

```python
from scrapling import DynamicFetcher

fetcher = DynamicFetcher(headless=True)

response = fetcher.fetch(
    "https://example.com",
    wait_selector="div.content",        # 等待特定元素出现
    wait_selector_state="visible",      # 等到可见
)
```

### 4.4 选择器语法

Scrapling 的 Response 对象支持多种选择器：

```python
# CSS 选择器（推荐，Scrapy 兼容）
response.css("div.product h2::text").get()           # 获取第一个匹配的文本
response.css("div.product h2::text").getall()        # 获取所有匹配的文本
response.css("a::attr(href)").get()                  # 获取属性值
response.css("div.product").css("span.price::text")  # 链式选择

# XPath 选择器
response.xpath("//div[@class='product']/h2/text()").get()

# 文本搜索
response.find_by_text("Add to Cart")                 # 精确匹配
response.find_by_text("price", partial=True)         # 模糊匹配
response.find_by_regex(r"\$[\d.]+")                  # 正则匹配

# 相似元素查找（AutoScraper 风格）
first_product = response.css("div.product").first
similar_products = first_product.find_similar()      # 找到所有相似元素
```

---

## 5. 进阶用法：Session 模式

Session 模式复用浏览器实例，**性能远高于每次新建**：

### 5.1 同步 Session

```python
from scrapling.engines._browsers._stealth import StealthySession

with StealthySession(
    headless=True,
    network_idle=True,
    block_webrtc=True,
) as session:
    # 在同一个浏览器中发送多个请求
    page1 = session.fetch("https://example.com/page1", wait=1000)
    page2 = session.fetch("https://example.com/page2", wait=1000)
    page3 = session.fetch("https://example.com/page3", wait=1000)

    # 每个 response 都是标准的 Selector 对象
    for resp in [page1, page2, page3]:
        print(resp.css("title::text").get())
```

### 5.2 异步 Session

```python
import asyncio
from scrapling.engines._browsers._stealth import AsyncStealthySession

async def main():
    async with AsyncStealthySession(
        headless=True,
        network_idle=True,
    ) as session:
        response = await session.fetch("https://example.com", wait=1000)
        print(response.css("title::text").get())

asyncio.run(main())
```

### 5.3 自定义页面操作（page_action）

```python
from scrapling import StealthyFetcher

def my_action(page):
    """在页面上执行自定义操作"""
    # 点击"加载更多"按钮
    page.click("button.load-more")
    page.wait_for_timeout(2000)

    # 滚动到底部
    page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
    page.wait_for_timeout(1000)

    # 填写表单
    page.fill("input[name='search']", "scrapling")
    page.click("button[type='submit']")
    page.wait_for_timeout(2000)

fetcher = StealthyFetcher(headless=True)
response = fetcher.fetch(
    "https://example.com",
    page_action=my_action,
    wait=1000,
)
```

---

## 6. 实战案例：电商评论抓取

### 6.1 Amazon 评论分层抓取

```python
import json
import time
import random
from scrapling import StealthyFetcher

ASIN = "B0C8HHV9DK"
BASE_URL = "https://www.amazon.com/product-reviews/{asin}"

star_filters = ["one_star", "two_star", "three_star", "four_star", "five_star"]
sort_options = ["recent", "helpful"]

all_reviews = []

fetcher = StealthyFetcher(
    headless=True,
    network_idle=True,
    block_webrtc=True,
)

for star in star_filters:
    for sort_by in sort_options:
        for page_num in range(1, 11):  # Amazon 最多 10 页
            url = (
                f"{BASE_URL.format(asin=ASIN)}"
                f"?filterByStar={star}"
                f"&sortBy={sort_by}"
                f"&pageNumber={page_num}"
            )

            print(f"[{star}/{sort_by}/p{page_num}] Fetching...")

            try:
                response = fetcher.fetch(url, wait=2000, disable_resources=True)
                reviews = response.css("div[data-hook='review']")

                if not reviews:
                    print(f"  → No reviews found, skipping remaining pages")
                    break

                for r in reviews:
                    review_data = {
                        "asin": ASIN,
                        "star_filter": star,
                        "sort": sort_by,
                        "title": r.css("a[data-hook='review-title'] span::text").get(""),
                        "rating": r.css("i[data-hook='review-star-rating'] span::text").get(""),
                        "text": r.css("span[data-hook='review-body'] span::text").get(""),
                        "date": r.css("span[data-hook='review-date']::text").get(""),
                        "verified": bool(r.css("span[data-hook='avp-badge']")),
                        "helpful": r.css("span[data-hook='helpful-vote-statement']::text").get(""),
                    }
                    all_reviews.append(review_data)

                print(f"  → Got {len(reviews)} reviews")

            except Exception as e:
                print(f"  → Error: {e}")
                continue

            # 随机延迟 2-5 秒，模拟人类行为
            time.sleep(random.uniform(2, 5))

# 去重
seen = set()
unique_reviews = []
for r in all_reviews:
    key = (r["title"], r["text"][:100] if r["text"] else "")
    if key not in seen:
        seen.add(key)
        unique_reviews.append(r)

# 保存
output_file = f"reviews_{ASIN}.json"
with open(output_file, "w", encoding="utf-8") as f:
    json.dump(unique_reviews, f, ensure_ascii=False, indent=2)

print(f"\nTotal scraped: {len(all_reviews)}")
print(f"After dedup: {len(unique_reviews)}")
print(f"Saved to: {output_file}")
```

### 6.2 韩国电商站点（Coupang 示例）

```python
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(
    headless=True,
    network_idle=True,
    locale="ko-KR",          # 韩国地区
    timezone_id="Asia/Seoul", # 首尔时区
)

def scroll_and_load(page):
    """模拟滚动加载更多内容"""
    for _ in range(5):
        page.evaluate("window.scrollBy(0, 800)")
        page.wait_for_timeout(1000)

response = fetcher.fetch(
    "https://www.coupang.com/np/search?q=laptop",
    page_action=scroll_and_load,
    wait=3000,
    extra_headers={"Accept-Language": "ko-KR,ko;q=0.9"},
)

products = response.css("li.search-product")
for p in products:
    name = p.css("div.name::text").get()
    price = p.css("strong.price-value::text").get()
    print(f"{name}: {price}")
```

---

## 7. 与 OpenClaw 集成

Scrapling 可以作为 OpenClaw 的**数据提取层**，两者职责分离：

```
OpenClaw (浏览器自动化/任务编排)
    ↓ 获取页面 HTML
Scrapling Selector (数据提取/智能匹配)
    ↓ 结构化数据
你的业务逻辑
```

### 7.1 用 Scrapling 的 Selector 解析 OpenClaw 获取的 HTML

```python
from scrapling import Selector

# OpenClaw 获取的 HTML 内容
html_content = """
<html>
  <body>
    <div class="product">
      <h2>Product Name</h2>
      <span class="price">$29.99</span>
    </div>
  </body>
</html>
"""

# 用 Scrapling 的 Selector 解析
selector = Selector(html_content)
name = selector.css("h2::text").get()
price = selector.css(".price::text").get()
print(f"{name}: {price}")
```

### 7.2 OpenClaw 负责导航 + Scrapling 负责提取

```python
from scrapling import Selector

# 假设 OpenClaw 已经获取了页面 HTML
def process_with_scrapling(html: str) -> list[dict]:
    """用 Scrapling 的解析引擎处理 OpenClaw 获取的 HTML"""
    selector = Selector(html)

    results = []
    for item in selector.css("div.product-card"):
        results.append({
            "name": item.css("h2::text").get(),
            "price": item.css(".price::text").get(),
            "rating": item.css(".rating::text").get(),
            "url": item.css("a::attr(href)").get(),
        })
    return results
```

### 7.3 利用 Adaptive 选择器应对网站改版

```python
from scrapling import Selector

# 首次运行：存储元素指纹
selector = Selector(html_content, auto_match=True, url="https://example.com")
price = selector.css("#price-tag", auto_save=True)  # 保存指纹到 SQLite

# 网站改版后：自动重新定位元素
selector2 = Selector(new_html_content, auto_match=True, url="https://example.com")
price = selector2.css("#price-tag", auto_match=True)  # 即使 CSS 变了也能找到
```

---

## 8. 代理配置

### 8.1 单个代理

```python
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(headless=True)
response = fetcher.fetch(
    "https://example.com",
    proxy="http://user:pass@proxy-host:port",
)
```

### 8.2 代理轮换

```python
from scrapling import StealthyFetcher

# 传入代理列表，自动轮换
fetcher = StealthyFetcher(
    headless=True,
    proxies=[
        "http://user:pass@proxy1:port",
        "http://user:pass@proxy2:port",
        "http://user:pass@proxy3:port",
    ],
)

# 每次 fetch 自动使用下一个代理
for url in urls:
    response = fetcher.fetch(url)
```

### 8.3 推荐代理服务

| 服务商 | 类型 | 价格 | 适用场景 |
|--------|------|------|---------|
| Bright Data | 住宅代理 | ~$12.5/GB | Amazon/大型电商 |
| SOAX | 住宅代理 | ~$6.6/GB | 韩国站点 |
| SmartProxy | 住宅代理 | ~$7/GB | 通用 |
| IPRoyal | 住宅代理 | ~$5/GB | 预算有限 |

---

## 9. Adaptive 智能选择器

这是 Scrapling 的独门绝技——**网站改版后选择器自动修复**。

### 9.1 工作原理

```
首次抓取:
  CSS选择器 "div.old-price" → 找到元素 → 存储指纹到 SQLite
  指纹内容: tag名、文本、属性、DOM路径、父元素、兄弟元素

网站改版后:
  "div.old-price" 不存在了 → 自动全DOM遍历
  → 对每个候选元素计算多维度相似度分数
  → 返回分数最高的元素（可能现在叫 "span.new-price"）
```

### 9.2 使用方法

```python
from scrapling import Fetcher

fetcher = Fetcher(auto_match=True)

# 第一次运行：保存元素指纹
response = fetcher.get("https://example.com/product")
price = response.css("div.price", auto_save=True)
print(f"Price: {price.text}")

# 之后运行：即使 CSS 结构变了，也能找到
response = fetcher.get("https://example.com/product")
price = response.css("div.price", auto_match=True)  # 自动重定位
print(f"Price: {price.text}")
```

### 9.3 find_similar — 自动发现相似元素

```python
# 给一个样本元素，找到页面上所有结构相似的元素
first_card = response.css("div.product-card").first
all_cards = first_card.find_similar()  # 自动找到所有类似的卡片

for card in all_cards:
    print(card.css("h2::text").get())
```

---

## 10. 常见问题与排错

### Q1: 安装 patchright 失败

```bash
# patchright 需要精确版本匹配
pip install patchright==1.58.2
patchright install chromium
```

### Q2: Cloudflare 仍然拦截

```python
# 确保开启了所有隐身选项
fetcher = StealthyFetcher(
    headless=True,           # 有些站点 headful 模式更好，试试 False
    block_webrtc=True,
    hide_canvas=True,
    solve_cloudflare=True,   # 默认开启
    google_search=True,      # 默认开启，模拟从Google跳转
)
```

### Q3: 页面内容为空

```python
# 增加等待时间
response = fetcher.fetch(
    url,
    wait=5000,                        # 等 5 秒
    network_idle=True,                # 等网络空闲
    wait_selector="div.content",      # 等特定元素出现
    wait_selector_state="visible",
)
```

### Q4: 被检测为机器人

```python
# 使用 page_action 模拟人类行为
def human_like(page):
    import random
    # 随机滚动
    for _ in range(3):
        page.evaluate(f"window.scrollBy(0, {random.randint(200, 600)})")
        page.wait_for_timeout(random.randint(500, 1500))
    # 随机鼠标移动
    page.mouse.move(random.randint(100, 800), random.randint(100, 600))

response = fetcher.fetch(url, page_action=human_like)
```

### Q5: 内存占用过高

```python
# 屏蔽不必要的资源
response = fetcher.fetch(
    url,
    disable_resources=True,  # 屏蔽图片/字体/CSS/媒体
    blocked_domains={"ads.example.com", "tracker.example.com"},
)
```

---

## 11. 补丁说明

本版本基于 Scrapling v0.4.2，修复了以下 P0 级问题：

### 修复 1: 无限循环保护

| 文件 | 原代码 | 修复后 | 超时时间 |
|------|--------|--------|---------|
| `convertor.py` _get_page_content | `while True` | `for attempt in range(20)` | ~10s |
| `convertor.py` _get_async_page_content | `while True` | `for attempt in range(20)` | ~10s |
| `_stealth.py` CF non-interactive wait | `while` 无限 | max 60 次 | ~60s |
| `_stealth.py` CF verify wait | `while` 无限 | max 40 次 | ~20s |
| `_stealth.py` CF iframe wait | `while` 无限 | max 20 次 | ~10s |
| 以上每项同步/异步版本均已修复 | | | |

### 未修改项

- `playwright==1.58.0` / `patchright==1.58.2` 版本锁定保持不变（确保兼容性）
- 所有原有功能和 API 接口保持不变

---

## 快速参考卡

```python
# === 一行代码抓取 ===
from scrapling import Fetcher, StealthyFetcher

# HTTP 模式（最快）
Fetcher().get("https://example.com").css("title::text").get()

# 隐身模式（最强）
StealthyFetcher(headless=True).fetch("https://example.com").css("title::text").get()

# === 选择器速查 ===
r.css("div.class")                    # CSS 选择器
r.css("a::attr(href)").getall()       # 所有链接
r.css("p::text").get()                # 文本内容
r.xpath("//div[@id='main']")          # XPath
r.find_by_text("keyword")            # 文本搜索
r.find_by_regex(r"\d+\.\d+")         # 正则搜索
element.find_similar()                # 相似元素
```

---

*Original project by Karim Shoair — [D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling) (BSD-3-Clause)*
*Patches applied for production stability.*
