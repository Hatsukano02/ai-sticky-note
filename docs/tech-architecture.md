# 技术架构文档

## 1. 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        用户                                   │
│                    （语音/按钮）                              │
└───────────────────┬─────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────┐
│                  ESP32 设备端                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   状态机     │  │   页面渲染   │  │   MCP客户端  │       │
│  │  StateMachine│  │  UIRenderer  │  │  MCP Client  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐       │
│  │  按钮驱动    │  │  墨水屏驱动  │  │  WebSocket   │       │
│  │  Button      │  │  E-Paper     │  │  Client      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                 │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐       │
│  │  麦克风(I2S) │  │  扬声器(I2S) │  │  WiFi        │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                    │ WebSocket
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    服务器端 (Skill Host)                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              MCP Server (FastMCP)                    │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │ DateSkill  │  │WeatherSkill│  │ TodoSkill  │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘    │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │CalendarSkill│ │Countdown   │  │SolarTerm   │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘    │   │
│  │  ┌────────────┐  ┌────────────┐                    │   │
│  │  │BatterySkill│  │ SyncSkill  │                    │   │
│  │  └────────────┘  └────────────┘                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────┐          │
│  │          LLM Router                          │          │
│  │  ┌────────┐  ┌────────┐  ┌────────┐         │          │
│  │  │ 豆包   │  │ 千问   │  │Claude  │         │          │
│  │  └────────┘  └────────┘  └────────┘         │          │
│  └─────────────────────────────────────────────┘          │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────┐          │
│  │          数据存储                             │          │
│  │  SQLite (用户数据) + Redis (缓存)            │          │
│  └─────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

## 2. ESP32端架构

### 2.1 核心模块

| 模块 | 职责 | 文件 |
|------|------|------|
| StateMachine | 管理设备状态、页面切换 | `state_machine.cpp` |
| UIRenderer | 统一渲染接口、模板系统 | `renderer.cpp` |
| MCPClient | MCP协议客户端、消息序列化 | `mcp_client.cpp` |
| WebSocketClient | WebSocket连接管理、心跳 | `websocket_client.cpp` |
| ButtonHandler | 按钮中断、单击/双击/长按识别 | `button.cpp` |
| AudioManager | 音频采集/播放管理 | `audio_manager.cpp` |
| StorageManager | SPIFFS文件系统操作 | `storage.cpp` |

### 2.2 任务分配

| 任务 | 优先级 | Core | 说明 |
|------|--------|------|------|
| main_task | 5 | 0 | 主循环、UI渲染 |
| button_isr | 10 | 0 | 按钮中断（最高） |
| network_task | 6 | 1 | 网络通信、WebSocket |
| audio_task | 7 | 1 | 音频采集/播放 |
| sync_timer | - | - | 15分钟定时同步 |

### 2.3 状态机定义

```cpp
enum DeviceState {
  STATE_IDLE,           // 待机，监听按钮
  STATE_PAGE_DISPLAY,   // 正常显示页面
  STATE_VOICE_INPUT,    // 长按中，录音
  STATE_VOICE_UPLOAD,   // 上传音频
  STATE_VOICE_PROCESS,  // 等待MCP响应
  STATE_SYNCING,        // 同步数据中
  STATE_ERROR           // 错误状态
};

struct StateContext {
  DeviceState current_state;
  PageType current_page;
  TemplateType current_template;
  uint32_t state_enter_time;
  void* state_data;
};
```

### 2.4 本地存储结构

```
/spiffs/
├── config/
│   ├── device.json       # 设备配置
│   ├── wifi.json         # WiFi凭证
│   └── user.json         # 用户偏好
├── cache/
│   ├── weather.json      # 天气缓存
│   ├── current_page.json # 当前页面状态
│   └── sync_status.json  # 同步状态
├── data/
│   ├── todos.json        # 待办列表
│   ├── countdowns.json   # 倒计时
│   └── events.json       # 日程事件
└── logs/
    └── system.log        # 运行日志
```

## 3. 服务器端架构

### 3.1 MCP Server设计

```python
class MCPServer:
    def __init__(self):
        self.skills = {}  # Skill注册表
        self.sessions = {}  # WebSocket会话
        self.llm_router = LLMRouter()
        
    async def handle_message(self, session_id, message):
        skill = self.skills.get(message.skill)
        tool = skill.tools.get(message.tool)
        result = await tool.execute(message.params)
        return result
```

### 3.2 Skill注册机制

```python
# skills/date_skill/manifest.json
{
  "skill_id": "date_skill",
  "entry_point": "date_skill:DateSkill",
  "tools": ["switch_template", "toggle_lunar"],
  "templates": ["standard", "minimal", "festival"]
}

# 自动加载
skill_manager.load_skill("skills/date_skill")
```

### 3.3 LLM Router

```python
class LLMRouter:
    def __init__(self):
        self.providers = {
            "doubao": DoubaoClient(),
            "qwen": QwenClient(),
            "claude": ClaudeClient()
        }
        self.default = "doubao"
    
    async def chat(self, messages, tools=None, provider=None):
        client = self.providers.get(provider, self.default)
        return await client.chat(messages, tools)
```

### 3.4 数据库Schema

```sql
-- 设备表
CREATE TABLE devices (
    id TEXT PRIMARY KEY,
    user_id TEXT,
    last_online TIMESTAMP,
    battery_percent INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 待办表
CREATE TABLE todos (
    id TEXT PRIMARY KEY,
    device_id TEXT,
    content TEXT,
    completed BOOLEAN DEFAULT FALSE,
    priority INTEGER,
    due_date DATE,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 倒计时表
CREATE TABLE countdowns (
    id TEXT PRIMARY KEY,
    device_id TEXT,
    name TEXT,
    target_date DATE,
    display_style TEXT,
    reminder_days TEXT, -- JSON array
    created_at TIMESTAMP
);

-- 同步日志
CREATE TABLE sync_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id TEXT,
    sync_type TEXT,
    status TEXT,
    synced_at TIMESTAMP
);
```

## 4. 通信协议

### 4.1 WebSocket消息类型

| 类型 | 方向 | 说明 |
|------|------|------|
| `auth` | C→S | 设备认证 |
| `auth_response` | S→C | 认证响应 |
| `heartbeat` | C→S | 心跳包 |
| `heartbeat_ack` | S→C | 心跳确认 |
| `tool_call` | C→S | 调用Skill Tool |
| `tool_result` | S→C | Tool执行结果 |
| `sync_request` | C→S | 请求同步数据 |
| `sync_data` | S→C | 同步数据 |
| `notification` | S→C | 服务器推送 |

### 4.2 认证流程

```
ESP32                          Server
  │                              │
  │ ───────── auth ─────────────▶│
  │ {device_id, token}           │
  │                              │
  │ ◀──────── auth_response ────│
  │ {success, session_id}        │
  │                              │
  │ ◀──── heartbeat (15min) ────│
```

### 4.3 语音处理流程

```
ESP32                          Server
  │                              │
  │ ─────── audio_stream ──────▶│
  │ (VAD检测到语音后上传)         │
  │                              │
  │ ◀────── asr_result ─────────│
  │ {text, confidence}           │
  │                              │
  │ ◀────── tool_call ──────────│
  │ {skill, tool, params}        │
  │                              │
  │ ─────── tool_result ───────▶│
  │ {success, data}              │
  │                              │
  │ ◀────── display_update ─────│
  │ {page, template, data}       │
```

## 5. 数据流设计

### 5.1 页面切换流程

```
[按钮触发] → [StateMachine] → [确定目标页面] 
    → [StorageManager读取本地缓存] 
    → [UIRenderer渲染] 
    → [E-Paper刷新]
```

### 5.2 语音操作流程

```
[长按按钮] → [VAD启动] → [录音] → [松手] 
    → [WebSocket上传] → [ASR识别] 
    → [Intent解析] → [Tool调用] 
    → [执行结果] → [更新显示]
```

### 5.3 同步流程

```
[定时触发/手动触发] → [检查本地dirty标记] 
    → [上传变更] → [服务器合并] 
    → [下载服务器更新] → [更新本地缓存]
    → [清除dirty标记] → [刷新显示]
```

## 6. 安全设计

### 6.1 设备认证
- 预烧录Device ID和Token
- WebSocket连接时携带Token认证
- Token可定期轮换

### 6.2 数据传输
- WebSocket over TLS (WSS)
- MCP消息签名验证

### 6.3 隐私保护
- 语音数据不存储，处理完立即删除
- 用户数据加密存储
- 支持数据导出/删除

## 7. 扩展性设计

### 7.1 Skill热插拔
- 运行时动态加载Skill
- Skill间隔离，互不影响
- 支持用户自定义Skill

### 7.2 多设备支持
- 一个用户可绑定多个设备
- 设备间数据同步
- 支持家庭共享

### 7.3 开放API
- 提供HTTP API供第三方集成
- Webhook支持实时通知
- 支持Home Assistant等智能家居平台
