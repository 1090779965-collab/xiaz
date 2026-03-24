---
name: government-scraper
description: |
  政府网站爬取Skill。针对政府/监管类网站的数据抓取，处理SPA应用、反爬机制。
  适用于：(1) 政府数据平台 (2) 监管信息查询 (3) SPA应用爬取 (4)
  用户说"抓取政府网站"、"监管平台数据"、"特殊食品/化妆品查询"等场景。
---

# 政府网站爬取

## 核心挑战

| 挑战 | 特点 | 解决方案 |
|------|------|----------|
| SPA应用 | 需JS渲染 | Browser自动化 |
| 反爬机制 | 验证码、IP限制 | 代理池、降低频率 |
| 数据分散 | 需多页遍历 | 队列管理 |
| 结构复杂 | 动态加载、隐藏字段 | 逆向分析 |

## 技术方案

### 方案1：Browser自动化（推荐）

**工具**：Playwright / Selenium

```python
from playwright.sync_api import sync_playwright

def scrape_special_food(keyword):
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()

        # 访问首页
        page.goto("http://ypzsx.gsxt.gov.cn/specialfood/")

        # 等待加载
        page.wait_for_load_state("networkidle")

        # 输入关键词
        page.fill("input[type='text']", keyword)
        page.click("button[type='submit']")

        # 等待搜索结果
        page.wait_for_selector(".search-result")

        # 提取结果
        results = page.eval_on_selector_all(
            ".search-result",
            "els => els.map(el => el.textContent)"
        )

        browser.close()
        return results
```

**优点**：
- ✅ 支持JS渲染
- ✅ 自动等待加载完成
- ✅ 可处理验证码

**缺点**：
- ❌ 速度慢
- ❌ 资源占用高

### 方案2：API逆向（高级）

**步骤**：
1. 打开开发者工具 → Network标签
2. 搜索"免疫" → 观察XHR请求
3. 找到数据接口URL（通常是/api/xxx）
4. 分析请求参数和响应格式

```python
import requests

def search_via_api(keyword):
    url = "http://ypzsx.gsxt.gov.cn/api/search"
    params = {
        "keyword": keyword,
        "page": 1,
        "pageSize": 20
    }
    response = requests.post(url, json=params)
    return response.json()
```

**优点**：
- ✅ 速度快
- ✅ 资源占用低
- ✅ 数据格式规范（JSON）

**缺点**：
- ❌ 需要技术分析
- ❌ API可能变动

### 方案3：混合方案（推荐）

```
首选API逆向 → 失败则用Browser → 两者都不行则手动
```

---

## 保健食品原料提取 - 具体实现

### 目标网站
- **URL**：http://ypzsx.gsxt.gov.cn/specialfood/
- **数据**：产品名称、配方原料、批准文号

### 实现流程

```
1. 逆向API
   ↓ 失败
2. Browser自动化
   ↓ 提取列表页
3. 爬取详情页
   ↓ 提取原料字段
4. 数据整理
```

### 伪代码

```python
import time
from playwright.sync_api import sync_playwright

class SpecialFoodScraper:
    def __init__(self):
        self.browser = None

    def __enter__(self):
        self.browser = sync_playwright().chromium.launch()

    def __exit__(self, *args):
        if self.browser:
            self.browser.close()

    def search(self, keyword, max_pages=10):
        """搜索保健食品"""
        with self.browser.new_page() as page:
            # 访问首页
            page.goto("http://ypzsx.gsxt.gov.cn/specialfood/")

            # 搜索
            page.fill("input[placeholder*='产品名称']", keyword)
            page.click("button[type='submit']")

            results = []
            for page_num in range(1, max_pages + 1):
                # 等待结果
                page.wait_for_selector(".result-item", timeout=10000)

                # 提取当前页结果
                items = self._extract_result_list(page)

                # 点击进入详情页提取原料
                for item in items:
                    detail_url = self._get_detail_url(page, item)
                    detail = self._scrape_detail(detail_url)
                    results.append(detail)

                    print(f"✅ 已提取: {detail['name']}")
                    time.sleep(1)  # 避免过快

                # 翻页
                try:
                    page.click(f"button[page='{page_num + 1}']")
                except:
                    break

        return results

    def _extract_result_list(self, page):
        """提取结果列表"""
        return page.eval_on_selector_all(
            ".result-item",
            """els => els.map(el => ({
                name: el.querySelector('.product-name')?.textContent,
                url: el.querySelector('a')?.href
            }))"""
        )

    def _get_detail_url(self, page, item):
        """获取详情页URL"""
        # 点击查看更多
        page.click(f"a[href='{item['url']}']")
        page.wait_for_url_regex(r'.*detail.*', timeout=5000)

        # 返回当前URL
        return page.url

    def _scrape_detail(self, url):
        """抓取详情页原料信息"""
        with self.browser.new_page() as page:
            page.goto(url)
            page.wait_for_load_state("networkidle")

            # 提取原料（假设字段名）
            materials = page.eval_on_selector_all(
                ".material-item",
                """els => els.map(el => el.textContent.trim())"""
            )

            return {
                'name': page.eval_on_selector(
                    ".product-name",
                    "el => el.textContent"
                ),
                'materials': materials,
                'url': url
            }
```

---

## 批量处理

### 关键词管理

```python
# 关键词列表
keywords = ["免疫", "抗氧化", "增强免疫", "抗疲劳"]

# 批量搜索
all_results = []
with SpecialFoodScraper() as scraper:
    for keyword in keywords:
        print(f"🔍 搜索: {keyword}")
        results = scraper.search(keyword, max_pages=5)
        all_results.extend(results)
        time.sleep(2)  # 避免触发反爬
```

### 数据去重

```python
import pandas as pd

# 转为DataFrame
df = pd.DataFrame(all_results)

# 去重（按产品名称）
df = df.drop_duplicates(subset=['name'])

# 统计原料出现频次
material_counts = df.explode('materials')['materials'].value_counts()

# 导出
df.to_excel("保健食品原料.xlsx")
material_counts.to_excel("原料频次统计.xlsx")
```

---

## 反爬应对

| 问题 | 解决方案 |
|------|----------|
| IP限制 | 使用代理池，每个代理爬取10页 |
| 验证码 | 手动输入或第三方打码平台 |
| 频率限制 | 随机延迟1-3秒 |
| UA检测 | 轮换User-Agent |

---

## 注意事项

### 合规性
- ⚠️ 遵守《网络安全法》
- ⚠️ 不影响网站正常服务
- ⚠️ 遵守robots.txt

### 效率
- 建议夜间爬取
- 使用多线程（注意IP限制）
- 本地缓存已抓取数据

### 稳定性
- 异常重试（3次）
- 超时设置（60秒）
- 失败日志记录

---

## 工具推荐

| 工具 | 用途 |
|------|------|
| Playwright | 浏览器自动化 |
| Selenium | 浏览器自动化 |
| Scrapy | 框架化爬虫 |
| Pandas | 数据处理 |
| 代理池 | IP轮换 |
