# AI便利贴 文档目录

## 文档组织结构

本文档按**层次化结构**组织，从高层产品定义到底层实现细节：

```
docs/
├── 01-product/           # 产品层：需求与规划
├── 02-architecture/      # 架构层：系统、数据、硬件架构
├── 03-protocols/         # 协议层：通信协议与交互约定
├── 04-modules/           # 模块层：功能模块详细规范
├── 05-hardware/          # 硬件层：硬件交互与实现
├── 06-interface/         # 接口层：后端对外接口
└── README.md             # 本文档（文档地图）
```

---

## 目录详解

### 01-product/ - 产品层

定义产品目标、用户故事和开发路线。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [PRD.md](01-product/PRD.md) | 产品需求文档：核心价值、用户故事、功能需求 | - |
| [roadmap.md](01-product/roadmap.md) | 开发路线图：阶段划分、里程碑、任务清单 | PRD.md |

---

### 02-architecture/ - 架构层

系统整体架构设计，包括服务端、数据层、硬件层。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [system-architecture.md](02-architecture/system-architecture.md) | 五层架构设计：WebUI、后端服务、Agent层、管理层、渲染层 | database-schema.md |
| [database-schema.md](02-architecture/database-schema.md) | 数据库设计：PostgreSQL表结构、Redis数据结构、多租户隔离 | - |
| [hardware-architecture.md](02-architecture/hardware-architecture.md) | 硬件架构：BOM清单、功耗预算、组件选型 | - |

---

### 03-protocols/ - 协议层

设备与后端之间的通信协议、交互约定。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [pairing-protocol.md](03-protocols/pairing-protocol.md) | 设备配对协议：AP模式、配对码、MQTT流程、安全机制 | system-architecture.md |
| [sync-protocol.md](03-protocols/sync-protocol.md) | 离线同步协议：同步触发、数据格式、冲突解决 | system-architecture.md, database-schema.md |
| [rendering-protocol.md](03-protocols/rendering-protocol.md) | 渲染协议：JSON Schema格式、组件定义、布局约束 | system-architecture.md |

---

### 04-modules/ - 模块层

各功能模块的详细规范。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [weather.md](04-modules/weather.md) | 天气系统：和风API集成、IP定位、缓存策略、MQTT消息 | pairing-protocol.md |
| [todo.md](04-modules/todo.md) | 待办系统：数据模型、Tools接口、离线支持 | sync-protocol.md |
| [date.md](04-modules/date.md) | 日期系统：RTC管理、农历算法、节日数据 | - |
| [calendar.md](04-modules/calendar.md) | 日历系统：日程管理、提醒机制 | - |
| [countdown.md](04-modules/countdown.md) | 倒计时系统：目标日期计算、显示样式 | - |
| [battery.md](04-modules/battery.md) | 电量系统：监控、预估、低电量警告 | hardware-architecture.md |
| [sync.md](04-modules/sync.md) | 同步模块：设备端同步逻辑、队列管理 | sync-protocol.md |
| [ui-display.md](04-modules/ui-display.md) | UI显示：屏幕状态栏、全屏提示、配对模式显示 | rendering-protocol.md |
| [state-machine.md](04-modules/state-machine.md) | 状态机：设备状态定义、转换规则、性能指标 | interaction-input.md |

**注**：其他功能模块（todo/date/calendar/countdown/battery）待从PRD和tools-spec中提取完善。

---

### 05-hardware/ - 硬件层

硬件交互规范，包括输入（按钮）和输出（LED、扬声器）。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [interaction-input.md](05-hardware/interaction-input.md) | 输入交互：按钮操作、时序定义、防抖动、双击检测 | - |
| [interaction-output.md](05-hardware/interaction-output.md) | 输出交互：LED PWM控制、7个基础提示音、音量控制 | interaction-input.md |
| [bom.md](05-hardware/bom.md) | 硬件清单：物料明细、成本估算 | hardware-architecture.md |

---

### 06-interface/ - 接口层

后端对外暴露的接口规范。

| 文档 | 说明 | 依赖 |
|------|------|------|
| [mcp-tools.md](06-interface/mcp-tools.md) | MCP Tools规范：所有Tools定义、参数、示例 | system-architecture.md, 各功能模块 |

---

## 文档类型标识

每个文档头部包含元信息：

```yaml
---
title: 文档标题
description: 简要描述
type: 文档类型
scope: [涉及范围]
dependencies: [依赖文档]
---
```

**文档类型**：
- `product` - 产品文档（PRD、Roadmap）
- `architecture` - 架构文档（系统、数据、硬件）
- `protocol` - 协议文档（通信约定）
- `module` - 模块文档（功能详细规范）
- `hardware` - 硬件文档（硬件交互与实现）
- `interface` - 接口文档（后端对外接口）

---

## 快速导航

### 按角色查找

**产品经理**：
1. 01-product/PRD.md - 了解产品需求
2. 01-product/roadmap.md - 查看开发计划

**架构师**：
1. 02-architecture/system-architecture.md - 系统整体架构
2. 02-architecture/database-schema.md - 数据层设计
3. 03-protocols/*.md - 通信协议设计

**后端开发**：
1. 02-architecture/system-architecture.md - 了解后端架构
2. 02-architecture/database-schema.md - 数据库设计
3. 06-interface/mcp-tools.md - Tools接口实现
4. 04-modules/*.md - 各功能模块实现

**嵌入式/硬件开发**：
1. 02-architecture/hardware-architecture.md - 硬件架构
2. 05-hardware/interaction-input.md - 按钮输入处理
3. 05-hardware/interaction-output.md - LED/扬声器输出
4. 03-protocols/pairing-protocol.md - 设备配对实现
5. 03-protocols/sync-protocol.md - 同步机制实现
6. 04-modules/state-machine.md - 设备状态管理

**全栈/WebUI**：
1. 02-architecture/system-architecture.md - 了解Layer 0
2. 04-modules/weather.md - 天气配置界面参考

---

## 文档维护规范

### 命名规范

- **目录**：`NN-名称/`（NN为两位数字序号，保证排序）
- **文件**：`kebab-case.md`（短横线连接的小写字母）
- **版本文档**：`YYYY-MM-DD-文档名.md`（仅PRD、Roadmap等主文档）

### 更新流程

1. **添加新文档**：在对应目录创建，更新本文档地图
2. **修改文档**：更新文档头部的`updated`时间戳
3. **删除文档**：移动到`archive/`目录，保留原始文件，更新本文档
4. **文档拆分**：保留原文件作为索引，指向新拆分的文档

### 交叉引用

文档内引用其他文档时使用相对路径：

```markdown
详见 [数据库设计](../02-architecture/database-schema.md)
参见 [配对协议](pairing-protocol.md#MQTT主题)
```

---

## 变更记录

| 日期 | 变更内容 |
|------|----------|
| 2026-03-13 | 文档重组：按层次重新组织目录结构，拆分interaction-spec.md为3个文档，移动并重命名所有规范文档 |

---

*本文档维护者：AI便利贴项目团队*
