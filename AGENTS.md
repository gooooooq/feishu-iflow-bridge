# 飞书 iFlow 桥接服务 - 项目上下文文档

## 项目概述

这是一个通过飞书消息调用 iFlow CLI 执行任务的桥接服务。服务使用 WebSocket 长连接实时接收飞书消息，调用 iFlow CLI 执行用户指定的命令，并支持多种格式文件的自动回传。

### 主要技术栈
- **Node.js** >= 16.0.0
- **@larksuiteoapi/node-sdk** v1.35.0 - 飞书官方 SDK
- **WebSocket** - 长连接通信
- **Winston** - 日志管理

### 核心功能
- 📨 通过 WebSocket 实时接收飞书消息
- 🤖 调用 iFlow CLI 执行用户命令
- 📁 支持多种格式文件自动回传（图片、音频、视频、文档）
- 📊 实时监控任务执行进度
- 💬 支持富文本消息发送

---

## 项目结构

```
C:\Users\39777\.iflow\feishu-iflow-bridge-main\
├── src/
│   ├── index.js                 # 主入口文件
│   ├── modules/                 # 核心模块
│   │   ├── WebSocketManager.js  # WebSocket 长连接管理
│   │   ├── EventHandler.js      # 飞书事件处理
│   │   ├── FeishuSender.js      # 飞书消息发送
│   │   ├── IFlowAdapter.js      # iFlow CLI 适配器
│   │   ├── ProgressManager.js   # 进度监控
│   │   ├── ResultAnalyzer.js    # 结果分析
│   │   └── SessionManager.js    # 会话管理
│   └── utils/                   # 工具函数
│       ├── logger.js            # 日志工具
│       └── sessionIdGenerator.js # 会话 ID 生成器
├── config/
│   ├── config.js                # 用户配置文件（需手动创建）
│   └── default.js               # 默认配置
├── data/
│   └── sessions/                # 会话数据存储目录
├── logs/                        # 日志目录
├── docs/                        # 项目文档
├── Dockerfile                   # Docker 构建配置
├── package.json                 # 项目依赖
└── README.md                    # 项目说明
```

---

## 构建和运行

### 环境要求
- Node.js >= 16.0.0
- iFlow CLI（需要预先安装）

### 安装依赖
```bash
npm install
```

### 配置环境变量

创建 `config/config.js` 文件并配置以下参数：

```javascript
module.exports = {
  feishu: {
    appId: 'your-app-id',           // 飞书应用 ID
    appSecret: 'your-app-secret',   // 飞书应用密钥
    encryptKey: process.env.FEISHU_ENCRYPT_KEY,
    verificationToken: process.env.FEISHU_VERIFICATION_TOKEN,
    hubbleUrl: 'wss://open.feishu.cn/open-apis/hubble-im/v1'
  },
  iflow: {
    cliPath: process.env.IFLOW_CLI_PATH || 'iflow',
    superpowersEnabled: process.env.SUPERPOWERS_ENABLED !== 'false',
    superpowersMode: process.env.SUPERPOWERS_MODE || 'yolo'
  },
  execution: {
    yoloMode: process.env.YOLO_MODE !== 'false',
    autoConfirm: process.env.AUTO_CONFIRM !== 'false',
    maxLoopDepth: parseInt(process.env.MAX_LOOP_DEPTH || '100'),
    timeoutPerStep: parseInt(process.env.TIMEOUT_PER_STEP || '300')
  },
  progress: {
    interval: parseInt(process.env.PROGRESS_INTERVAL || '180'),
    enabled: process.env.PROGRESS_ENABLED !== 'false'
  },
  session: {
    timeout: parseInt(process.env.SESSION_TIMEOUT || '3600'),
    dir: process.env.SESSION_DIR || './data/sessions'
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    dir: process.env.LOG_DIR || './logs'
  }
};
```

### 启动服务

```bash
# 直接启动
npm start

# 开发模式（使用 nodemon）
npm run dev
```

### 测试 SDK
```bash
node test_sdk.js
```

---

## 开发约定

### 代码风格
- 使用 ES6+ 语法
- 使用 async/await 处理异步操作
- 使用 Winston 日志库记录日志
- 模块化设计，单一职责原则

### 错误处理
- 所有异步操作必须包含 try-catch
- 使用 logger 记录错误信息
- 不要忽略 Promise 拒绝

### 会话管理
- 每个会话使用 `chatId` + `senderId` 组合生成唯一 sessionId
- 会话数据以 `.md` 文件形式存储在 `data/sessions/` 目录
- 会话默认超时时间为 3600 秒（1小时）

### 消息处理流程
1. WebSocketManager 接收飞书消息事件
2. EventHandler 解析消息内容
3. SessionManager 创建或获取会话
4. IFlowAdapter 调用 iFlow CLI 执行命令
5. ResultAnalyzer 分析执行结果
6. ProgressManager 监控进度（如需要）
7. FeishuSender 发送结果回飞书

---

## 核心模块说明

### WebSocketManager (`src/modules/WebSocketManager.js`)
管理与飞书 Hubble WebSocket 的连接。使用飞书官方 SDK 的 WSClient 和 EventDispatcher 进行事件分发。

### EventHandler (`src/modules/EventHandler.js`)
处理飞书消息事件，提取消息内容和用户信息，路由到消息处理器。支持文本消息处理，忽略非文本消息。

### IFlowAdapter (`src/modules/IFlowAdapter.js`)
封装 iFlow CLI 的调用。支持：
- `executeSkill(skillInput, sessionId)` - 执行 Skill 调用
- `execute(input, options)` - 执行普通命令
- `executeWithRetry(input, options, maxRetries)` - 带重试的执行
- `isAvailable()` - 检查 iFlow CLI 是否可用
- `getVersion()` - 获取 iFlow CLI 版本

### FeishuSender (`src/modules/FeishuSender.js`)
发送消息到飞书。支持：
- 文本消息
- 图片消息（自动上传并发送）
- 音频消息
- 视频消息
- 文件消息（文档等）
- 富文本卡片消息

自动识别输出中的文件路径并上传对应类型的消息。

### SessionManager (`src/modules/SessionManager.js`)
管理会话生命周期。会话数据以 Markdown 文件形式存储，包含执行历史、状态等信息。

### ProgressManager (`src/modules/ProgressManager.js`)
定时生成进度摘要并发送到飞书。默认间隔 180 秒。

### ResultAnalyzer (`src/modules/ResultAnalyzer.js`)
分析 iFlow CLI 执行结果，识别执行状态、提取下一阶段目标。

---

## 支持的文件类型

### 图片
`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg`, `.bmp`

### 音频
`.mp3`, `.wav`, `.aac`, `.ogg`, `.flac`, `.m4a`

### 视频
`.mp4`, `.avi`, `.mov`, `.mkv`, `.flv`, `.webm`

### 文档
`.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.pdf`, `.txt`, `.csv`, `.md`

---

## 配置说明

### 飞书配置
- `appId` - 飞书应用 ID
- `appSecret` - 飞书应用密钥
- `encryptKey` - 加密密钥（可选）
- `verificationToken` - 验证令牌（可选）

### iFlow 配置
- `cliPath` - iFlow CLI 路径，默认为 `iflow`
- `superpowersEnabled` - 是否启用 superpowers 技能，默认 true
- `superpowersMode` - superpowers 模式，默认 'yolo'

### 执行配置
- `yoloMode` - YOLO 模式，自动确认所有操作，默认 true
- `autoConfirm` - 自动确认，默认 true
- `maxLoopDepth` - 最大循环深度，默认 100
- `timeoutPerStep` - 每步超时时间（秒），默认 300

### 进度监控配置
- `interval` - 进度汇报间隔（秒），默认 180
- `enabled` - 是否启用进度监控，默认 true

### 会话配置
- `timeout` - 会话超时时间（秒），默认 3600
- `dir` - 会话数据存储目录，默认 './data/sessions'

### 日志配置
- `level` - 日志级别，默认 'info'
- `dir` - 日志存储目录，默认 './logs'

---

## Docker 支持

```bash
# 构建镜像
docker build -t feishu-iflow-bridge .

# 运行容器
docker run -d --name feishu-iflow-bridge \
  -v $(pwd)/config:/app/config \
  -e APP_ID=your-app-id \
  -e APP_SECRET=your-app-secret \
  feishu-iflow-bridge
```

---

## 相关文档

- [接口设计](docs/接口设计.md)
- [数据存储设计](docs/数据存储设计.md)
- [核心模块设计](docs/核心模块设计.md)
- [系统架构设计](docs/系统架构设计.md)