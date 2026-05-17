# 用Playwright 爬取 Redfin 数据，ScraperAPI 能帮你省掉多少麻烦？

我第一次试着用 Playwright 抓 Redfin 的时候，脚本跑了不到两分钟就被封了。换了 User-Agent，再试，还是封。加了随机延迟，依然封。那种感觉很熟悉——你知道数据就在页面上，但网站就是不让你拿。

Redfin 的反爬机制不算最凶的，但也绝对不好对付。它会检测无头浏览器特征、限制高频 IP、对自动化行为做指纹识别。单靠 Playwright 裸跑，成功率很不稳定。

> **ScraperAPI** 是一个专门处理反爬障碍的代理 API，支持直接与 Playwright 集成，帮你绕过 IP 封锁、验证码和浏览器指纹检测。适合需要稳定抓取房产数据的开发者。

>

---

## Redfin 为什么这么难爬

Redfin 的页面是动态渲染的——大量数据通过 JavaScript 异步加载，这就是为什么你需要 Playwright 而不是简单的 `requests`。但动态渲染只是第一道门槛，真正让人头疼的是：

- **IP 频率限制**：同一 IP 短时间内请求过多，直接 429 或静默封锁
- **无头浏览器检测**：Redfin 会检查 `navigator.webdriver`、Canvas 指纹、WebGL 特征等
- **地理位置限制**：某些数据只对特定地区 IP 开放
- **动态 CSS 类名**：选择器经常变，今天能用的 XPath 下周可能就失效了

光靠 Playwright 自带的 `stealth` 插件能解决一部分问题，但 IP 轮换这块它帮不了你。

---

## 用 ScraperAPI 配合 Playwright 的基本思路

ScraperAPI 的核心逻辑是：你把目标 URL 发给它的代理端点，它帮你处理 IP 轮换、请求头伪装、验证码绕过，然后把渲染好的 HTML 返回给你。

有两种接入方式：

**方式一：通过代理模式（Proxy Mode）**

把 ScraperAPI 当成一个 HTTP 代理，Playwright 的请求通过它转发出去。

```python
from playwright.sync_api import sync_playwright

# 你的 ScraperAPI API Key
API_KEY = "yourapi_key_here"
PROXY_URL = f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001"

with sync_playwright() as p:
    browser = p.chromium.launch(
        proxy={
            "server": "http://proxy-server.scraperapi.com:8001",
            "username": "scraperapi",
            "password": API_KEY,
        }
    )
    context = browser.new_context(
        ignore_https_errors=True  # ScraperAPI 使用自签证书，需要忽略
    )
    page = context.new_page()
    
    # 抓取 Redfin 某城市的房源列表
    page.goto("https://www.redfin.com/city/30772/CA/San-Francisco/filter/property-type=house")
    page.wait_for_load_state("networkidle")
    
    # 等待房源卡片加载
    page.wait_for_selector('[data-rf-testid="abp-homecard"]', timeout=3000)
    
    # 提取数据
    listings = page.query_selector_all('[data-rf-test-id="abp-homecard"]')
    for listing in listings:
        price = listing.query_selector('[data-rf-test-id="abp-price"]')
        address = listing.query_selector('[data-rf-test-id="abp-streetLine"]')
        if price and address:
            print(f"{address.inner_text()} - {price.inner_text()}")
    browser.close()
```

**方式二：通过 Async Request Mode（推荐用于大批量）**

直接调用 ScraperAPI 的 REST 接口，让它帮你渲染页面，你只需要处理返回的 HTML：

```python
import requests
from bs4 import BeautifulSoup

API_KEY = "your_api_key_here"

def scrape_redfin_page(url: str) -> BeautifulSoup:
    """
    通过 ScraperAPI 抓取 Redfin 页面
    render=true 参数告诉 ScraperAPI 需要 JS 渲染
    """
    params = {
        "api_key": API_KEY,
        "url": url,
        "render": "true",          # 启用 JS 渲染
        "country_code": "us",      # 使用美国 IP（Redfin 是美国房产网站）
        "premium": "true",         # 使用高级代理池，成功率更高
    }
    
    response = requests.get(
        "https://api.scraperapi.com/",
        params=params,
        timeout=60
    )
    if response.status_code == 200:
        return BeautifulSoup(response.text, "html.parser")
    else:
        raise Exception(f"请求失败：{response.status_code} - {response.text}")

def extract_listings(soup: BeautifulSoup) -> list[dict]:
    """从 Redfin 房源页面提取关键字段"""
    listings = []
    # Redfin 的房源卡片结构（注意：选择器可能随版本更新）
    cards = soup.find_all("div", {"data-rf-test-id": "abp-homecard"})
    for card in cards:
        listing = {}
        
        # 价格
        price_el = card.find(attrs={"data-rf-test-id": "abp-price"})
        listing["price"] = price_el.get_text(strip=True) if price_el else None
        
        # 地址
        street_el = card.find(attrs={"data-rf-test-id": "abp-streetLine"})
        city_el = card.find(attrs={"data-rf-test-id": "abp-cityStateZip"})
        listing["address"] = f"{street_el.get_text(strip=True)}, {city_el.get_text(strip=True)}" \
            if street_el and city_el else None
        
        # 卧室/浴室/面积
        stats = card.find_all("span", class_=lambda c: c and "stat" in c.lower())
        listing["stats"] = [s.get_text(strip=True) for s in stats]
        
        listings.append(listing)
    return listings

# 使用示例
if __name__ == "__main__":
    target_url = "https://www.redfin.com/city/30772/CA/San-Francisco/filter/property-type=house"
    
    soup = scrape_redfin_page(target_url)
    results = extract_listings(soup)
    
    for r in results:
        print(r)
```

---

## 抓取 Redfin 详情页：单个房源数据

列表页给你的是概览，真正有价值的数据在详情页——历史成交价、学区信息、税务记录、房屋描述。

```python
import asyncio
import aiohttp
from bs4 import BeautifulSoup

API_KEY = "your_api_key_here"

async def scrape_listing_detail(session: aiohttp.ClientSession, listing_url: str) -> dict:
    """异步抓取单个房源详情页"""
    params = {
        "api_key": API_KEY,
        "url": listing_url,
        "render": "true",
        "country_code": "us",
        "premium": "true",
    }
    
    async with session.get("https://api.scraperapi.com/", params=params) as resp:
        if resp.status != 200:
            return {"url": listing_url, "error": resp.status}
        
        html = await resp.text()
        soup = BeautifulSoup(html, "html.parser")
        
        detail = {"url": listing_url}
        # 房源标题/地址
        title = soup.find("h1", class_=lambda c: c and "address" in c.lower())
        detail["title"] = title.get_text(strip=True) if title else None
        
        # 挂牌价
        price = soup.find("div", {"data-rf-test-id": "abp-price"})
        detail["price"] = price.get_text(strip=True) if price else None
        # 房屋描述
        description = soup.find("p", class_=lambda c: c and "remarks" in c.lower())
        detail["description"] = description.get_text(strip=True) if description else None
        
        return detail

async def batch_scrape(urls: list[str]) -> list[dict]:
    """并发抓取多个详情页，控制并发数避免超出 API 限额"""
    connector = aiohttp.TCPConnector(limit=5)  # 最多 5 个并发请求
    
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [scrape_listing_detail(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if isinstance(r, dict)]

# 运行
listing_urls = [
    "https://www.redfin.com/CA/San-Francisco/123-Main-St-94102/home12345678",
    # 更多 URL...
]

results = asyncio.run(batch_scrape(listing_urls))
```

并发这块要注意：ScraperAPI 的并发限额取决于你的套餐，免费版是 1 个并发，Hobby 套餐是 5 个，往上走才能开更多。

---

## 处理 Redfin 的分页

Redfin 的搜索结果通常分页，URL 规律是 `usp=v3&num_homes=350&start=0`，`start` 参数控制偏移量。

```python
import time

def scrape_all_pages(base_url: str, max_pages: int = 10) -> list[dict]:
    """
    遍历 Redfin 搜索结果的多页
    Redfin 每页默认显示约 40 条，start 参数以 40 为步长递增
    """
    all_listings = []
    page_size = 40
    
    for page_num in range(max_pages):
        startoffset = page_num * page_size
        # 构造分页 URL（根据实际 Redfin URL 结构调整）
        paginated_url = f"{base_url}&start={start_offset}"
        
        try:
            soup = scrape_redfin_page(paginated_url)
            listings = extract_listings(soup)
            
            if not listings:
                print(f"第 {page_num + 1} 页无数据，停止翻页")
                break
            all_listings.extend(listings)
            print(f"第 {page_num + 1} 页：抓取到 {len(listings)} 条")
            
            # 礼貌性延迟，即使用了 ScraperAPI 也建议加
            time.sleep(1)
            
        except Exception as e:
            print(f"第 {page_num + 1} 页出错：{e}")
            break
    return all_listings
```

---

## ScraperAPI 套餐对比

| 套餐 | 月请求量 | 并发数 | JS 渲染 | 价格 | 适合场景 | 购买链接 |
| ------ | ------ | --------- | ------ | ------ | --- | --- |
| 免费版 | 1,000 次 | 1 | ✅ | 免费 | 测试和学习 | [免费注册试用](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 次 | 5 | ✅ | $49/月 | 个人项目、小规模抓取 | [开通 Hobby 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 250,000 次 | 10 | ✅ | $99/月 | 中等规模数据采集 | [开通 Startup 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 次 | 25 | ✅ | $249/月 | 大规模生产环境 | [开通 Business 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 自定义 | 自定义 | ✅ | 联系报价 | 企业级需求 | [联系 Enterprise 方案](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

抓 Redfin 这类 JS 渲染页面，每次请求会消耗 5 个 API 积分（因为启用了 `render=true`）。算一下：Hobby 套餐的 10 万次请求，实际可用于渲染页面的是 2 万次。做城市级别的房源扫描，Startup 套餐更合适。

---

## 常见问题

**Playwright 直接爬 Redfin 成功率有多低？**

裸跑的话，短时间内成功率还行，但跑几百个请求之后基本就开始大量失败。Redfin 的封锁不是立即的，它会先让你跑一段时间，然后突然开始返回空页面或者 403。加了 ScraperAPI 之后，IP 轮换在后台自动处理，成功率稳定很多。

**`render=true` 和不加这个参数有什么区别？**

不加的话，ScraperAPI 只返回服务器端的原始 HTML，Redfin 的动态内容不会加载。加了 `render=true`，ScraperAPI 会用无头浏览器完整执行 JavaScript，你拿到的是用户实际看到的页面内容。抓 Redfin 必须加这个参数。

**Redfin 的选择器经常变，怎么维护？**

这是真实痛点。建议用 `data-rf-test-id` 属性而不是 CSS 类名——Redfin 的 `data-rf-test-id` 相对稳定，类名则经常被混淆处理。另外可以考虑用 JSON-LD 结构化数据（`<script type="application/ld+json">`），Redfin 的详情页通常有这个，解析起来比 DOM 选择器稳定得多。

**免费版够用吗？**

测试代码逻辑完全够。1000 次请求，按每次消耗 5 积分算，能渲染 200 个页面，跑通整个抓取流程没问题。真正上生产再升级。

👉 [查看 ScraperAPI 完整定价和功能对比](https://www.scraperapi.com/pricing/?fp_ref=coupons)

**能抓 Redfin 的历史成交数据吗？**

可以，但要注意 Redfin 的历史数据有些是通过 API 异步加载的，不是初始 HTML 里就有的。你可能需要在 Playwright 里等待特定元素出现，或者直接拦截 Redfin 的内部 XHR 请求——后者效率更高，但需要先用 DevTools 分析它的网络请求规律。

---

用 Playwright 裸爬 Redfin 不是不行，但你会把大量时间花在跟封锁较劲上，而不是处理数据本身。ScraperAPI 把那层麻烦接走了，你只需要写好选择器逻辑和数据处理部分。

如果只是偶尔抓几个页面，免费版完全够用。要做持续性的房产数据采集，Startup 套餐的性价比最合理。

👉 [现在注册 ScraperAPI，免费额度直接可用](https://www.scraperapi.com/?fp_ref=coupons)
