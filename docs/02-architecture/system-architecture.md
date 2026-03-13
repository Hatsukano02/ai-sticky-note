---
title: 服务端架构规范
description: 五层架构设计：WebUI、后端服务、Agent层、管理层、渲染层，含RTC休眠策略
type: architecture
scope: [backend, device]
created: "2026-03-13"
updated: "2026-03-13"
dependencies:
  - database-schema.md
  - hardware-architecture.md
related:
  - ../03-protocols/pairing-protocol.md
  - ../03-protocols/sync-protocol.md
  - ../03-protocols/rendering-protocol.md
  - ../06-interface/mcp-tools.md
---

# 服务端架构规范

## 1. 五层架构设计

### 1.1 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 0: Web管理界面                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ • 管理员登录                                                      │ │
│  │ • 用户管理（创建/编辑用户）                                       │ │
│  │ • 设备管理（查看/分配/远程控制）                                  │ │
│  │ • 配对管理（查看待配对设备，确认绑定）                            │ │
│  │ • Registry可视化编辑器                                            │ │
│  │ • 系统日志与监控                                                  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                     ↓ HTTPS REST/WebSocket
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 1: 后端层 (Backend)                                               │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Auth Service                                              │ │
│  │ • JWT认证（管理员账号）                                           │ │
│  │ • 用户CRUD管理                                                    │ │
│  │                                                                   │ │
│  │ Pairing Service                                           │ │
│  │ • 首次启动配对处理（AP模式）                                      │ │
│  │ • 配对码生成/验证（Redis TTL 10min）                              │ │
│  │ • 设备绑定到用户                                                  │ │
│  │                                                                   │ │
│  │ Device Manager                                            │ │
│  │ • 设备注册表：device_id → user_id                                 │ │
│  │ • 连接池管理：{device_id: websocket_conn}                         │ │
│  │ • 多租户隔离：用户只能访问自己的设备                              │ │
│  │ • 状态追踪：设备在线/离线/休眠状态                              │ │
│  │ • 消息路由：按device_id分发到指定设备                             │ │
│  │                                                                   │ │
│  │ Sync Service                                              │ │
│  │ • 用户级Registry同步（同一用户的所有设备镜像）                    │ │
│  │ • 变更检测 → 推送在线设备                                         │ │
│  │ • 离线设备上线时拉取                                              │ │
│  │ • 小时级全量同步（兜底）                                          │ │
│  │                                                                   │ │
│  │ WebSocket Gateway                                         │ │
│  │ • 音频流传输（ASR识别）                                           │ │
│  │ • 认证：device_id + token                                         │ │
│  │                                                                   │ │
│  │ 豆包 ASR（流式语音识别）                                          │ │
│  │                                                                   │ │
│  │ Data Persistence（PostgreSQL + Redis）                            │ │
│  │ • 用户数据隔离                                                    │ │
│  │ • Registry按用户存储                                              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                     ↓ 音频/指令
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 2: Agent 层                                                       │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Pydantic AI Agent                                                 │ │
│  │ • 语音意图解析（中文自然语言 → 结构化指令）                        │ │
│  │ • 单模型推理（千问/豆包/Kimi/DeepSeek，部署时固定）                │ │
│  │ • 工具调度（调用 MCP Tools）                                       │ │
│  │ • 上下文：绑定到 user_id 的 Registry                               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                     ↓ 调用 Tools
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 3: 管理层 (Management)                                            │
│  ┌───────────────────────────┐    ┌─────────────────────────────────┐ │
│  │ MCP Tools                 │    │ Page Registry（页面注册表）       │ │
│  │                           │    │                                 │ │
│  │ • 原8个页面 Tools         │    │ • 用户级Registry（按user隔离）   │ │
│  │   - todo/weather/date...  │    │ • 同一用户的所有设备共享         │ │
│  │                           │    │ • 内置页面（8个，可显隐）         │ │
│  │ • PageManager             │    │ • 自定义页面（Agent 生成）        │ │
│  │   - create_page()         │    │ • 布局模板库（8个预置模板）       │ │
│  │   - hide_page()           │    │ • 页面状态（visible/order）       │ │
│  │   - list_pages()          │    │                                 │ │
│  │   - switch_layout()       │    │ JSON Schema 格式：               │ │
│  │                           │    │ {                                │ │
│  │ • DeviceManager   │    │   "page_id": "weather",         │ │
│  │   - list_user_devices()   │    │   "visible": true,              │ │
│  │   - get_device_status()   │    │   "template": "standard",       │ │
│  │   - remote_command()      │    │   "elements": [...]             │ │
│  │                           │    │ }                                │ │
│  └───────────────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                     ↓ MQTT推送
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 4: 渲染层 (Rendering)                                             │
│  ESP32 动态渲染引擎（基于 Paperd.Ink 扩展）                              │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ • 首次启动检测（SPIFFS无Registry → AP模式）                       │ │
│  │ • 配对模式：显示6位码 + QR Code                                   │ │
│  │ • 组合键：双击+长按 → 重置并重启                                  │ │
│  │                                                                   │ │
│  │ • Layout Engine（布局引擎）                                       │ │
│  │   - 解析 JSON Schema                                              │ │
│  │   - 计算元素位置（自动布局）                                       │ │
│  │   - 支持预置模板（15种）                                          │ │
│  │                                                                   │ │
│  │ • Component Library（组件库）                                     │ │
│  │   - Text（文字，支持4种字号）                                     │ │
│  │   - Icon（图标，预置100个）                                       │ │
│  │   - Line/Rect（线条/矩形）                                        │ │
│  │   - QrCode（二维码，算法生成）                                    │ │
│  │                                                                   │ │
│  │ • GxEPD2 Driver（墨水屏驱动）                                     │ │
│  │   - 400x300 分辨率                                                │ │
│  │   - 局部刷新优化                                                  │ │
│  │                                                                   │ │
│  │ • 电源管理                                                        │ │
│  │   - RTC定时唤醒：白天(8:00-22:00)30分钟，夜间1小时               │ │
│  │   - 深度休眠：WiFi断开，RTC定时器运行，按键中断监听              │ │
│  │   - 活跃期：语音操作后维持5分钟不进入休眠                        │ │
│  │                                                                   │ │
│  │ • 网络管理                                                        │ │
│  │   - 唤醒后连接WiFi检查MQTT消息                                    │ │
│  │   - 按键触发立即连接并同步                                        │ │
│  │   - 消息订阅：registry/{device_id}                                │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                     ↓ 绘制指令
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 5: UI 层                                                          │
│  • 4.2寸 400x300 电子墨水屏                                            │
│  • 黑白显示（灰度通过抖动算法模拟）                                    │
│  • 刷新时间：1.5-3秒（全屏）/ 0.3秒（局部）                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 多租户数据隔离模型

```
User A (uuid-a)
├── Registry A (共享配置)
│   ├── Pages: [date, weather, todo, stock_simple]
│   └── Templates: [standard, center_large, ...]
│
├── Device 1 (AA:BB:CC:01) - 在线
│   ├── Connection: WebSocket conn-001
│   ├── Status: online
│   └── Last Sync: 2026-03-13T12:00:00Z
│
├── Device 2 (AA:BB:CC:02) - 离线
│   ├── Connection: null
│   ├── Status: offline
│   └── Last Sync: 2026-03-13T10:00:00Z
│
└── Device 3 (AA:BB:CC:03) - 在线
    ├── Connection: WebSocket conn-003
    ├── Status: online
    └── Last Sync: 2026-03-13T12:00:00Z

User B (uuid-b) - 完全隔离
├── Registry B (独立配置)
│   └── Pages: [date, todo, countdown]
│
└── Device 4 (AA:BB:CC:04) - 在线
    └── Connection: WebSocket conn-004

数据隔离保证:
- User A 无法访问 User B 的任何数据
- 设备只能访问所属用户的 Registry
- 管理员可查看所有用户和设备（用于管理）
```

## 2. 核心数据流

### 2.1 语音控制现有页面

```
用户长按 → ESP32录音 → WebSocket上传 → 豆包ASR识别
                                               ↓
                                        Pydantic AI Agent
                                               ↓
                              调用 todo.add_todo(content="买牛奶")
                                               ↓
                                        更新Registry
                                               ↓
                              MQTT推送 → ESP32渲染待办页面
```

### 2.2 Agent创建自定义页面（关键新功能）

```
用户语音:"添加一个股票页面，只显示涨跌幅"
                    ↓
            豆包ASR识别文本
                    ↓
    Agent理解意图 → 调用 page_manager.create_page()
                    ↓
    LLM生成JSON Schema（符合400x300约束）：
    {
      "page_id": "stock_simple",
      "template": "center_large",
      "elements": [
        {"type": "text", "field": "change_percent", "font": "xlarge", "align": "center"}
      ]
    }
                    ↓
    保存到Page Registry → 标记为新页面
                    ↓
    MQTT推送配置 → ESP32动态渲染
                    ↓
    语音反馈:"股票页面已添加"
```

### 2.3 页面显隐管理

```
用户语音:"隐藏天气和节气页面"
                    ↓
    Agent调用 page_manager.hide_pages(["weather", "solar_term"])
                    ↓
    Registry更新 visible=false
                    ↓
    Sync Service检测到Registry变更
                    ↓
    推送到该用户的所有在线设备
                    ↓
    MQTT推送 → ESP32从页面循环中移除
                    ↓
    用户单击/双击不再显示这些页面
```

### 2.4 设备首次配对流程

```
[设备首次上电]
    ↓
[检查SPIFFS：无WiFi配置？]
    ↓ 是
[进入AP模式：发射WiFi "AI-Sticky-XXXX"]
    ↓
[屏幕显示配置指引]
    ↓
                    [用户手机连接AI-Sticky-XXXX]
                              ↓
                    [访问192.168.4.1]
                              ↓
                    [提交家庭WiFi配置]
                              ↓
                    [设备连接家庭WiFi成功]
                              ↓
[设备生成配对码，通过MQTT上报后端]
    ↓
                    [管理员在WebUI查看待配对设备列表]
                              ↓
                    [点击"确认绑定"，分配给用户]
                              ↓
[后端下发：配对成功 + Registry]
    ↓
[设备保存配置，退出AP模式]
    ↓
[进入正常模式，开始同步]
```

### 2.5 多设备同步流程

**场景：用户有3个设备，修改Registry后自动同步**

```
设备A（在线）        设备B（离线）        设备C（在线）
    │                  │                  │
    │                   │                  │                  │
    │                  │                  │
    │←──────────Registry v5──────────────→│
    │                  │                  │
[用户语音：添加股票页面]
    ↓
Agent修改Registry → v6
    ↓
Sync Service检测到变更
    ↓
┌──────────────────────────────────────────┐
│ 推送到用户下所有在线设备                  │
│ • 设备A：WebSocket推送 → 即时更新        │
│ • 设备B：离线，标记待同步                 │
│ • 设备C：WebSocket推送 → 即时更新        │
└──────────────────────────────────────────┘
    │                  │                  │
    │←──Registry v6──→│                  │←──Registry v6──→│
    │                  │                  │
    │                  │ [设备B上线]      │
    │                  │                  │
    │                  │←──拉取Registry──→│
    │                  │   （检测到v6>v5） │
    │                  │                  │
    │                  │←──下载v6────────→│
    │                  │                  │
```

**同步触发机制**：
1. **服务器推送**：Registry变更时立即推送给所有在线设备
2. **离线设备上线**：设备重连时比较版本号，落后则拉取
3. **定时轮询**：每小时全量检查一次（兜底）

## 3. Page Registry 设计

### 3.1 用户级Registry隔离

**核心变更**：Registry 从"设备级"变为"用户级"

```
数据库层面隔离:
├── users 表
│   └── user_id (PK)
│
├── registries 表
│   ├── registry_id (PK)
│   ├── user_id (FK) ←── 关键：按用户隔离
│   ├── pages (JSONB)
│   ├── templates (JSONB)
│   └── version
│
└── devices 表
    ├── device_id (PK)
    ├── user_id (FK) ←── 关联到用户
    └── registry_version

一个用户的所有设备共享同一个Registry:
User A
├── Registry A (pages, templates, version=5)
├── Device 1 (registry_version=5) ✓ 同步
└── Device 2 (registry_version=5) ✓ 同步

User B
├── Registry B (完全不同的配置)
└── Device 3

Device 1 无法访问 Registry B（权限验证）
```

### 3.2 数据结构

```json
{
  "version": "1.0.0",
  "user_id": "uuid-of-user",      // ← 新增：所属用户
  "pages": [
    {
      "page_id": "date",
      "type": "builtin",
      "visible": true,
      "order": 0,
      "template": "standard",
      "config": {
        "show_lunar": true,
        "font_size": "large"
      }
    },
    {
      "page_id": "weather",
      "type": "builtin", 
      "visible": true,
      "order": 1,
      "template": "standard",
      "config": {
        "show_forecast": true
      }
    },
    {
      "page_id": "todo",
      "type": "builtin",
      "visible": true,
      "order": 2,
      "template": "list",
      "config": {
        "filter": "incomplete",
        "max_items": 7
      }
    },
    {
      "page_id": "stock_simple",
      "type": "custom",
      "visible": true,
      "order": 8,
      "created_by": "agent",
      "created_at": "2026-03-13T10:30:00Z",
      "template": "center_large",
      "config": {
        "data_source": "stock_api",
        "symbol": "AAPL",
        "refresh_interval": 300
      },
      "layout": {
        "elements": [
          {
            "type": "text",
            "field": "change_percent",
            "x": 200,
            "y": 150,
            "font": "xlarge",
            "align": "center",
            "color": "dynamic"
          }
        ]
      }
    }
  ],
  "templates": [
    {
      "template_id": "standard",
      "description": "标准布局：标题+内容+底部信息",
      "slots": ["header", "main", "footer"]
    },
    {
      "template_id": "center_large",
      "description": "居中大字：适合单数据展示",
      "slots": ["main"]
    },
    {
      "template_id": "list",
      "description": "列表布局：垂直排列",
      "slots": ["items"]
    }
  ]
}
```

### 3.2 存储位置

- **云端**: PostgreSQL 数据库存储（按 `user_id` 隔离）
- **设备端**: `/spiffs/registry.json`（离线缓存）
- **同步**: 启动时云端→设备，变更时实时推送

## 4. PageManager Tools

### 4.1 工具清单

```python
@mcp.tool()
async def create_page(
    description: str,
    page_id: str = None,
    template: str = "standard"
) -> dict:
    """
    根据自然语言描述创建新页面
    
    Agent会:
    1. 用LLM解析描述，提取关键信息
    2. 选择合适模板
    3. 生成符合约束的JSON Schema
    4. 保存到Registry
    5. 推送至设备
    """

@mcp.tool()
async def hide_page(page_id: str) -> dict:
    """隐藏指定页面（从页面循环中移除）"""

@mcp.tool()
async def show_page(page_id: str) -> dict:
    """显示指定页面（重新加入页面循环）"""

@mcp.tool()
async def list_pages(
    include_hidden: bool = False
) -> dict:
    """列出所有页面（可包含已隐藏的）"""

@mcp.tool()
async def delete_page(page_id: str) -> dict:
    """删除自定义页面（不能删除内置页面）"""

@mcp.tool()
async def switch_layout(
    layout_name: str
) -> dict:
    """
    切换到预设布局模式
    
    预设布局:
    - "default": 显示所有可见页面
    - "focus": 只显示当前页面
    - "work": 待办+日历+天气
    - "exam": 倒计时+待办
    - "minimal": 只显示日期
    """

@mcp.tool()
async def modify_page(
    page_id: str,
    changes: dict
) -> dict:
    """修改页面配置（字体、颜色、数据源等）"""
```

### 4.2 使用示例

**语音指令 → Tool调用映射**:

| 语音指令 | 调用Tool | 效果 |
|---------|---------|------|
| "隐藏天气页面" | `hide_page("weather")` | weather.visible=false |
| "显示所有页面" | 遍历调用`show_page()` | 所有内置页面visible=true |
| "列举现在的页面" | `list_pages()` | 语音播报页面列表 |
| "添加股票页面只显示涨跌幅" | `create_page(desc="...")` | 生成stock_simple页面 |
| "切换到考试模式" | `switch_layout("exam")` | 只显示countdown+todo |
| "删除股票页面" | `delete_page("stock_simple")` | 从Registry移除 |

## 4.3 DeviceManager Tools

多设备管理相关的MCP Tools：

```python
@mcp.tool()
async def list_user_devices(
    user_id: str = None  # 不传则使用当前用户
) -> dict:
    """
    列出当前用户的所有设备
    
    返回:
    {
        "devices": [
            {
                "device_id": "AA:BB:CC:DD:EE:FF",
                "name": "厨房便利贴",
                "status": "online",
                "last_seen": "2026-03-13T12:00:00Z",
                "battery": 85,
                "registry_version": 5
            }
        ]
    }
    """

@mcp.tool()
async def get_device_status(
    device_id: str
) -> dict:
    """获取指定设备的详细状态"""

@mcp.tool()
async def remote_command(
    device_id: str = None,  # None=所有设备
    command: str = "refresh",  # refresh/reboot/update
    params: dict = None
) -> dict:
    """
    向设备发送远程命令
    
    命令:
    - "refresh": 刷新当前页面
    - "reboot": 远程重启
    - "update": 强制同步Registry
    - "ota": 固件更新
    """

@mcp.tool()
async def assign_device_to_user(
    pairing_code: str,
    user_id: str = None  # None=当前用户（需管理员权限）
) -> dict:
    """
    将待配对设备绑定到指定用户
    
    使用场景:
    1. 管理员查看待配对设备列表
    2. 点击确认绑定，输入配对码
    3. 设备自动接收Registry并进入正常模式
    """
```

## 5. MQTT 主题设计

### 5.1 配对相关主题

| 主题 | 方向 | QoS | 描述 | Payload |
|------|------|-----|------|---------|
| `pairing/request/{device_id}` | D→B | 1 | 设备请求配对 | `{"device_name": "厨房", "firmware": "1.0.0"}` |
| `pairing/code/{device_id}` | B→D | 1 | 下发配对码 | `{"code": "123456", "expires_in": 600}` |
| `pairing/confirmed/{device_id}` | B→D | 2 | 配对确认+Registry | `{"status": "ok", "user_id": "uuid", "registry": {...}}` |
| `pairing/cancelled/{device_id}` | B→D | 1 | 配对取消 | `{"reason": "expired"}` |

### 5.2 设备管理主题

| 主题 | 方向 | QoS | 描述 | Payload |
|------|------|-----|------|---------|
| `device/{device_id}/command` | B→D | 1 | 远程命令 | `{"cmd": "reboot", "timestamp": 1234567890}` |
| `device/{device_id}/status` | D→B | 0 | 状态上报 | `{"online": true, "battery": 85, "page": "date"}` |
| `device/{device_id}/wakeup` | D→B | 0 | RTC/按键唤醒 | `{"timestamp": 1234567890, "reason": "rtc"}` |

### 5.3 Registry 同步主题

| 主题 | 方向 | QoS | 描述 | Payload |
|------|------|-----|------|---------|
| `registry/{user_id}/update` | B→D | 2 | Registry更新推送 | 完整Registry JSON |
| `registry/{device_id}/sync` | D→B | 1 | 设备请求同步 | `{"current_version": 5}` |
| `registry/{device_id}/response` | B→D | 1 | 同步响应 | `{"version": 6, "registry": {...}}` |

### 5.4 语音交互主题（WebSocket，非MQTT）

| 主题 | 方向 | 描述 |
|------|------|------|
| `asr/stream/{device_id}` | D→B | 音频流（二进制） |
| `asr/result/{device_id}` | B→D | ASR识别结果（文本） |
| `agent/response/{device_id}` | B→D | Agent执行结果 |

## 6. Sync Service 设计

### 6.1 核心职责

```python
class SyncService:
    """
    用户级Registry同步服务
    保证同一用户的所有设备最终一致性
    """
    
    async def on_registry_change(self, user_id: str, new_registry: dict):
        """
        Registry变更回调
        1. 更新数据库
        2. 递增版本号
        3. 推送到所有在线设备
        4. 记录同步日志
        """
        
    async def on_device_online(self, device_id: str, user_id: str):
        """
        设备上线回调
        1. 检查设备registry_version
        2. 如果落后，推送更新
        3. 更新设备在线状态
        """
        
    async def periodic_sync(self):
        """
        定时全量同步（每小时）
        1. 遍历所有在线设备
        2. 检查版本号
        3. 落后设备推送更新
        """
```

### 6.2 同步冲突解决

**场景**：设备离线时Registry被修改多次

```
设备离线                 用户修改Registry
    │                           │
    │                    v5 → v6（添加股票页）
    │                           │
    │                    v6 → v7（隐藏天气页）
    │                           │
    │←────设备上线──────────────│
    │                           │
    │ 上报：registry_version=5  │
    │                           │
    │←────推送v7───────────────→│
    │                           │
    │ 设备自动应用v7（覆盖本地） │
    │                           │

策略：服务器版本优先（Server Wins）
原因：
1. 服务器是权威数据源
2. 设备离线期间的所有修改都是"最新意图"
3. 实现简单，无需复杂冲突合并
```

### 6.3 同步性能优化

```python
# 1. 增量推送（Registry很大时）
# 只推送变更的部分
registry_delta = {
    "version": 7,
    "changes": [
        {"op": "add", "path": "/pages/-", "value": new_page},
        {"op": "replace", "/pages/1/visible", "value": false}
    ]
}

# 2. 批量推送（同一用户多设备在线）
async def broadcast_to_user(user_id: str, message: dict):
    """向用户的所有在线设备广播"""
    devices = await get_online_devices_by_user(user_id)
    await asyncio.gather(*[
        send_to_device(d.device_id, message)
        for d in devices
    ])

# 3. 版本号缓存（避免频繁查库）
# Redis: registry:version:{user_id} -> "7"
```

## 7. JSON Schema 布局规范

### 5.1 屏幕约束（必须告知Agent）

```yaml
constraints:
  screen:
    width: 400
    height: 300
    colors: [black, white]  # 灰度通过抖动
    
  fonts:
    sizes: [12, 16, 24, 48]  # small, medium, large, xlarge
    
  icons:
    sizes: [16, 32, 48, 64]
    library: "predefined_100"  # 预置100个图标
    
  layout:
    max_elements: 7  # 单页最多元素数
    templates: 8     # 预置模板数量
```

### 5.2 元素类型

```json
{
  "elements": [
    {
      "type": "text",
      "content": "24°C",
      "field": "temperature",
      "x": 200,
      "y": 100,
      "font": "xlarge",
      "align": "center",
      "color": "black"
    },
    {
      "type": "icon",
      "name": "sunny",
      "x": 150,
      "y": 80,
      "size": 48
    },
    {
      "type": "line",
      "x1": 50,
      "y1": 150,
      "x2": 350,
      "y2": 150,
      "width": 2
    },
    {
      "type": "rect",
      "x": 100,
      "y": 200,
      "width": 200,
      "height": 50,
      "fill": false,
      "border": true
    },
    {
      "type": "qrcode",
      "data": "https://example.com",
      "x": 150,
      "y": 100,
      "size": 100
    }
  ]
}
```

### 5.3 预置模板（15个）

```json
{
  "templates": [
    {
      "id": "center_large",
      "description": "居中大字：适合单数据展示",
      "layout": {
        "main": {"x": 200, "y": 150, "align": "center"}
      }
    },
    {
      "id": "header_list",
      "description": "标题+列表：顶部标题，下方列表",
      "layout": {
        "header": {"x": 200, "y": 30, "align": "center"},
        "list": {"x": 20, "y": 60, "max_items": 7}
      }
    },
    {
      "id": "card_grid",
      "description": "卡片网格：2x2或3x2网格",
      "layout": {
        "grid": {"cols": 2, "rows": 2, "margin": 10}
      }
    },

  ]
}
```

## 6. ESP32 渲染层实现

### 6.1 基于 Paperd.Ink 扩展

**Paperd.Ink 现有能力**:
- 基础 UI 组件（按钮、文本、图标）
- JSON 配置驱动
- GxEPD2 集成

**需要扩展的部分**:

```cpp
// layout_engine.h
class LayoutEngine {
public:
    void parseConfig(const char* json);
    void render();
    
private:
    void renderText(JsonObject element);
    void renderIcon(JsonObject element);
    void renderLine(JsonObject element);
    void renderRect(JsonObject element);
    void renderBitmap(JsonObject element);
    
    // 自动布局计算
    void calculateLayout();
    Point applyTemplate(String templateId, int slotIndex);
};

// registry_manager.h
class PageRegistry {
public:
    void loadFromSPIFFS();
    void syncFromCloud();
    void applyUpdate(JsonObject update);
    
    bool isVisible(String pageId);
    int getPageCount();
    String getNextPage();
    String getPrevPage();
    
private:
    JsonDocument registry;
    int currentPageIndex;
    std::vector<String> visiblePages;
};
```

### 6.2 关键流程

**启动时**:
```
ESP32启动 → 从SPIFFS加载Registry → 计算可见页面列表 → 显示第一页
```

**页面切换时（单击/双击）**:
```
按钮中断 → 查询Registry下一页 → 从SPIFFS加载页面配置 → LayoutEngine渲染 → 刷新墨水屏
```

**收到MQTT更新时**:
```
WebSocket消息 → 解析JSON更新 → 更新Registry → 保存到SPIFFS → 如当前页被影响则重新渲染
```

## 7. RTC 休眠与电源管理

### 7.1 休眠策略

**目标**：最大化续航，在便利贴使用场景下实现低功耗。

**休眠模式**：
- **深度休眠 (Deep Sleep)**：ESP32 进入深度休眠，WiFi 断开，RTC 定时器和 GPIO 中断保持运行
- **活跃模式 (Active)**：WiFi 连接，正常响应语音和页面切换

**唤醒源**：
1. **RTC 定时器**：周期性唤醒检查消息
2. **按键中断**：用户按键立即唤醒

### 7.2 RTC 唤醒周期

| 时段 | 唤醒间隔 | 说明 |
|------|----------|------|
| 白天 (8:00-22:00) | 30 分钟 | 消息检查较频繁 |
| 夜间 (22:00-8:00) | 60 分钟 | 降低频率以省电 |

**动态调整**：
- 唤醒后检查当前时间，计算下次唤醒时间
- 使用 RTC 内存保存下次唤醒时间戳

### 7.3 活跃期管理

**语音操作后的活跃期**：
- 语音操作完成后，设备保持活跃 **5 分钟**
- 期间不进入休眠，方便用户连续操作
- 5 分钟内无操作，自动进入深度休眠

**活跃期计时器**：
```cpp
// 每次用户操作后重置计时器
void resetActivityTimer() {
    activityTimeout = millis() + 5 * 60 * 1000;  // 5分钟后
}

// 主循环检查
void loop() {
    if (millis() > activityTimeout && !inActiveMode) {
        enterDeepSleep();
    }
}
```

### 7.4 休眠与唤醒流程

**进入深度休眠**：
```
1. 保存当前状态到 RTC 内存
2. 断开 WiFi 连接
3. 配置 RTC GPIO 唤醒（按键）
4. 配置 RTC 定时器唤醒
5. 进入 esp_deep_sleep_start()
```

**唤醒后恢复**：
```
1. 从 RTC 内存恢复状态
2. 判断唤醒源（RTC/按键）
3. 如果是 RTC 唤醒：
   - 连接 WiFi
   - 检查 MQTT 消息
   - 如有更新，刷新屏幕
   - 进入休眠
4. 如果是按键唤醒：
   - 连接 WiFi
   - 同步最新数据
   - 处理按键事件（页面切换/语音）
   - 重置活跃期计时器
   - 保持活跃直到超时
```

### 7.5 功耗估算

| 模式 | 电流 | 占比 | 备注 |
|------|------|------|------|
| 深度休眠 | 0.1-0.5 mA | 95%+ | RTC 运行，WiFi 关闭 |
| WiFi 连接 | 50-100 mA | 短暂 | 每次唤醒约 3-5 秒 |
| 墨水屏刷新 | 20-30 mA | 极短 | 仅刷新时 |
| 语音处理 | 100-150 mA | 用户触发 | 仅录音和上传时 |

**续航估算**（假设 2000mAh 电池）：
- 纯休眠模式：理论续航 4-6 个月
- 实际使用（含唤醒）：约 1-2 个月
- 频繁语音操作：约 2-4 周

### 7.6 实现代码框架

```cpp
// power_manager.h
class PowerManager {
public:
    void init();
    void enterDeepSleep();
    void handleWakeup();
    void resetActivityTimer();
    bool shouldEnterSleep();
    
private:
    uint64_t calculateSleepDuration();
    void setupWakeupSources();
    bool isDaytime();
};

// 主循环
void loop() {
    // 处理按键、语音、网络等...
    
    // 检查是否应该进入休眠
    if (powerManager.shouldEnterSleep()) {
        powerManager.enterDeepSleep();
    }
}
```

## 8. 依赖清单

### 8.1 云端

```toml
[project]
name = "ai-sticky-server"
version = "2.0.0"
dependencies = [
    # Web
    "fastapi>=0.100.0",
    "uvicorn>=0.23.0",
    "websockets>=11.0",
    
    # Auth
    "python-jose[cryptography]>=3.3.0",  # JWT
    "passlib[bcrypt]>=1.7.4",  # 密码哈希
    
    # Agent
    "pydantic-ai>=0.1.0",
    "pydantic>=2.0.0",
    "openai>=1.0.0",
    
    # MCP
    "mcp>=1.0.0",
    
    # Data - PostgreSQL + Redis
    "asyncpg>=0.29.0",  # PostgreSQL异步驱动
    "redis>=5.0.0",  # Redis客户端
    
    # MQTT
    "aiomqtt>=1.2.0",  # MQTT客户端
    "paho-mqtt>=1.6.0",  # MQTT（备选）
    
    # Config
    "python-dotenv>=1.0.0",
]
```

### 8.2 设备端（ESP32）

```ini
[env:esp32-s3]
platform = espressif32
board = esp32-s3-devkitc-1
framework = arduino

lib_deps =
    # 墨水屏驱动
    zinggjm/GxEPD2@^1.5.0
    adafruit/Adafruit GFX Library@^1.11.0
    
    # JSON解析
    bblanchon/ArduinoJson@^7.0.0
    
    # 网络
    links2004/WebSockets@^2.4.0
    
    # AP模式配网
    me-no-dev/AsyncTCP@^1.1.1
    me-no-dev/ESPAsyncWebServer@^1.2.3
    
    # MQTT
    knolleary/PubSubClient@^2.8
    
    # WiFi管理
    tzapu/WiFiManager@^2.0.0  # 可选，如果不用自定义AP模式
    
    # UI（Paperd.Ink扩展）
    # 从 https://github.com/paperd.ink/Paperd.Ink-Library fork并扩展
```

## 9. 部署配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  server:
    build: ./server
    ports:
      - "8000:8000"
    environment:
      # ASR
      - DOUBAO_ASR_APPID=${DOUBAO_ASR_APPID}
      - DOUBAO_ASR_TOKEN=${DOUBAO_ASR_TOKEN}
      
      # LLM（固定单一模型）
      - LLM_PROVIDER=kimi  # 或 doubao/deepseek/qwen
      - LLM_MODEL=moonshot-v1-8k
      - LLM_API_KEY=${KIMI_API_KEY}
      
      # Database
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/ai_sticky
      
      # Redis
      - REDIS_URL=redis://redis:6379/0
      
      # MQTT Broker（使用emqx或mosquitto）
      - MQTT_BROKER=mqtt
      - MQTT_PORT=1883
      
      # Admin
      - ADMIN_USERNAME=${ADMIN_USERNAME:-admin}
      - ADMIN_PASSWORD_HASH=${ADMIN_PASSWORD_HASH}
      
    depends_on:
      - postgres
      - redis
      - mqtt
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=ai_sticky
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # 初始化脚本
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  mqtt:
    image: emqx:5  # 或使用 eclipse-mosquitto:2
    ports:
      - "1883:1883"   # MQTT
      - "8083:8083"   # WebSocket
      - "18083:18083" # Dashboard
    volumes:
      - emqx_data:/opt/emqx/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  emqx_data:
```

## 9. 语音交互示例

```
用户：隐藏天气
系统：已隐藏天气页面

用户：列举页面
系统：您有8个页面：日期、待办、倒计时、节气、电量、同步

用户：添加股票页面显示涨跌幅
系统：股票页面已添加

用户：切换到考试模式
系统：已切换到考试模式，只显示倒计时和待办
```

## 10. 关键技术决策

### 10.1 为什么选择 JSON Schema 而非直接代码生成？

| 方案 | 优点 | 缺点 | 选择 |
|------|------|------|------|
| **JSON Schema** | 安全（可验证）、轻量、易传输 | 功能受限 | ✅ 采用 |
| **直接生成C++代码** | 功能强大 | 风险高、需编译、难验证 | ❌ 否决 |
| **Lua脚本** | 灵活 | 需要嵌入式Lua解释器 | ❌ 资源有限 |

### 10.2 为什么基于 Paperd.Ink 而非从头写？

- Paperd.Ink 已有基础 UI 组件和 JSON 驱动能力
- 只需扩展 Layout Engine 和 Registry Manager
- 减少 50% 开发工作量

### 10.3 墨水屏约束如何处理？

- **Prompt工程**：在 Agent System Prompt 中明确约束（400x300、4种字号、黑白）
- **Schema验证**：Pydantic 模型强制验证坐标、字号合法性
- **模板限制**：8个预置模板限制自由度过高导致的无效布局
- **运行时保护**：ESP32 端过滤越界元素，防止崩溃

---

**架构完成**。这个设计实现了：
- ✅ 5层清晰分层
- ✅ Agent 自主管理页面（创建/隐藏/列举）
- ✅ JSON Schema 驱动渲染
- ✅ 基于 Paperd.Ink 的 ESP32 渲染层
- ✅ 完整的语音交互闭环
