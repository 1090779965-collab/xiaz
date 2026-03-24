# ☁️ openclaw-deployment - OpenClaw部署实战

## 1. 技能概述

**核心能力**：在多平台（Mac/Windows/Linux/云服务器）部署OpenClaw，实现7×24小时运行

---

## 2. 部署方案

### 2.1 Mac本地部署

```bash
# 1. 安装Node.js
brew install node@22

# 2. 安装OpenClaw
npm install -g openclaw@latest

# 3. 初始化配置
openclaw onboard --install-daemon

# 4. 启动
openclaw start
```

### 2.2 云服务器部署（Linux）

```bash
# 1. 安装Node.js
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs

# 2. 安装PM2（进程管理）
npm install -g pm2

# 3. 安装OpenClaw
npm install -g openclaw@latest

# 4. 后台运行
pm2 start openclaw --name "openclaw"
pm2 save
```

### 2.3 Docker部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  openclaw:
    image: openclaw/openclaw:latest
    volumes:
      - ./config:/root/.openclaw
      - ./data:/app/data
    ports:
      - "18789:18789"
    environment:
      - API_KEY=your-api-key
    restart: unless-stopped
```

---

## 3. 配置要点

### 3.1 必配项

| 配置项 | 说明 | 示例 |
|--------|------|------|
| channel | 通信渠道 | feishu/telegram/whatsapp |
| api-key | 大模型API | sk-xxx |
| model | 使用模型 | claude-sonnet-4 |

### 3.2 可选优化

```json
{
  "gateway": {
    "port": 18789,
    "password": "xxx"
  },
  "agent": {
    "model": "claude-sonnet-4-20250514",
    "temperature": 0.7
  }
}
```

---

## 4. 24×7运行配置

### 4.1 PM2进程管理

```bash
# 查看状态
pm2 status

# 查看日志
pm2 logs openclaw

# 重启
pm2 restart openclaw

# 开机自启
pm2 startup
pm2 save
```

### 4.2 健康检查

```bash
# 添加监控
pm2 monit
```

---

## 5. 常见问题

**Q：端口被占用？**
A：`lsof -i :18789` 查找占用进程

**Q：模型调用失败？**
A：检查API KEY配置和网络

**Q：消息收不到？**
A：检查channel配置和webhook

---

*🦞 Skill by 虾米*
