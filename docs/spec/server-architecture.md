# 服务端架构规范

## 1. 五层架构设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 1: 后端层 (Backend)                                               │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ • WebSocket Gateway（设备连接管理）                               │ │
│  │ • 豆包 ASR（流式语音识别）                                         │ │
│  │ • 数据持久化（SQLite/MongoDB）                                     │ │
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
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓ 调用 Tools
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 3: 管理层 (Management)                                            │
│  ┌───────────────────────────┐    ┌─────────────────────────────────┐ │
│  │ MCP Tools                 │    │ Page Registry（页面注册表）       │ │
│  │                           │    │                                 │ │
│  │ • 原8个页面 Tools         │    │ • 内置页面（8个，可显隐）         │ │
│  │   - todo/weather/date...  │    │ • 自定义页面（Agent 生成）        │ │
│  │                           │    │ • 布局模板库（8个预置模板）       │ │
│  │ • PageManager（新增）     │    │ • 页面状态（visible/order）       │ │
│  │   - create_page()         │    │                                 │ │
│  │   - hide_page()           │    │ JSON Schema 格式：               │ │
│  │   - list_pages()          │    │ {                                │ │
│  │   - switch_layout()       │    │   "page_id": "weather",         │ │
│  │                           │    │   "visible": true,              │ │
│  │                           │    │   "template": "standard",       │ │
│  │                           │    │   "elements": [...]             │ │
│  │                           │    │ }                                │ │
│  └───────────────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓ MQTT推送
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 4: 渲染层 (Rendering)                                             │
│  ESP32 动态渲染引擎（基于 Paperd.Ink 扩展）                              │
│  ┌───────────────────────────────────────────────────────────────────┐ │
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
    MQTT推送 → ESP32从页面循环中移除
                    ↓
    用户单击/双击不再显示这些页面
```

## 3. Page Registry 设计

### 3.1 数据结构

```json
{
  "version": "1.0.0",
  "device_id": "esp32_abc123",
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

- **云端**: `/data/registry/{device_id}.json`（MongoDB 或 SQLite）
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

## 5. JSON Schema 布局规范

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

## 7. 依赖清单

### 7.1 云端

```toml
[project]
name = "ai-sticky-server"
version = "2.0.0"
dependencies = [
    # Web
    "fastapi>=0.100.0",
    "uvicorn>=0.23.0",
    "websockets>=11.0",
    
    # Agent
    "pydantic-ai>=0.1.0",
    "pydantic>=2.0.0",
    "openai>=1.0.0",
    
    # MCP
    "mcp>=1.0.0",
    
    # Data
    "motor>=3.0.0",  # MongoDB
    "aiosqlite>=0.19.0",
    
    # Config
    "python-dotenv>=1.0.0",
]
```

### 7.2 设备端（ESP32）

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
    
    # UI（Paperd.Ink扩展）
    # 从 https://github.com/paperd.ink/Paperd.Ink-Library fork并扩展
```

## 8. 部署配置

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
      - MONGODB_URL=mongodb://mongo:27017/ai_sticky
      
    depends_on:
      - mongo
    restart: unless-stopped

  mongo:
    image: mongo:7
    volumes:
      - mongo_data:/data/db
    restart: unless-stopped

volumes:
  mongo_data:
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
