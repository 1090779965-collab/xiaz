# 🌐 web-content-extraction - 网页内容提取

## 1. 技能概述

**核心能力**：自动抓取、解析、提取、总结网页内容

**适用场景**：
- 竞品监控：定期抓取对手网站更新
- 舆情分析：跟踪行业新闻和社交媒体
- 数据采集：从公开页面提取结构化数据
- 内容聚合：汇总多个来源的资讯
- 知识积累：保存感兴趣的文章供离线阅读

**对比传统方式**：
- 手动复制粘贴10篇文章：20分钟
- OpenClaw自动提取：30秒

---

## 2. 核心原理

### 2.1 爬取流程

```
URL → 请求 → 解析 → 提取 → 结构化 → 存储
```

### 2.2 技术栈

| 工具 | 用途 |
|------|------|
| fetch/axios | HTTP请求 |
| cheerio | HTML解析 |
| puppeteer | 动态页面渲染 |
| jina-ai_reader | AI内容提取 |

### 2.3 常用提取模式

```javascript
// 提取文章正文
const article = {
  title: $('h1').text(),
  author: $('[rel="author"]').text(),
  publishDate: $('time').attr('datetime'),
  content: $('article').text(),
  images: $('article img').map((i, el) => $(el).attr('src')).get()
};
```

---

## 3. 实际指令示例

### 示例1：提取文章内容
```
提取这篇博客的完整内容：https://example.com/blog/ai-trends
```

**输出示例**：
```
📄 文章标题：2026年AI发展的5大趋势

👤 作者：张三
📅 发布日期：2025-03-20
🔗 原文链接：https://example.com/blog/ai-trends

📝 内容摘要：
本文探讨了2026年AI领域的5个重要发展方向...

📊 关键要点：
1. 多模态模型成为主流
2. AI Agent应用爆发
3. 边缘AI设备普及
4. 开源模型性能超越闭源
5. AI监管框架完善

🖼️ 图片数量：3张
📖 全文长度：约3500字
```

---

### 示例2：批量采集结构化数据
```
采集某招聘网站的Python工程师职位信息
要求：城市北京、薪资15k-30k、工作3-5年
提取：职位名、公司名、薪资、要求、链接
```

**输出示例**：
```
🔍 共找到 45 个符合条件的职位

📋 职位列表（显示前10个）：

1. 高级Python开发工程师
   公司：字节跳动
   薪资：25k-40k·15薪
   要求：3-5年·本科
   链接：https://job.example.com/job/123

2. Python后端工程师
   公司：快手
   薪资：20k-35k·14薪
   要求：3-5年·本科
   ...

💾 已保存到：~/Data/jobs_beijing_python.csv
```

---

### 示例3：竞品监控
```
每周一抓取竞品官网首页，提取产品更新和新闻动态
```

**输出示例**：
```
🔍 竞品监控报告 - 2025-03-24

📌 竞品A (example-a.com)
• 新增功能：AI助手集成
• 更新文章：2篇
• 首页Banner：3个活动

📌 竞品B (example-b.com)  
• 新增功能：API v2.0
• 更新文章：5篇
• 首页Banner：1个促销

📊 对比总结：
- 竞品A本周主推AI能力
- 竞品B重点宣传API开放平台
- 行业趋势：AI + 开放生态
```

---

### 示例4：内容聚合
```
汇总过去24小时内AI领域的10条热门新闻
来源：HackerNews、TechCrunch、36kr
```

**输出示例**：
```
📰 AI要闻汇总 - 2025-03-24

来源分布：HackerNews(4) | TechCrunch(3) | 36kr(3)

🔥 Top 3 热门：

1. OpenAI发布GPT-5
   来源：HackerNews | 热度：482
   链接：https://news.ycombinator.com/item?id=123

2. Anthropic发布Claude 4
   来源：TechCrunch | 热度：356
   ...

3. 斯坦福发布开源AI Agent
   来源：36kr | 热度：289
   ...

💾 已保存到：~/News/ai-news-2025-03-24.md
```

---

## 4. 高级功能

### 4.1 动态页面处理
```javascript
// 使用 Puppeteer 处理 SPA/React/Vue 等动态页面
const browser = await puppeteer.launch();
const page = await browser.newPage();
await page.goto(url, { waitUntil: 'networkidle0' });
const html = await page.content();
```

### 4.2 AI智能提取
```
使用 jina-ai_reader 提取页面核心内容
- 自动去除广告、导航、侧边栏
- 保留文章主体结构
- 支持多语言
```

### 4.3 增量更新
```
对比上次抓取的内容，只返回新增/修改的部分
适用于：价格监控、库存跟踪、竞品更新
```

---

## 5. 最佳实践

### ✅ 合规
- 遵守 robots.txt
- 控制请求频率（>1秒/请求）
- 不抓取登录后的私有内容

### ✅ 性能
- 并发请求控制（<5个）
- 缓存已抓取内容
- 失败自动重试（3次）

### ✅ 数据质量
- 验证提取字段完整性
- 处理编码问题（GBK/UTF-8）
- 清洗HTML标签

---

## 6. 常见问题

**Q：部分页面内容抓不到？**
A：可能是动态渲染页面，需要用Puppeteer等待JS加载完成

**Q：被网站封了IP？**
A：降低请求频率，使用代理IP，或添加随机User-Agent

**Q：编码乱码？**
A：指定正确的编码（utf-8/gbk），或让AI自动检测

---

## 7. 实战练习

**任务1**：提取你关注的博客最新文章
```
抓取 https://example.com/blog 的最新5篇文章
```

**任务2**：采集行业资讯
```
采集36kr的AI频道今日热点新闻
```

**任务3**：竞品分析
```
抓取竞品官网的产品页面，提取所有产品功能和定价
```

---

## 8. 进阶方向

- [ ] 定时自动抓取（cron）
- [ ] 存储到向量数据库（RAG）
- [ ] 自动生成摘要报告
- [ ] 可视化数据分析

---

*🦞 Skill by 虾米 - 让信息获取自动化*
