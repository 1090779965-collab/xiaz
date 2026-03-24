---
name: function-calling
description: |
  Function CallingSkill。AI调用外部工具和能力的技术，实现与现实世界的连接。
  适用于：(1) 调用API (2) 查询数据库 (3) 执行操作 (4)
  用户说"调用工具"、"function calling"、"帮我查一下"等场景。
---

# Function Calling

## 核心概念

让AI能够：
1. 理解何时需要调用外部工具
2. 知道如何调用
3. 处理返回结果
4. 决定下一步行动

```
用户问题 → LLM判断 → 调用函数 → 获取结果 → 整合回答
```

## OpenAI Function Calling

### 定义函数

```python
functions = [
    {
        "name": "get_weather",
        "description": "获取指定城市的天气信息",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如北京、上海"
                }
            },
            "required": ["city"]
        }
    }
]
```

### 调用函数

```python
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "北京今天天气怎么样？"}],
    functions=functions
)

# 解析函数调用
function_call = response.choices[0].message.function_call
if function_call:
    function_name = function_call.name
    arguments = json.loads(function_call.arguments)
    
    # 执行函数
    if function_name == "get_weather":
        result = get_weather(arguments["city"])
```

## 工具设计原则

### 1. 函数命名清晰

| ❌ | ✅ |
|----|-----|
| do(x) | get_weather(location) |
| handle | query_database(sql) |
| process | send_email(to, subject, body) |

### 2. 参数简洁

```python
# ❌ 参数太多
def create_calendar_event(
    title, description, start_time, end_time,
    location, attendees, reminder, timezone,
    recurrence, color, visibility
):

# ✅ 拆分成多个函数
def create_event(title, start_time, end_time):
    # ...
    
def add_attendees(event_id, attendees):
    # ...
```

### 3. 描述准确

```python
{
    "name": "search_documents",
    "description": "搜索企业文档库，查找包含关键词的文档",
    "parameters": {
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索关键词，支持空格分隔的多个词"
            },
            "doc_type": {
                "type": "string",
                "enum": ["pdf", "docx", "xlsx", "all"],
                "description": "文档类型筛选"
            }
        },
        "required": ["query"]
    }
}
```

## 常用函数模板

### 1. 搜索

```python
{
    "name": "web_search",
    "description": "搜索互联网获取最新信息",
    "parameters": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "搜索关键词"},
            "max_results": {"type": "integer", "description": "返回数量"}
        },
        "required": ["query"]
    }
}
```

### 2. 查询

```python
{
    "name": "query_database",
    "description": "查询数据库获取数据",
    "parameters": {
        "type": "object",
        "properties": {
            "sql": {"type": "string", "description": "SQL查询语句"},
            "limit": {"type": "integer", "description": "返回行数限制"}
        },
        "required": ["sql"]
    }
}
```

### 3. 发送

```python
{
    "name": "send_notification",
    "description": "发送通知消息",
    "parameters": {
        "type": "object",
        "properties": {
            "channel": {
                "type": "string",
                "enum": ["email", "sms", "feishu", "dingtalk"]
            },
            "to": {"type": "string", "description": "接收人"},
            "content": {"type": "string", "description": "消息内容"}
        },
        "required": ["channel", "to", "content"]
    }
}
```

### 4. 日历

```python
{
    "name": "create_calendar_event",
    "description": "创建日历事件/会议",
    "parameters": {
        "type": "object",
        "properties": {
            "title": {"type": "string"},
            "start_time": {"type": "string", "description": "ISO 8601格式"},
            "end_time": {"type": "string"},
            "attendees": {"type": "array", "items": {"type": "string"}},
            "description": {"type": "string"}
        },
        "required": ["title", "start_time"]
    }
}
```

## 工作流模式

### 1. 单次调用

```
用户 → LLM判断 → 调用函数 → 返回结果 → 回答用户
```

### 2. 多次调用

```
用户 → LLM判断 → 调用函数1 → 返回结果 
                         ↓
                LLM判断 → 调用函数2 → 返回结果
                                         ↓
                               整合所有结果 → 回答用户
```

### 3. 循环调用

```
用户 → LLM判断 → 调用函数 → 检查结果
                              ↓
                    需要更多信息? 
                    ↙          ↘
                   是          否
                    ↓          ↓
              继续调用      回答用户
```

## 错误处理

```python
def handle_function_call(function_call):
    try:
        # 执行函数
        result = execute_function(function_call)
        return result
    except ValidationError as e:
        return {"error": f"参数错误: {e}"}
    except TimeoutError as e:
        return {"error": "请求超时，请重试"}
    except Exception as e:
        return {"error": f"未知错误: {e}"}
```

## 安全考虑

### 1. 权限控制

```python
def execute_function(function_call, user_permissions):
    # 检查用户权限
    function_name = function_call.name
    if function_name in user_permissions.get("denied", []):
        raise PermissionError(f"无权限调用 {function_name}")
    
    # 执行
    return actual_execute(function_call)
```

### 2. 输入验证

```python
def validate_arguments(function_name, arguments):
    # 验证参数类型和范围
    schema = function_schemas[function_name]
    validate(instance=arguments, schema=schema)
```

### 3. 审计日志

```python
def audit_log(function_call, user_id, result):
    log = {
        "timestamp": datetime.now(),
        "user": user_id,
        "function": function_call.name,
        "arguments": function_call.arguments,
        "success": "error" not in result
    }
    # 记录到审计日志
```

## 实践检查清单

- [ ] 函数描述清晰准确
- [ ] 参数定义完整
- [ ] 错误处理完善
- [ ] 权限控制到位
- [ ] 日志记录完整
- [ ] 延迟可接受

## 工具框架

| 框架 | 特点 |
|------|------|
| OpenAI Functions | 官方支持 |
| LangChain Tools | 生态丰富 |
| Claude Function Calling | 准头好 |
| Toolformer | 自学工具 |
