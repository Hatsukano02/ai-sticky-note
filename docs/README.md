# AI便利贴 文档目录

## 结构说明

```
docs/
├── 2026-03-13-PRD.md           # 产品需求文档（带日期版本）
├── 2026-03-13-roadmap.md       # 开发路线图（带日期版本）
├── hardware-bom.md             # 硬件物料清单
│
├── spec/                       # 技术规范（设计+实现）
│   ├── server-architecture.md  # 服务端架构
│   ├── rendering-spec.md       # 渲染框架规范
│   ├── tools-spec.md           # MCP Tools规范
│   ├── interaction-spec.md     # 交互规范
│   └── offline-sync-spec.md    # 离线同步规范
│
├── postmortem/                 # 复盘记录
│   └── README.md               # 复盘记录说明
│
├── archive/                    # 归档文档
│   └── README.md               # 归档说明
│
└── assets/                     # 图片等资源文件
```

## 目录用途

### 1. 日期前缀命名（借鉴）

主文档（PRD、roadmap）带日期前缀，方便追溯版本：
- `2026-03-13-PRD.md` - 初始版本
- 后续更新：`2026-03-20-PRD-v1.1.md`

### 2. spec/ 技术规范

详细的技术设计和实现规范，保持原有结构。

### 3. postmortem/ 复盘记录

记录踩过的坑、技术决策反思、问题解决方案。

模板：`YYYY-MM-DD-简短描述.md`

示例：
- `2026-03-15-墨水屏刷新失败原因.md`
- `2026-03-18-语音识别准确率优化.md`

### 4. archive/ 归档

废弃的方案、过时的文档，不直接删除以便追溯。

示例：
- `2026-03-10-废弃的LangChain方案.md`

---

*文档结构借鉴企业级项目最佳实践，适度简化适合小团队。*
