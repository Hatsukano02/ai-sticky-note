# AI 便利贴

> 便携式智能语音交互墨水屏设备

![概念图](docs/assets/concept_img.png)

## 项目简介

AI 便利贴是一款便携式智能显示设备，通过磁吸或支架方式灵活放置于冰箱、桌面等场景。它采用**电子墨水屏**实现常显不耗电，支持**语音交互**简化操作，具备**2-4 周超长续航**，让你无需打开手机即可查看待办、天气、日程等生活信息。

**核心价值**：低侵入感、无感自然、语音驱动、长期续航

---

## 核心功能

| 页面 | 功能描述 | 语音控制示例 |
|------|----------|-------------|
| 📅 **日期** | 大号日期显示，支持农历、节日提醒 | "显示农历"、"切换到节日模板" |
| 🌤️ **天气** | 当前温度、天气状况、空气质量 | "显示未来三天预报"、"切换到极简模式" |
| ✅ **待办** | 语音添加待办清单，支持优先级 | "添加待办买牛奶"、"完成第二个待办" |
| 📆 **日历** | 月历/周历/日视图，支持日程提醒 | "切换到周视图"、"查看下个月的日程" |
| ⏰ **倒计时** | 考试、纪念日等重要日子倒数 | "设置高考倒计时 86 天" |
| 🌾 **节气** | 当前节气展示，附带意象图标 | - |
| 🔋 **电量** | 电量百分比、预估续航天数 | - |
| 🔄 **同步** | 同步状态、网络连接状态 | "立即同步"、"重置网络" |

**扩展能力**：Agent 可根据用户需求**动态创建自定义页面**（如股票、空气质量等）

---

## 硬件规格

| 组件 | 规格 |
|------|------|
| **主控** | ESP32-S3（WiFi + 蓝牙） |
| **显示屏** | 4.2 寸电子墨水屏，400×300 分辨率，黑白显示 |
| **音频输入** | I2S 数字麦克风 |
| **音频输出** | I2S 扬声器 |
| **交互方式** | 单按钮（单击/双击/长按） |
| **供电** | 2000mAh 锂电池，Type-C 充电 |
| **续航** | 正常使用 2-4 周，深度休眠可达 3 个月 |
| **安装方式** | 磁吸（冰箱）+ 支架（桌面） |
| **外壳尺寸** | 96 × 100 × 13 mm |

### 按钮操作

- **单击** → 切换到下一页
- **双击** → 切换到上一页  
- **长按**（>800ms）→ 进入语音输入模式

---

## 系统架构

AI 便利贴采用**五层分层架构**，实现云端智能与设备端的解耦：

```
┌─────────────────────────────────────────┐
│  Layer 0: Web 管理界面                   │
│  • 用户/设备管理 • 配对管理 • 可视化配置   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  Layer 1: 后端服务层                     │
│  • 设备管理 • 配对服务 • 同步服务 • ASR   │
│  • PostgreSQL + Redis 数据持久化         │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  Layer 2: Agent 层                       │
│  • Pydantic AI Agent • 语音意图解析      │
│  • LLM 路由（豆包/千问/Kimi/DeepSeek）    │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  Layer 3: 管理层（MCP Tools）            │
│  • 8 个页面 Tools（todo/weather/date...）│
│  • PageManager（创建/隐藏/切换页面）     │
│  • Page Registry（用户级页面配置）       │
└─────────────────────────────────────────┘
                    ↓ MQTT
┌─────────────────────────────────────────┐
│  Layer 4: 渲染层（ESP32 端）              │
│  • 布局引擎 • 组件库 • 墨水屏驱动         │
│  • RTC 休眠管理 • 电源优化                │
└─────────────────────────────────────────┘
```

### 关键特性

- **多租户隔离**：用户级 Registry，同一用户的所有设备共享配置
- **动态页面**：Agent 可根据语音指令创建自定义页面
- **离线模式**：本地缓存，断网后仍可查看已同步数据
- **RTC 休眠**：白天 30 分钟/夜间 60 分钟定时唤醒，最大化续航

---

## 项目结构

```
ai-sticky-note/
├── docs/                          # 项目文档
│   ├── 01-product/               # 产品文档
│   │   ├── PRD.md                # 产品需求文档
│   │   └── roadmap.md            # 开发路线图
│   ├── 02-architecture/          # 架构设计
│   │   ├── system-architecture.md    # 服务端五层架构
│   │   ├── hardware-architecture.md  # 硬件架构
│   │   └── database-schema.md        # 数据库设计
│   ├── 03-protocols/             # 通信协议
│   │   ├── pairing-protocol.md   # 设备配对协议
│   │   ├── sync-protocol.md      # 数据同步协议
│   │   └── rendering-protocol.md # 渲染协议
│   ├── 04-modules/               # 模块设计
│   │   ├── ui-display.md         # UI 显示模块
│   │   ├── state-machine.md      # 状态机设计
│   │   └── weather.md            # 天气模块
│   ├── 05-hardware/              # 硬件规格
│   │   ├── case-dimensions.md    # 外壳尺寸规格
│   │   ├── interaction-input.md  # 输入交互
│   │   └── interaction-output.md # 输出交互
│   ├── 06-interface/             # 接口规范
│   │   └── mcp-tools.md          # MCP Tools 定义
│   └── assets/                   # 图片资源
├── AGENTS.md                      # AI 代理配置
└── README.md                      # 本文件
```

---

## 开发路线图

| 阶段 | 目标 | 周期 | 关键产出 |
|------|------|------|---------|
| **M1** | 硬件验证 | 2 周 | 墨水屏点亮，基础驱动正常 |
| **M2** | 核心软件 | 4 周 | 8 个页面可切换，WiFi 连接稳定 |
| **M3** | 服务器与 MCP | 3 周 | 语音交互可用，8 个 Tools 完整 |
| **M3.5** | 设备配对 | 2 周 | 设备配对可用，多设备同步正常 |
| **M4** | 系统集成 | 3 周 | 系统稳定运行，离线模式可用 |
| **M4.5** | Web 管理 | 2 周 | WebUI 可用，支持远程管理 |
| **M5** | 优化完善 | 3 周 | 续航达标，可以日常使用 |

**当前状态**：M1 硬件验证阶段，已完成外壳尺寸设计

---

## 技术栈

### 云端服务

| 类别 | 技术选型 |
|------|----------|
| Web 框架 | FastAPI + WebSocket |
| Agent 框架 | Pydantic AI |
| MCP 协议 | Model Context Protocol |
| ASR | 豆包流式语音识别 |
| LLM | 千问 / 豆包 / Kimi / DeepSeek（可切换） |
| 数据库 | PostgreSQL 16 + Redis 7 |
| MQTT | EMQX / Mosquitto |
| 部署 | Docker + Docker Compose |

### 设备端（ESP32）

| 类别 | 技术选型 |
|------|----------|
| 开发框架 | Arduino / ESP-IDF |
| 墨水屏驱动 | GxEPD2 |
| JSON 处理 | ArduinoJson |
| WebSocket | links2004/WebSockets |
| MQTT | PubSubClient |
| UI 框架 | Paperd.Ink（扩展） |

---

## 快速开始

### 云端部署

```bash
# 1. 克隆仓库
git clone https://github.com/Hatsukano02/ai-sticky-note.git
cd ai-sticky-note

# 2. 配置环境变量
cp server/.env.example server/.env
# 编辑 .env，填入 API Keys

# 3. 启动服务
docker-compose up -d

# 4. 访问服务
# WebUI: http://localhost:8000/admin
# API: http://localhost:8000/api
```

### 设备开发

```bash
# 1. 安装 PlatformIO
pip install platformio

# 2. 打开固件项目
cd firmware

# 3. 编译并上传
pio run --target upload --environment esp32-s3

# 4. 查看串口日志
pio device monitor
```

---

## 文档索引

| 主题 | 文档 |
|------|------|
| 产品需求 | [docs/01-product/PRD.md](docs/01-product/PRD.md) |
| 开发路线图 | [docs/01-product/roadmap.md](docs/01-product/roadmap.md) |
| 系统架构 | [docs/02-architecture/system-architecture.md](docs/02-architecture/system-architecture.md) |
| 硬件架构 | [docs/02-architecture/hardware-architecture.md](docs/02-architecture/hardware-architecture.md) |
| 数据库设计 | [docs/02-architecture/database-schema.md](docs/02-architecture/database-schema.md) |
| 外壳尺寸 | [docs/05-hardware/case-dimensions.md](docs/05-hardware/case-dimensions.md) |
| MCP Tools | [docs/06-interface/mcp-tools.md](docs/06-interface/mcp-tools.md) |

---

## 贡献

这是一个个人兴趣项目，开发节奏：

> **有空就搞，没空就停。开心最重要。**

欢迎提交 Issue 或 PR，但请理解响应时间可能较长。

---

## License

MIT License © 2026 Hatsukano Nagi
