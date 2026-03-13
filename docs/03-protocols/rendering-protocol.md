---
title: JSON Schema 渲染协议
description: 跨平台嵌入式UI渲染框架，定义JSON Schema格式、组件系统、布局引擎
type: protocol
scope: [device]
created: "2026-03-13"
updated: "2026-03-13"
dependencies: []
related:
  - ../04-modules/ui-display.md
  - ../06-interface/mcp-tools.md
---

# JSON Schema 渲染框架规范

## 1. 概述

### 1.1 框架定位

**通用嵌入式 UI 框架，AI便利贴是其中一个应用实例**

设计目标：
- **跨平台**: ESP32、STM32、Linux
- **跨显示技术**: 墨水屏、OLED、LCD
- **声明式**: JSON 配置驱动，无需编译
- **可扩展**: 插件化组件系统

### 1.2 架构分层

```
应用层 (AI便利贴)
    ↓ JSON Schema
┌─────────────────────────────────────────────┐
│ 框架核心                                     │
│ ├── Schema Parser (JSON验证、布局计算)      │
│ ├── Component System (组件库)               │
│ └── Render Backend Interface (抽象接口)     │
└─────────────────────────────────────────────┘
    ↓ 适配层
后端实现 (GxEPD2 墨水屏驱动)
```

## 2. JSON Schema 格式

### 2.1 根结构

```json
{
  "version": "1.0",
  "meta": {
    "target": {
      "display": {
        "width": 400,
        "height": 300,
        "type": "epaper",
        "color_depth": 1
      }
    }
  },
  "page_id": "date",
  "layout": {
    "type": "flex",
    "direction": "column",
    "justify": "center",
    "align": "center"
  },
  "elements": []
}
```

### 2.2 元素类型

#### Text (文字)
```json
{
  "type": "text",
  "text": "Hello",
  "bind": "${data.field}",
  "x": 200,
  "y": 150,
  "font": "xxlarge",
  "align": "center"
}
```

**支持字号**: 12/16/24/48/72/96px（6种）

#### Icon (图标)
```json
{
  "type": "icon",
  "name": "sunny",
  "x": 100,
  "y": 50,
  "size": 48
}
```

**预置图标**: 天气、日历、状态等100个

#### Symbol (矢量符号)
```json
{
  "type": "symbol",
  "name": "circle_empty",
  "x": 20,
  "y": 80,
  "size": 16
}
```

**预置符号**: circle_empty/filled, check_mark, arrow_*, battery_*

#### Shape (图形)
```json
{
  "type": "line",
  "x1": 50, "y1": 150, "x2": 350, "y2": 150,
  "width": 2
}
```

```json
{
  "type": "rect",
  "x": 100, "y": 100,
  "width": 200, "height": 50,
  "fill": false, "border": true
}
```

#### CalendarGrid (月历网格)
```json
{
  "type": "calendar_grid",
  "x": 20, "y": 60,
  "width": 360, "height": 200,
  "year": 2026, "month": 3,
  "highlight_today": true,
  "show_lunar": true
}
```

#### QrCode (二维码)
```json
{
  "type": "qrcode",
  "data": "https://example.com",
  "x": 150, "y": 100,
  "size": 100
}
```

### 2.3 布局系统

**flex 布局**:
```json
{
  "layout": {
    "type": "flex",
    "direction": "column",
    "justify": "center",
    "align": "center",
    "gap": 10
  }
}
```

**absolute 布局**:
```json
{
  "layout": {
    "type": "absolute"
  },
  "elements": [
    {"type": "text", "x": 200, "y": 150}
  ]
}
```

### 2.4 数据绑定

```json
{
  "text": "${temperature}°C",
  "visible": "${temperature > 0}",
  "color": "${temperature > 30 ? '#FF0000' : '#000000'}"
}
```

## 3. 预置模板库

框架提供8个常用模板：

| 模板ID | 说明 | 适用场景 |
|--------|------|----------|
| center_large | 居中大字 | 日期、倒计时 |
| header_list | 标题+列表 | 待办、日程 |
| info_card | 图标+数值 | 天气、电量 |
| calendar_month | 月历视图 | 日历页面 |
| two_column | 双栏对比 | 对比信息 |
| vertical_stack | 垂直堆叠 | 多行信息 |
| status_board | 状态面板 | 同步页 |
| minimal | 极简单行 | 单数据展示 |

## 4. 预制模板与组件库

### 4.1 预制模板系统

框架支持**模板继承和复用**，避免重复定义相似布局。

#### 模板定义

```json
{
  "templates": {
    "base_card": {
      "type": "container",
      "layout": {
        "type": "flex",
        "direction": "column",
        "padding": 10,
        "gap": 5
      },
      "style": {
        "border": true,
        "border_width": 1
      }
    },
    "info_card": {
      "extends": "base_card",
      "elements": [
        {"type": "icon", "id": "icon", "size": 32},
        {"type": "text", "id": "value", "font": "large"},
        {"type": "text", "id": "label", "font": "small"}
      ]
    },
    "weather_card": {
      "extends": "info_card",
      "elements": [
        {"type": "icon", "id": "icon", "name": "sunny", "size": 48},
        {"type": "text", "id": "temp", "bind": "${temperature}°C", "font": "xlarge"},
        {"type": "text", "id": "desc", "bind": "${description}", "font": "small"}
      ]
    }
  }
}
```

#### 使用模板

```json
{
  "type": "template",
  "template_id": "weather_card",
  "data": {
    "temperature": 24,
    "description": "多云"
  }
}
```

展开后等效于：

```json
{
  "type": "container",
  "layout": {"type": "flex", "direction": "column", "padding": 10, "gap": 5},
  "style": {"border": true, "border_width": 1},
  "elements": [
    {"type": "icon", "name": "sunny", "size": 48},
    {"type": "text", "text": "24°C", "font": "xlarge"},
    {"type": "text", "text": "多云", "font": "small"}
  ]
}
```

### 4.2 组件库

框架内置**常用组件组合**，开箱即用。

#### 预置组件清单

| 组件ID | 说明 | 组成 |
|--------|------|------|
| `list_item` | 列表项 | 图标 + 文字 + 箭头 |
| `status_badge` | 状态标记 | 圆点 + 文字 |
| `progress_bar` | 进度条 | 背景条 + 进度条 + 百分比文字 |
| `toggle_switch` | 开关 | 滑块 + 背景（支持点击）|
| `input_field` | 输入框 | 边框 + 文字 + 光标（仅OLED/LCD）|

#### 组件使用示例

```json
{
  "type": "list_item",
  "icon": "calendar",
  "text": "日程安排",
  "detail": "3项待办",
  "show_arrow": true,
  "on_click": "${navigate('calendar')}"
}
```

展开为：

```json
{
  "type": "container",
  "layout": {"type": "flex", "direction": "row", "align": "center"},
  "elements": [
    {"type": "icon", "name": "calendar", "size": 24},
    {"type": "text", "text": "日程安排", "flex": 1},
    {"type": "text", "text": "3项待办", "font": "small", "color": "gray"},
    {"type": "symbol", "name": "arrow_right", "size": 16}
  ]
}
```

### 4.3 自定义模板库

项目可以定义自己的模板库：

```json
{
  "template_library": {
    "name": "ai_sticky_templates",
    "version": "1.0",
    "templates": {
      "date_display": {
        "description": "日期展示模板",
        "elements": [
          {"type": "text", "bind": "year_month", "font": "small"},
          {"type": "text", "bind": "day", "font": "xxlarge"},
          {"type": "text", "bind": "weekday", "font": "normal"}
        ]
      },
      "todo_list": {
        "description": "待办列表模板",
        "repeat": {
          "data": "todos",
          "max_items": 7,
          "template": {
            "type": "container",
            "layout": {"type": "flex", "direction": "row"},
            "elements": [
              {"type": "symbol", "name": "${completed ? 'circle_filled' : 'circle_empty'}"},
              {"type": "text", "bind": "content", "flex": 1}
            ]
          }
        }
      }
    }
  }
}
```

使用时：

```json
{
  "page_id": "date",
  "template": "date_display",
  "data": {
    "year_month": "2026年3月",
    "day": "13",
    "weekday": "星期五"
  }
}
```

## 5. AI便利贴配置

### 4.1 硬件配置

```json
{
  "meta": {
    "target": {
      "display": {
        "width": 400,
        "height": 300,
        "type": "epaper",
        "color_depth": 1
      },
      "platform": "esp32"
    }
  }
}
```

### 4.2 字体配置

```json
{
  "fonts": {
    "tiny": {"size": 12},
    "small": {"size": 16},
    "normal": {"size": 24},
    "large": {"size": 48},
    "xlarge": {"size": 72},
    "xxlarge": {"size": 96}
  }
}
```

### 4.3 页面示例

**日期页面**:
```json
{
  "page_id": "date",
  "layout": {
    "type": "flex",
    "direction": "column",
    "justify": "center",
    "align": "center"
  },
  "elements": [
    {"type": "text", "bind": "year_month", "font": "small"},
    {"type": "text", "bind": "day", "font": "xxlarge"},
    {"type": "text", "bind": "weekday", "font": "normal"}
  ]
}
```

**待办页面**:
```json
{
  "page_id": "todo",
  "layout": {"type": "absolute"},
  "elements": [
    {"type": "text", "text": "待办清单", "x": 20, "y": 30, "font": "normal"},
    {
      "type": "repeat",
      "data": "todos",
      "template": {
        "elements": [
          {"type": "symbol", "name": "circle_empty", "x": 20, "y": "${index * 30 + 60}"},
          {"type": "text", "bind": "content", "x": 45, "y": "${index * 30 + 60}", "font": "small"}
        ]
      }
    }
  ]
}
```

## 5. 系统状态栏

固定在屏幕顶部 (y: 0-25)，由系统层独立管理：

```
┌──────────────────────────────────────┐
│ WiFi ●  🔋85%              14:32    │ ← 状态栏 (25px)
├──────────────────────────────────────┤
│         页面内容区域 (400x275)        │
└──────────────────────────────────────┘
```

状态栏更新不触发页面重绘，使用局部刷新。

## 6. 性能优化

### 6.1 局部刷新策略

| 操作 | 刷新区域 | 耗时 |
|------|---------|------|
| 页面切换 | 全屏 | 1.5s |
| 待办完成 | 单个条目 | 0.3s |
| 时间更新 | 状态栏 | 0.3s |

### 6.2 内存管理

- 字体缓存: 最多3个
- 图标缓存: 100个预置
- JSON缓冲: 4KB栈分配

## 7. 实现指南

### 7.1 核心类

```cpp
class Renderer {
    virtual void drawText(int x, int y, String text, Font font);
    virtual void drawRect(int x, int y, int w, int h);
    virtual void present();
};

class Component {
    virtual void render(Renderer* r);
    virtual Size measure(Constraint c);
};
```

### 7.2 依赖

```ini
lib_deps =
    GxEPD2@^1.5.0
    Adafruit GFX
    ArduinoJson@^7.0.0
    Paperd.Ink (fork定制版)
```

## 8. 扩展开发

### 8.1 自定义组件

```cpp
class MyComponent : public Component {
    void render(Renderer* r) override {
        r->drawRect(x, y, w, h);
    }
};
REGISTER_COMPONENT("my_component", MyComponent);
```

### 8.2 自定义后端

```cpp
class MyDisplay : public Renderer {
    void drawPixel(int x, int y, Color c) override {
        // 驱动实现
    }
};
```

---

**框架完成**。这是一个通用的 JSON Schema 渲染框架，AI便利贴是典型应用实例。支持跨平台、跨显示技术、可扩展组件系统。
