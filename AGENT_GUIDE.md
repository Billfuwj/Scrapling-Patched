# Scrapling Agent Guide

> **This guide is designed for AI agents (e.g., OpenClaw) to read, install, configure, and use Scrapling autonomously.**
> Every step includes exact commands and expected outputs. Follow sequentially.

---

## Step 1: Environment Check

Run these commands to verify the environment is ready:

```bash
python3 --version
# Expected: Python 3.10+ (3.10, 3.11, 3.12, 3.13)
# If Python < 3.10, STOP. Install Python 3.10+ first.

pip3 --version
# Expected: pip 21+

git --version
# Expected: git 2+
```

---

## Step 2: Install Scrapling (Patched Version)

```bash
# Clone the patched repo
git clone https://github.com/Billfuwj/Scrapling-Patched.git
cd Scrapling-Patched

# Create virtual environment
python3 -m venv venv
source venv/bin/activate    # Linux/macOS
# venv\Scripts\activate     # Windows

# Install with all dependencies
pip install -e ".[all]"

# Install browser engine (required for StealthyFetcher)
playwright install chromium
```

### Verify Installation

```bash
python3 -c "
from scrapling import Fetcher, StealthyFetcher, Selector
print('OK: All modules imported successfully')
"
```

Expected output: `OK: All modules imported successfully`

If import fails, run: `pip install -e ".[fetchers]"` and retry.

---

## Step 3: Configure Proxy

Scrapling supports HTTP/HTTPS/SOCKS5 proxies. Proxy is **required** for production scraping to avoid IP bans.

### 3.1 Proxy Format

```
http://username:password@host:port
socks5://username:password@host:port
```

### 3.2 Single Proxy

```python
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(headless=True)
response = fetcher.fetch(
    "https://httpbin.org/ip",
    proxy="http://USER:PASS@PROXY_HOST:PORT",
)
print(response.text)  # Should show proxy IP, not real IP
```

### 3.3 Multiple Proxies with Auto-Rotation

```python
from scrapling import StealthyFetcher

proxy_list = [
    "http://user:pass@proxy1:port",
    "http://user:pass@proxy2:port",
    "http://user:pass@proxy3:port",
]

fetcher = StealthyFetcher(
    headless=True,
    proxies=proxy_list,  # Auto-rotates per request
)
```

### 3.4 Load Proxies from File

Create a file `proxies.txt`, one proxy per line:

```
http://user:pass@proxy1:port
http://user:pass@proxy2:port
http://user:pass@proxy3:port
```

Load and use:

```python
def load_proxies(filepath="proxies.txt"):
    with open(filepath, "r") as f:
        return [line.strip() for line in f if line.strip() and not line.startswith("#")]

proxy_list = load_proxies()
fetcher = StealthyFetcher(headless=True, proxies=proxy_list)
```

---

## Step 4: Three Engines — When to Use Which

### Decision Rule

```
IF target page is static HTML (no JavaScript needed):
    USE Fetcher              # Fastest, lowest resource usage

ELIF target page needs JavaScript rendering BUT has no anti-bot:
    USE DynamicFetcher       # Medium speed

ELIF target page has anti-bot protection (Cloudflare, WAF, etc.):
    USE StealthyFetcher      # Slowest but strongest anti-detection
```

### Engine Reference

#### Fetcher (HTTP mode — fastest)

```python
from scrapling import Fetcher

fetcher = Fetcher(auto_match=False)
response = fetcher.get("https://example.com")

# Extract data
title = response.css("title::text").get()
all_links = response.css("a::attr(href)").getall()
```

#### StealthyFetcher (stealth browser — strongest)

```python
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(
    headless=True,
    network_idle=True,
    block_webrtc=True,
)

response = fetcher.fetch(
    "https://example.com",
    wait=2000,
    disable_resources=True,
)

title = response.css("title::text").get()
```

#### DynamicFetcher (standard browser)

```python
from scrapling import DynamicFetcher

fetcher = DynamicFetcher(headless=True)
response = fetcher.fetch(
    "https://example.com",
    wait_selector="div.content",
    wait_selector_state="visible",
)

title = response.css("title::text").get()
```

---

## Step 5: Selector API Reference

All three engines return the same `Response` object. Use these methods to extract data:

### CSS Selectors

```python
# Get first match text
response.css("h1::text").get()

# Get first match text with default
response.css("h1::text").get(default="N/A")

# Get ALL match texts as list
response.css("p::text").getall()

# Get attribute value
response.css("a::attr(href)").get()

# Get all attribute values
response.css("img::attr(src)").getall()

# Nested/chained selection
response.css("div.product").css("span.price::text").get()

# Get first element
response.css("div.item").first

# Iterate elements
for item in response.css("div.item"):
    name = item.css("h2::text").get()
    price = item.css(".price::text").get()
```

### XPath Selectors

```python
response.xpath("//div[@class='product']/h2/text()").get()
response.xpath("//a/@href").getall()
```

### Text Search

```python
# Find element containing exact text
response.find_by_text("Add to Cart")

# Find element containing partial text
response.find_by_text("price", partial=True)

# Find by regex pattern
response.find_by_regex(r"\$[\d.]+")
```

### Parse Raw HTML (without fetching)

```python
from scrapling import Selector

html = "<html><body><h1>Hello</h1></body></html>"
selector = Selector(html)
title = selector.css("h1::text").get()  # "Hello"
```

---

## Step 6: Common Scraping Patterns

### Pattern A: Scrape a List Page

```python
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(headless=True, network_idle=True)
response = fetcher.fetch("https://example.com/products", wait=2000)

products = []
for item in response.css("div.product-card"):
    products.append({
        "name": item.css("h2::text").get(default=""),
        "price": item.css(".price::text").get(default=""),
        "url": item.css("a::attr(href)").get(default=""),
        "image": item.css("img::attr(src)").get(default=""),
    })
```

### Pattern B: Paginate Through Multiple Pages

```python
import time
import random

fetcher = StealthyFetcher(headless=True, network_idle=True)
all_data = []

for page_num in range(1, 11):
    url = f"https://example.com/products?page={page_num}"
    response = fetcher.fetch(url, wait=2000)

    items = response.css("div.product-card")
    if not items:
        break  # No more pages

    for item in items:
        all_data.append({
            "name": item.css("h2::text").get(default=""),
            "price": item.css(".price::text").get(default=""),
        })

    time.sleep(random.uniform(2, 5))  # REQUIRED: random delay between requests
```

### Pattern C: Click and Interact Before Extracting

```python
def page_action(page):
    """Custom actions on the page before extraction."""
    # Click a button
    page.click("button.load-more")
    page.wait_for_timeout(2000)

    # Scroll down
    page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
    page.wait_for_timeout(1000)

    # Type in a search box
    page.fill("input[name='search']", "keyword")
    page.click("button[type='submit']")
    page.wait_for_timeout(2000)

fetcher = StealthyFetcher(headless=True)
response = fetcher.fetch(
    "https://example.com",
    page_action=page_action,
    wait=1000,
)
```

### Pattern D: Amazon Review Scraping (Layered)

```python
import json
import time
import random
from scrapling import StealthyFetcher

ASIN = "B0C8HHV9DK"
star_filters = ["one_star", "two_star", "three_star", "four_star", "five_star"]
sort_options = ["recent", "helpful"]
all_reviews = []

fetcher = StealthyFetcher(headless=True, network_idle=True, block_webrtc=True)

for star in star_filters:
    for sort_by in sort_options:
        for page_num in range(1, 11):
            url = (
                f"https://www.amazon.com/product-reviews/{ASIN}"
                f"?filterByStar={star}&sortBy={sort_by}&pageNumber={page_num}"
            )
            try:
                response = fetcher.fetch(url, wait=2000, disable_resources=True)
                reviews = response.css("div[data-hook='review']")
                if not reviews:
                    break

                for r in reviews:
                    all_reviews.append({
                        "asin": ASIN,
                        "star_filter": star,
                        "sort": sort_by,
                        "title": r.css("a[data-hook='review-title'] span::text").get(""),
                        "rating": r.css("i[data-hook='review-star-rating'] span::text").get(""),
                        "text": r.css("span[data-hook='review-body'] span::text").get(""),
                        "date": r.css("span[data-hook='review-date']::text").get(""),
                        "verified": bool(r.css("span[data-hook='avp-badge']")),
                        "helpful": r.css("span[data-hook='helpful-vote-statement']::text").get(""),
                    })
            except Exception as e:
                continue
            time.sleep(random.uniform(2, 5))

# Deduplicate
seen = set()
unique = []
for r in all_reviews:
    key = (r["title"], r["text"][:100] if r["text"] else "")
    if key not in seen:
        seen.add(key)
        unique.append(r)

with open(f"reviews_{ASIN}.json", "w", encoding="utf-8") as f:
    json.dump(unique, f, ensure_ascii=False, indent=2)
```

---

## Step 7: Error Handling

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ModuleNotFoundError: No module named 'scrapling'` | Not installed | `pip install -e ".[all]"` |
| `playwright._impl._errors.Error: Executable doesn't exist` | Browser not installed | `playwright install chromium` |
| `TimeoutError` | Page took too long | Increase `wait` parameter or add `network_idle=True` |
| `Empty response / no elements found` | JS not fully rendered | Add `wait=5000` and `wait_selector="<target>"` |
| `403 Forbidden` | Bot detected | Switch to `StealthyFetcher`, add proxy |
| `Connection refused` | Proxy down | Check proxy, try next one |

### Retry Pattern

```python
import time

def fetch_with_retry(fetcher, url, max_retries=3, **kwargs):
    for attempt in range(max_retries):
        try:
            response = fetcher.fetch(url, **kwargs)
            if response.status == 200:
                return response
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise
    return None
```

---

## Step 8: Important Rules

1. **ALWAYS add random delays** between requests: `time.sleep(random.uniform(2, 5))`
2. **ALWAYS use proxies** for production scraping to avoid IP bans
3. **ALWAYS handle exceptions** — network requests can fail at any time
4. **ALWAYS deduplicate** results when scraping paginated content
5. **NEVER** send more than 10 requests per minute to the same domain without proxy rotation
6. **NEVER** ignore `robots.txt` without explicit user permission
7. **Prefer `disable_resources=True`** to speed up browser-based scraping
8. **Use `Fetcher` first** — only upgrade to `StealthyFetcher` if `Fetcher` gets blocked

---

## Quick Copy-Paste Templates

### Template: Minimal Scrape

```python
from scrapling import StealthyFetcher
fetcher = StealthyFetcher(headless=True)
r = fetcher.fetch("URL_HERE", wait=2000)
print(r.css("SELECTOR_HERE::text").get())
```

### Template: Bulk Scrape to JSON

```python
import json, time, random
from scrapling import StealthyFetcher

fetcher = StealthyFetcher(headless=True, network_idle=True)
results = []

urls = ["URL1", "URL2", "URL3"]
for url in urls:
    r = fetcher.fetch(url, wait=2000, disable_resources=True)
    results.append({
        "url": url,
        "title": r.css("title::text").get(""),
        "data": r.css("SELECTOR::text").getall(),
    })
    time.sleep(random.uniform(2, 5))

with open("output.json", "w") as f:
    json.dump(results, f, ensure_ascii=False, indent=2)
```

### Template: Parse HTML Without Fetching

```python
from scrapling import Selector

html = open("page.html", "r").read()
sel = Selector(html)
data = sel.css("SELECTOR::text").getall()
```

---

*Patched fork of [D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling) (BSD-3-Clause). Original author: Karim Shoair.*
