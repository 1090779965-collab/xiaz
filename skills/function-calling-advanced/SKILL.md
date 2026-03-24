# 🔌 function-calling-advanced - 函数调用进阶

## 1. 技能概述

**核心能力**：让AI自主调用外部API，实现与真实世界的连接

**适用场景**：
- 实时数据查询（天气、股票、新闻）
- 跨系统操作（创建日历、发送邮件）
- 数据库交互（查询、写入）
- IoT设备控制

---

## 2. 进阶模式

### 2.1 基础模式：被动调用

```
LLM: 我需要查天气
  ↓
工具: get_weather(城市="北京")
  ↓
LLM: 北京今天晴，25°C
```

**特点**：LLM明确要求才调用

### 2.2 进阶模式：智能判断

```
LLM分析后自动决定：
- 用户问"明天去上海要带伞吗？" → 自动调天气API
- 用户说"帮我约周三下午3点" → 自动调日历API
- 用户问"这个文件多大" → 自动调文件系统API
```

### 2.3 自主模式：ReAct循环

```
循环直到任务完成：
1. 思考：需要什么信息？
2. 行动：调用合适的工具
3. 观察：分析返回结果
4. 决策：继续或结束
```

---

## 3. 工具设计

### 3.1 函数定义规范

```json
{
  "name": "create_calendar_event",
  "description": "创建日历事件",
  "parameters": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "事件标题"
      },
      "start_time": {
        "type": "string", 
        "format": "datetime",
        "description": "开始时间 ISO8601格式"
      },
      "duration_minutes": {
        "type": "integer",
        "description": "持续分钟数"
      },
      "attendees": {
        "type": "array",
        "items": {"type": "string"},
        "description": "参会人邮箱列表"
      }
    },
    "required": ["title", "start_time"]
  }
}
```

### 3.2 最佳实践

| 原则 | 说明 |
|------|------|
| 描述清晰 | description要详细说明功能 |
| 参数必填 | required明确哪些必须 |
| 类型准确 | string/integer/array/boolean |
| 示例有用 | 适当提供枚举值 |

---

## 4. 实际案例

### 案例1：智能日程管理

```
用户：帮我安排周五下午2点的项目评审会

[函数调用链]
1. create_event
   title: "项目评审会"
   start_time: "2026-03-28T14:00:00+08:00"
   duration_minutes: 60
   
2. send_email (自动触发)
   to: ["team@company.com"]
   subject: "会议邀请：项目评审会"
   
[返回]
✅ 已创建会议：周五 14:00-15:00
✅ 已发送邀请邮件给团队成员
```

### 案例2：数据分析pipeline

```
用户：分析这个CSV文件，找出销售最好的产品

[函数调用链]
1. read_file
   path: "sales_data.csv"
   
2. analyze_data (LLM内部处理)
   - 解析CSV
   - 按销售额排序
   - 计算增长率
   
[返回]
📊 销售Top 3产品：
1. iPhone 15 Pro - ¥12,000,000 (+23%)
2. MacBook Air - ¥8,500,000 (+15%)
3. AirPods Pro - ¥6,200,000 (+8%)
```

### 案例3：多API组合

```
用户：帮我买最近一周科技行业最热的股票

[函数调用链]
1. search_news
   query: "科技行业 热门股票"
   category: "finance"
   
2. get_stock_price (循环)
   symbol: ["AAPL", "MSFT", "GOOGL"]
   
3. send_email
   subject: "今日股票分析"
   content: "根据最新科技新闻..."
   
[返回]
📈 今日科技股分析：
- AAPL: $178.50 (+1.2%)
- MSFT: $412.30 (+0.8%)
建议关注：AI概念股持续升温
```

---

## 5. 错误处理

### 5.1 常见错误

| 错误类型 | 原因 | 处理方式 |
|----------|------|----------|
| 参数错误 | 格式/类型不匹配 | 重试或询问用户 |
| 超时 | 网络/服务慢 | 重试3次后放弃 |
| 权限 | 未授权 | 引导授权流程 |
| 限流 | API调用超限 | 等待后重试 |

### 5.2 容错设计

```python
async def call_with_retry(func, max_retries=3):
    for i in range(max_retries):
        try:
            return await func()
        except RetryableError as e:
            if i == max_retries - 1:
                raise
            await asyncio.sleep(2 ** i)  # 指数退避
        except PermanentError as e:
            return {"error": str(e), "fallback": "xxx"}
```

---

## 6. 性能优化

### 6.1 批量操作

```
❌ 逐个调用：100次API
✅ 批量调用：1次批量API

# 错误示例
for user in users:
    send_email(user)  # 100次

# 正确示例
batch_send_email(users)  # 1次
```

### 6.2 缓存策略

```
频繁查询的数据加缓存：
- 用户信息：缓存30分钟
- 配置参数：缓存1小时
- 热门数据：缓存5分钟
```

### 6.3 并发控制

```
控制并发数避免被限流：
semaphore = asyncio.Semaphore(5)  # 最多5个并发
```

---

## 7. 最佳实践

### ✅ 设计原则

- **最小权限**：只请求必要的权限
- **幂等性**：重复调用不会产生副作用
- **可观测**：每次调用记录日志
- **超时控制**：设置合理超时时间

### ⚠️ 安全注意

- 不在函数中暴露敏感信息
- 验证用户身份和权限
- 限制单次/批量操作数量

---

## 8. 常见问题

**Q：LLM不调用函数？**
A：检查function calling是否正确配置，prompt中强调可用工具

**Q：返回结果太长？**
A：设置max_tokens或后处理截取关键信息

**Q：调用顺序混乱？**
A：用链式调用确保顺序，或让LLM自己判断

---

## 9. 实战练习

**任务1**：实现天气查询
```
创建天气查询函数，支持：
- 指定城市
- 3天预报
- 带伞提醒
```

**任务2**：实现日程创建
```
创建日历事件函数，支持：
- 标题、时间、时长
- 自动发送邀请
```

---

## 10. 进阶方向

- [ ] Webhook：让外部事件触发Agent
- [ ] 长时任务：处理异步任务回调
- [ ] 多租户：隔离不同用户的API调用

---

*🦞 Skill by 虾米 - 让AI连接真实世界*
