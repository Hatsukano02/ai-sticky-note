# MCP Tools 规范

## 1. Tools 目录结构

每个功能模块一个 Python 文件，使用 `@mcp.tool()` 装饰器：

```
mcp_tools/
├── __init__.py
├── date.py              # 日期相关
├── weather.py           # 天气相关
├── todo.py              # 待办清单
├── calendar.py          # 日历日程
├── countdown.py         # 倒计时
├── solar_term.py        # 节气
├── battery.py           # 电量
├── sync.py              # 同步控制
├── page.py              # 页面导航
└── system.py            # 系统控制
```

## 2. Tool 定义示例

```python
# mcp_tools/todo.py
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field
from typing import Optional
import uuid

mcp = FastMCP("todo")

class AddTodoInput(BaseModel):
    """添加待办输入"""
    content: str = Field(
        ..., 
        description="待办内容，最多15字",
        max_length=15
    )
    priority: int = Field(
        default=1,
        ge=1, le=3,
        description="优先级：1=普通, 2=重要, 3=紧急"
    )
    due_date: Optional[str] = Field(
        None,
        description="截止日期（YYYY-MM-DD）"
    )

@mcp.tool()
async def add_todo(input: AddTodoInput, device_id: str = None) -> dict:
    """
    添加待办事项
    
    语音示例：
    - "添加待办买牛奶"
    - "添加重要待办明天下午3点开会"
    """
    pass
```

## 3. 所有 Tools 清单

### 3.1 Date
| Tool | 描述 | 参数 |
|------|------|------|
| `switch_template` | 切换日期模板 | `template: standard\|minimal\|festival` |
| `toggle_lunar` | 开关农历显示 | `show: bool` |
| `set_font_size` | 设置字体大小 | `size: small\|medium\|large` |

### 3.2 Weather
| Tool | 描述 | 参数 |
|------|------|------|
| `switch_template` | 切换天气模板 | `template: standard\|detail\|minimal` |
| `toggle_forecast` | 开关预报显示 | `show: bool` |
| `refresh_weather` | 刷新天气数据 | - |

### 3.3 Todo
| Tool | 描述 | 参数 |
|------|------|------|
| `add_todo` | 添加待办 | `content, priority, due_date` |
| `list_todos` | 列出待办 | `filter: all\|today\|high_priority\|completed` |
| `toggle_todo` | 切换完成状态 | `todo_id` |
| `delete_todo` | 删除待办 | `todo_id` |
| `set_filter` | 设置筛选 | `filter` |

### 3.4 Calendar
| Tool | 描述 | 参数 |
|------|------|------|
| `switch_view` | 切换视图 | `view: month\|week\|day\|agenda` |
| `toggle_options` | 开关选项 | `lunar, holiday, workday_adjust` |
| `add_event` | 添加事件 | `title, date, reminder` |
| `delete_event` | 删除事件 | `event_id` |

### 3.5 Countdown
| Tool | 描述 | 参数 |
|------|------|------|
| `add_countdown` | 添加倒计时 | `name, target_date, display_style` |
| `delete_countdown` | 删除倒计时 | `countdown_id` |
| `switch_style` | 切换显示样式 | `style: days\|weeks\|percent` |
| `adjust_target` | 调整目标日期 | `days: int` |

### 3.6 SolarTerm
- 只读，无操作 Tools

### 3.7 Battery
- 只读，无操作 Tools

### 3.8 Sync
| Tool | 描述 | 参数 |
|------|------|------|
| `force_sync` | 强制同步 | - |
| `reset_connection` | 重置连接 | - |
| `get_sync_status` | 获取同步状态 | - |

### 3.9 Page（页面导航）
| Tool | 描述 | 参数 |
|------|------|------|
| `switch_page` | 切换页面 | `page: date\|weather\|todo\|...` |

### 3.10 System（系统控制）
| Tool | 描述 | 参数 |
|------|------|------|
| `reboot` | 重启设备 | - |
| `shutdown` | 关机 | - |

## 4. 语音意图映射

| 自然语言 | Tool | 参数示例 |
|---------|------|----------|
| 切换到天气 | `switch_page` | `{"page": "weather"}` |
| 显示详细天气 | `weather.switch_template` | `{"template": "detail"}` |
| 添加待办买牛奶 | `todo.add_todo` | `{"content": "买牛奶"}` |
| 完成买牛奶 | `todo.toggle_todo` | `{"content": "买牛奶"}` |
| 设置考试倒计时100天 | `countdown.add_countdown` | `{"name": "考试", "target_date": "2026-06-07"}` |
| 切换到周视图 | `calendar.switch_view` | `{"view": "week"}` |
| 立即同步 | `sync.force_sync` | `{}` |
| 重启设备 | `system.reboot` | `{}` |

## 5. ESP32 通信协议

### 5.1 WebSocket 消息类型

| 类型 | 方向 | 说明 |
|------|------|------|
| `auth` | C→S | 设备认证 |
| `audio_stream` | C→S | 音频数据流 |
| `audio_end` | C→S | 音频发送完成 |
| `tool_call` | S→C | 调用 Tool |
| `tool_result` | C→S | Tool 执行结果 |
| `display_update` | S→C | 显示更新 |
| `sync_request` | C→S | 同步请求 |
| `wakeup` | C→S | 设备唤醒（RTC/按键） |

### 5.2 典型语音流程

```
[长按] → ESP32 开始录音
        ↓
[松手] → WebSocket 发送音频数据
        ↓
[云端] → 豆包 ASR 识别
        ↓
       → Pydantic AI Agent 解析意图
        ↓
       → 调用相应 Tools
        ↓
       → 返回 JSON 结果
        ↓
[ESP32] → 更新显示 + 语音播报
```

## 6. 数据格式

### 6.1 Tool 调用请求（服务端→设备）
```json
{
  "msg_type": "tool_call",
  "tool": "todo.add_todo",
  "params": {
    "content": "买牛奶",
    "priority": 1
  }
}
```

### 6.2 Tool 调用响应（设备→服务端）
```json
{
  "msg_type": "tool_result",
  "tool": "todo.add_todo",
  "success": true,
  "data": {
    "todo_id": "abc123",
    "message": "已添加待办：买牛奶"
  }
}
```

### 6.3 显示更新（服务端→设备）
```json
{
  "msg_type": "display_update",
  "voice_text": "已添加待办",
  "page": "todo",
  "data": {
    "todos": [...]
  },
  "should_refresh": true
}
```
