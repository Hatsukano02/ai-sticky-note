# AI便利贴 - Agent 协作指南

## 项目概述

这是一个AI-powered电子墨水屏便利贴硬件项目的文档仓库。主要包含产品需求、技术规范、架构设计等Markdown文档。

## 文档规范

### 文件命名

- **主文档**: `YYYY-MM-DD-文档名.md` (如: `2026-03-13-PRD.md`)
- **技术规范**: 放在 `docs/spec/` 目录，无需日期前缀
- **复盘记录**: `docs/postmortem/YYYY-MM-DD-简述.md`
- **归档文档**: `docs/archive/YYYY-MM-DD-原文件名.md`

### Markdown 格式

- 使用 **GitHub Flavored Markdown**
- 标题层级: `#` (文档标题) → `##` (章节) → `###` (小节)
- 列表统一使用 `-` (无序) 或 `1.` (有序)
- 代码块标注语言: `json`, `cpp`, `python`, `yaml`
- 表格用于对比数据，保持列对齐

### 中文写作规范

- 使用简体中文
- 技术术语保留英文: JSON, API, WebSocket, ESP32
- 标点符号使用中文全角
- 数字与单位之间不加空格: `400x300`, `2-4周`

## 文档结构

```
docs/
├── README.md              # 目录导航
├── YYYY-MM-DD-PRD.md      # 产品需求文档
├── YYYY-MM-DD-roadmap.md  # 开发路线图
├── hardware-bom.md        # 硬件物料清单
├── spec/                  # 技术规范
│   ├── server-architecture.md
│   ├── rendering-spec.md
│   ├── tools-spec.md
│   ├── interaction-spec.md
│   └── offline-sync-spec.md
├── postmortem/            # 问题复盘
└── archive/               # 废弃方案归档
```

## 编辑检查清单

修改文档前确认:

- [ ] 概念一致性 (如: Tools vs Skill, Agent vs Router)
- [ ] 链接有效性 (内部锚点、图片路径)
- [ ] 版本更新 (修改日期、版本号)
- [ ] 相关文档同步更新

## 禁止事项

- ❌ 直接删除文档 → 移动到 `docs/archive/`
- ❌ 硬编码硬件参数到通用框架 → 保持抽象，具体值放配置
- ❌ 混用中英文标点 → 统一中文全角
- ❌ 大段ASCII艺术 → 使用简洁的代码块或文字描述

## 协作流程

1. **读取**: 先读相关文档了解上下文
2. **规划**: 复杂修改先简述计划
3. **执行**: 单文件修改，保持原子性
4. **验证**: 检查链接、格式、概念一致性
5. **提交**: 清晰描述修改原因

## 技术术语表

| 术语 | 含义 |
|------|------|
| MCP | Model Context Protocol |
| Tools | Pydantic AI 可调用的功能单元 |
| Registry | 页面配置注册表 |
| EPD | Electronic Paper Display (墨水屏) |
| VAD | Voice Activity Detection (语音活动检测) |
| ASR | Automatic Speech Recognition (语音识别) |

## 联系方式

项目问题在 GitHub Issues 讨论，或联系项目维护者。

---

*本文档指导 AI 助手在此仓库中高效协作。*
