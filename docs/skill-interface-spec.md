# Skill接口设计规范

## 1. Skill结构

每个Skill独立目录，标准结构：

```
skill_name/
├── manifest.json          # Skill元数据
├── tools.py              # Tool实现
├── prompts.yaml          # System prompts
├── templates.json        # 页面模板定义
└── data_schema.json      # 数据结构定义
```

## 2. manifest.json

```json
{
  "skill_id": "date_skill",
  "name": "Date Display",
  "version": "1.0.0",
  "description": "日期显示与农历管理",
  "author": "AI冰箱贴官方",
  "dependencies": [],
  "tools": [
    "switch_template",
    "toggle_lunar",
    "set_font_size"
  ],
  "pages": ["date"],
  "default_template": "standard"
}
```

## 3. Tools规范

### 3.1 DateSkill

**switch_template**
```json
{
  "tool": "switch_template",
  "params": {
    "template": "standard|minimal|festival"
  }
}
```

**toggle_lunar**
```json
{
  "tool": "toggle_lunar", 
  "params": {
    "show": true
  }
}
```

### 3.2 WeatherSkill

**switch_template**
```json
{
  "tool": "switch_template",
  "params": {
    "template": "standard|detail|minimal"
  }
}
```

**toggle_forecast**
```json
{
  "tool": "toggle_forecast",
  "params": {
    "show": true
  }
}
```

### 3.3 TodoSkill

**add_todo**
```json
{
  "tool": "add_todo",
  "params": {
    "content": "买牛奶",
    "priority": 1,
    "due_date": "2026-03-15"
  }
}
```

**toggle_todo**
```json
{
  "tool": "toggle_todo",
  "params": {
    "id": "uuid"
  }
}
```

**delete_todo**
```json
{
  "tool": "delete_todo",
  "params": {
    "id": "uuid"
  }
}
```

**set_filter**
```json
{
  "tool": "set_filter",
  "params": {
    "filter": "all|today|high_priority|completed"
  }
}
```

### 3.4 CalendarSkill

**switch_view**
```json
{
  "tool": "switch_view",
  "params": {
    "view": "month|week|day|agenda"
  }
}
```

**toggle_options**
```json
{
  "tool": "toggle_options",
  "params": {
    "lunar": true,
    "holiday": true,
    "workday_adjust": false
  }
}
```

### 3.5 CountdownSkill

**add_countdown**
```json
{
  "tool": "add_countdown",
  "params": {
    "name": "考试",
    "target_date": "2026-06-07",
    "display_style": "days"
  }
}
```

**switch_display_style**
```json
{
  "tool": "switch_display_style",
  "params": {
    "style": "days|weeks|months|percent|progress_ring"
  }
}
```

**adjust_target**
```json
{
  "tool": "adjust_target",
  "params": {
    "days": 7
  }
}
```

### 3.6 SyncSkill

**force_sync**
```json
{
  "tool": "force_sync"
}
```

**reset_connection**
```json
{
  "tool": "reset_connection"
}
```

## 4. MCP消息协议

### 4.1 请求格式

```json
{
  "msg_type": "tool_call",
  "msg_id": "uuid",
  "timestamp": 1710420000,
  "skill": "todo_skill",
  "tool": "add_todo",
  "params": {},
  "context": {
    "current_page": "todo",
    "device_id": "device_uuid"
  }
}
```

### 4.2 响应格式

```json
{
  "msg_type": "tool_result",
  "msg_id": "uuid",
  "timestamp": 1710420000,
  "success": true,
  "data": {},
  "error": null
}
```

### 4.3 错误格式

```json
{
  "msg_type": "tool_result",
  "success": false,
  "error": {
    "code": "INVALID_PARAM",
    "message": "参数错误"
  }
}
```

## 5. 模板定义规范

### 5.1 templates.json

```json
{
  "templates": [
    {
      "id": "standard",
      "name": "标准视图",
      "layout": {
        "type": "vertical",
        "elements": [
          {"type": "date", "size": "large"},
          {"type": "weekday", "size": "medium"},
          {"type": "lunar", "size": "small"}
        ]
      }
    }
  ]
}
```

## 6. 语音意图映射

| 自然语言 | Skill | Tool | 参数 |
|---------|-------|------|------|
| 切换到天气 | - | switch_page | {"page": "weather"} |
| 显示详细天气 | weather_skill | switch_template | {"template": "detail"} |
| 添加待办买牛奶 | todo_skill | add_todo | {"content": "买牛奶"} |
| 设置考试倒计时30天 | countdown_skill | add_countdown | {"name": "考试", "target_date": "..."} |
| 切换到周视图 | calendar_skill | switch_view | {"view": "week"} |
| 立即同步 | sync_skill | force_sync | {} |
