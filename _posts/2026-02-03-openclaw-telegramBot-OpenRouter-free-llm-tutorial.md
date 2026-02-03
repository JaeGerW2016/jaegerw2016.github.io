---
layout: mypost
title: OpenClaw Use OpenRouter free LLM with TelegramBot Tutorial
categories: [OpenClaw OpenRouter]  
---

# 📋 OpenClaw 配合 OpenRouter 免费模型保姆级教程

## 前言

本教程将手把手教你如何配置和使用 OpenClaw 配合 OpenRouter 的免费模型（如 `arcee-ai/trinity-large-preview:free`），实现 Telegram Bot 与 OpenClaw 的聊天交互。

### 目标读者

- 完全没有 OpenClaw 使用经验的小白
- 想快速跑通 Telegram Bot 和 OpenClaw 交互过程的人
- 对 AI 助手配置感兴趣的初学者

### 技术栈

- OpenClaw：开源 AI 助手平台
- OpenRouter：AI 模型聚合平台
- Telegram：消息传递渠道
- 免费模型：OpenRouter 提供的免费大语言模型



# 📋 第一部分：准备工作

## 1.1 系统要求检查



#### 硬件要求

- 内存：建议 8GB 以上（推荐 16GB）
- 存储空间：至少 2GB 可用空间
- CPU：现代 64 位处理器
- 网络：稳定宽带连接

#### 软件要求

- 操作系统：支持 Linux/macOS/Windows
- Node.js：版本 22 或更高
- Git：用于克隆仓库（可选）

#### 网络要求

- 稳定网络连接：用于下载依赖和 API 调用
- 端口访问：需要开放端口 18789（默认网关端口）
- 代理设置：如有需要配置网络代理


## 1.2 OpenClaw安装

### 方法一：使用npm安装（推荐）

```shell
# 安装OpenClaw

npm install -g @qingchencloud/openclaw-zh

# 验证安装

openclaw --version
```

## 方法二：从源码安装

```shell
# 克隆仓库

`git clone https://github.com/qingchencloud/openclaw-zh.git`

### 进入目录

`cd openclaw-zh`

### 安装依赖

`npm install`

### 链接到全局

`npm link`
```

验证安装

```shell
# 运行测试命令

`openclaw help`

# 应该看到帮助信息
```

## 1.3 OpenRouter账号注册

### 访问 OpenRouter 官网

1. 打开浏览器访问：[https://openrouter.ai/](https://openrouter.ai/)
2. 点击右上角 **"Sign Up"** 按钮注册账号
3. 邮箱注册：
   - 输入邮箱地址
   - 设置密码
   - 点击 "Sign Up"
4. 验证邮箱：
   - 检查收件箱
   - 点击验证链接
   - 完成注册

### 获取 API 密钥

1. 登录账号：
   - 访问 https://openrouter.ai/
   - 输入注册的邮箱和密码
   - 点击 "Log In"
2. 获取 API 密钥：
   - 点击右上角头像
   - 选择 "API Keys"
   - 点击 "Create API Key"
   - 复制生成的 API 密钥

### 保存 API 密钥
```bash
# 将 API 密钥保存到环境变量
echo 'export OPENROUTER_API_KEY="your_api_key_here"' >> ~/.bashrc
source ~/.bashrc
```

---

# 🔧 第二部分：配置阶段

## 2.1 OpenClaw 初始配置

### 运行 onboard 向导

```bash
# 启动 onboard 向导
openclaw onboard

# 或者指定部分配置
openclaw onboard --section gateway
```



### onboard 向导步骤

1. **选择运行模式**：

   ```text
   Where will the Gateway run?
   ● Local (this machine) (Gateway reachable (ws://127.0.0.1:18789))
   ○ Remote (info-only)
   ```

2. **设置网关参数**：

   ```text
   gateway.port: 18789
   gateway.bind: loopback
   gateway.mode: local
   ```

3. **设置认证方式**：

   ```text
   auth.mode: token
   auth.token: (自动生成)
   ```

4. **确认配置**：

   ```text
   1. Workspace: ~/.openclaw/workspace
      Model: openrouter/arcee-ai/trinity-large-preview:free
      Gateway.mode: local
      Gateway.port: 18789
   ```

### 验证配置

```bash
# 检查配置状态
openclaw gateway status

# 应该看到类似输出
{
  "status": "running",
  "url": "ws://127.0.0.1:18789",
  "version": "2026.2.1-zh.3"
}
```
   
   

2. 设置网关参数：
   
   ```shell
   gateway.port: 18789
   gateway.bind: loopback
   gateway.mode: local
   ```
   
   

3. 设置认证方式：
   
   ```shell
   auth.mode: token
   auth.token: (自动生成)
   ```
   
   

4. 确认配置：
   
   ```shell
   1. Workspace: ~/.openclaw/workspace
     Model: openrouter/arcee-ai/trinity-large-preview:free
     Gateway.mode: local
     Gateway.port: 18789
     
   
   ```
   
   ## 验证配置

```shell
# 检查配置状态

openclaw gateway status

# 应该看到类似输出

{
 "status": "running",
 "url": "ws://127.0.0.1:18789",
 "version": "2026.2.1-zh.3"
}
```





## 2.2 Telegram Bot 设置

### 创建 Bot

1. **打开 Telegram**：
   - 启动 Telegram 应用
   - 搜索用户 `@BotFather`

2. **创建新 Bot**：
   - 发送命令 `/start`
   - 发送命令 `/newbot`
   - 输入 Bot 名称（如：`MyOpenClawBot`）
   - 输入 Bot 用户名（如：`MyOpenClawBot_bot`）

3. **获取 Bot Token**：
   - BotFather 会返回类似格式的 Token：`123456789:ABCdefGHIjklMNOPQRstUVwxyZ`
   - 复制这个 Token 备用



### 配置 Webhook

```bash
# 设置 Bot Token
openclaw configure --section channels

# 按照提示输入：
# Telegram botToken: 123456789:ABCdefGHIjklMNOPQRstUVwxyZ
# Telegram allowFrom: [your_telegram_id]
```

### 验证 Telegram 配置

```bash
# 测试连接
curl -X POST "https://api.telegram.org/bot<YOUR_TOKEN>/getMe"

# 应该看到 Bot 信息
{
  "ok": true,
  "result": {
    "id": 123456789,
    "is_bot": true,
    "first_name": "MyOpenClawBot"
  }
}
```



### 2.3 OpenRouter 模型配置

#### 选择模型

OpenRouter 提供多个免费模型，推荐选择：

- `arcee-ai/trinity-large-preview:free`（[Trinity Large Preview (free) - API, Providers, Stats | OpenRouter](https://openrouter.ai/arcee-ai/trinity-large-preview:free)）

#### 设置 API 密钥

```bash
# 编辑配置文件
nano ~/.openclaw/config.json

# 添加或修改：
{
  "auth": {
    "profiles": {
      "openrouter:default": {
        "provider": "openrouter",
        "mode": "api_key",
        "apiKey": "your_openrouter_api_key"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openrouter/arcee-ai/trinity-large-preview:free"
      }
    }
  }
  }
}
```



#### 测试模型连接

```bash
# 测试模型连接
openclaw model test openrouter/arcee-ai/trinity-large-preview:free

# 应该看到类似输出
{
  "status": "success",
  "model": "openrouter/arcee-ai/trinity-large-preview:free",
  "response": "Hello! I'm ready to help you."
}
```



# 🚀 第三部分：跑通交互

## 3.1 启动 OpenClaw

### 启动网关

```bash
# 启动 OpenClaw 网关
openclaw gateway start

# 或者后台启动
nohup openclaw gateway start > openclaw.log 2>&1 &

# 检查状态
openclaw gateway status
```



### 连接 Telegram

1. **启动 Telegram**：
   - 打开 Telegram 应用
   - 找到你创建的 Bot（如 `MyOpenClawBot`）

2. **发送测试消息**：
   - 输入任意消息（如 `你好`）
   - 等待 Bot 回复

### 验证连接状态

```bash
# 检查网关日志
tail -f ~/.openclaw/logs/gateway.log

# 应该看到类似日志
[2026-02-03 14:25:00] Gateway connected to Telegram
[2026-02-03 14:25:01] Received message from user 886605066
[2026-02-03 14:25:02] Sent response to Telegram
```



## 3.2 首次对话测试

### 发送测试消息

1. **打开 Telegram 对话**：
   - 点击你的 Bot
   - 在输入框中输入消息

2. **测试消息示例**：

   ```text
   # 简单问候
   你好

   # 测试功能
   你会什么？

   # 测试工具
   现在几点了？
   ```

### 接收回复

1. **等待响应**：
   - 通常在几秒内收到回复
   - 响应时间取决于模型和网络

2. **检查回复质量**：
   - 内容是否相关
   - 格式是否正确
   - 语气是否合适

### 检查响应时间

```text
# 在 Telegram 中，长按消息
# 查看 "发送" 和 "已读" 的时间差
```



### 3.3 基础功能测试

#### 简单问答

```text
# 测试基本对话
你好
今天天气怎么样？
你能帮我写代码吗？
```

#### 工具使用

```text
# 测试内置工具
现在几点了？        # 测试时间工具
搜索一下 AI 新闻   # 测试搜索工具
翻译这句话         # 测试翻译工具
```

#### 消息格式

```text
# 测试不同格式

# 纯文本
你好，很高兴见到你！

# 带格式
**加粗文本**
*斜体文本*
`代码格式`
```

#### 多轮对话

```text
# 测试上下文保持

你：今天的计划是什么？
AI：...
你：那明天的呢？

# AI 应该能理解 "那" 指的是 "计划"
```

#### 错误处理

```text
# 测试错误处理
输入乱码或无意义的内容
测试网络断开的情况
```

# 🛠️ 第四部分：进阶配置

## 4.1 个性化设置

### 设置名称

```bash
# 编辑配置文件
nano ~/.openclaw/config.json

# 添加个性化设置
{
  "agents": {
    "defaults": {
      "name": "MyAssistant",
      "greeting": "Hello! I'm your OpenClaw assistant.",
      "farewell": "Goodbye! Have a great day!"
    }
  }
}
```

### 配置个性

```json
{
  "agents": {
    "defaults": {
      "personality": {
        "tone": "friendly",
        "formality": "casual",
        "humor": "moderate"
      }
    }
  }
}
```

### 调整响应风格

```json
{
  "agents": {
    "defaults": {
      "response": {
        "max_tokens": 1000,
        "temperature": 0.7,
        "top_p": 0.9
      }
    }
  }
}
```

## 4.2 技能系统

### 安装基础技能

```bash
# 安装天气技能
openclaw skills install weather

# 安装 GitHub 技能
openclaw skills install github
```

### 配置技能

```bash
# 查看已安装技能
openclaw skills list

# 配置技能参数
openclaw skills configure weather
```

### 测试技能功能

```text
# 测试天气技能
天气预报
```

```text
# 测试 GitHub 技能
搜索 GitHub 上的 AI 项目
```

## 4.3 多渠道支持

### 添加其他渠道

```bash
# 配置 Discord
openclaw configure --section discord

# 配置 Slack
openclaw configure --section slack
```

### 切换渠道

```bash
# 设置默认渠道
openclaw configure --section channels

# 选择默认渠道
default_channel: telegram
```

### 统一管理

```bash
# 查看所有渠道
openclaw channels list

# 管理消息
openclaw message send --channel telegram --message "Hello"
```

---

# 🔍 第五部分：故障排查

## 5.1 常见问题解决

### 连接问题

```bash
# 检查网络连接
ping openrouter.ai

# 检查网关状态
openclaw gateway status

# 检查 Telegram 连接
curl -X POST "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates"
```

### 响应问题

```bash
# 测试模型连接
openclaw model test openrouter/arcee-ai/trinity-large-preview:free

# 检查 API 配额
openclaw model usage
```

### 配置问题

```bash
# 验证配置文件
openclaw config validate

# 重置配置
openclaw config reset
```

## 5.2 日志查看

### 查看运行日志

```bash
# 查看网关日志
tail -f ~/.openclaw/logs/gateway.log

# 查看模型日志
tail -f ~/.openclaw/logs/model.log

# 查看技能日志
tail -f ~/.openclaw/logs/skills.log
```

### 定位问题

```bash
# 启用调试模式
openclaw gateway start --debug

# 查看详细日志
openclaw logs show --level debug
```

## 5.3 性能优化

### 调整参数

```json
{
  "agents": {
    "defaults": {
      "response": {
        "max_tokens": 500,
        "temperature": 0.5,
        "top_p": 0.8
      }
    }
  }
}
```

### 优化响应

```json
{
  "cache": {
    "enabled": true,
    "max_size": 1000,
    "ttl": 3600
  }
}
```

### 扩展功能

```bash
# 安装性能优化技能
openclaw skills install performance

# 配置优化参数
openclaw skills configure performance
```

---

# 📚 第六部分：资源与扩展

## 6.1 官方文档

### OpenClaw 文档

- 官方网站：https://openclaw.ai/
- 文档中心：https://docs.openclaw.ai/
- GitHub 仓库：https://github.com/qingchencloud/openclaw-zh

### OpenRouter 文档

- 官方网站：https://openrouter.ai/
- API 文档：https://openrouter.ai/docs
- 定价页面：https://openrouter.ai/pricing

### 社区资源

- Discord 社区：https://discord.gg/openclaw
- Reddit 论坛：https://www.reddit.com/r/OpenClaw/
- Stack Overflow：https://stackoverflow.com/questions/tagged/openclaw

## 6.2 社区支持

### 论坛

- GitHub Issues：https://github.com/qingchencloud/openclaw-zh/issues
- 社区论坛：https://community.openclaw.ai/

### Discord 群组

- 官方 Discord：https://discord.gg/openclaw
- 开发频道：#developers
- 用户支持：#help

### GitHub Issues

- 提交 Bug：https://github.com/qingchencloud/openclaw-zh/issues/new
- 请求功能：https://github.com/qingchencloud/openclaw-zh/issues/new?template=feature_request.md

## 6.3 扩展学习

### 进阶教程

- 高级配置：https://docs.openclaw.ai/advanced
- 技能开发：https://docs.openclaw.ai/skills
- 性能调优：https://docs.openclaw.ai/performance

### 最佳实践

- 安全指南：https://docs.openclaw.ai/security
- 备份策略：https://docs.openclaw.ai/backup
- 监控指南：https://docs.openclaw.ai/monitoring

### 开发指南

- API 参考：https://docs.openclaw.ai/api
- 插件开发：https://docs.openclaw.ai/plugins
- 贡献指南：https://docs.openclaw.ai/contributing

---

# 📋 总结

恭喜！你已经成功完成了 OpenClaw 配合 OpenRouter 免费模型的保姆级教程。现在你应该能够：

- ✅ 安装和配置 OpenClaw
- ✅ 设置 Telegram Bot
- ✅ 配置 OpenRouter 模型
- ✅ 跑通聊天交互
- ✅ 进行个性化设置
- ✅ 安装和使用技能
- ✅ 进行故障排查和优化

如果你遇到任何问题，请参考故障排查部分或联系社区支持。祝你使用 OpenClaw 愉快！

---

# 📚 附录

## 附录 A：命令速查表

| 命令                      | 描述         |
|-------------------------|------------|
| `openclaw --version`    | 查看版本      |
| `openclaw help`         | 查看帮助      |
| `openclaw onboard`      | 运行配置向导    |
| `openclaw gateway start`| 启动网关      |
| `openclaw gateway status` | 检查网关状态   |
| `openclaw skills list`  | 列出已安装技能  |
| `openclaw channels list` | 列出可用渠道   |

## 附录 B：配置参考

```json
{
  "auth": {
    "profiles": {
      "openrouter:default": {
        "provider": "openrouter",
        "mode": "api_key",
        "apiKey": "your_api_key"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openrouter/arcee-ai/trinity-large-preview:free"
      },
      "name": "MyAssistant",
      "greeting": "Hello!",
      "personality": {
        "tone": "friendly"
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "your_bot_token",
      "allowFrom": ["your_user_id"]
    }
  }
}
```
