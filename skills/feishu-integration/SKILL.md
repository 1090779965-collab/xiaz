# 📱 feishu-integration - 飞书深度集成

## 1. 技能概述

**核心能力**：深度集成飞书生态（多维表格、日历、文档、云空间），实现企业自动化

---

## 2. 核心能力

### 2.1 多维表格（Bitable）

```python
# 读取数据
app_token = "xxx"
table_id = "tblxxx"
records = bitable.list(app_token, table_id)

# 写入数据
bitable.create(app_token, table_id, {
    "字段名": "值"
})
```

### 2.2 日历管理

```python
# 创建日程
calendar.create_event(
    title="项目评审会",
    start_time="2026-03-28T14:00:00+08:00",
    duration_minutes=60,
    attendees=["ou_xxx"]
)

# 查询忙闲
freebusy.query(
    user_ids=["ou_xxx"],
    time_min="2026-03-28T00:00:00+08:00",
    time_max="2026-03-28T23:59:59+08:00"
)
```

### 2.3 文档操作

```python
# 创建文档
doc.create(title="周报模板", content="# 周报\n## 本周工作...")

# 读取文档
content = doc.get("doc_xxx")

# 更新文档
doc.update("doc_xxx", "append", "## 新增内容")
```

---

## 3. 实战案例

### 案例：自动填写日报

```
触发：每天17:00
流程：
1. 获取当日工作任务（从任务表）
2. 调用大模型生成日报
3. 写入多维表格"虾工作日报表"
4. 发送通知给主人
```

### 案例：会议智能提醒

```
触发：会前30分钟
流程：
1. 查询主人下个会议
2. 获取会议相关文档
3. 提前整理会议资料
4. 发送飞书消息提醒
```

---

## 4. 最佳实践

| 场景 | 工具 | 优势 |
|------|------|------|
| 结构化数据 | 多维表格 | 权限细、格式丰富 |
| 文档协作 | 云文档 | 版本管理、评论 |
| 定时任务 | 日历+机器人 | 提醒到位 |
| 流程审批 | 审批流 | 规范高效 |

---

*🦞 Skill by 虾米*
