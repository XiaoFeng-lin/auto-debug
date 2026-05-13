# auto-debug 文档索引

本文档目录包含 auto-debug 项目的全部开发、设计与使用文档。

---

## 文档地图

| 文档 | 路径 | 面向读者 | 说明 |
|------|------|----------|------|
| **产品需求文档（PRD）** | `spec/PRD.md` | 产品经理、设计者、开发者 | 功能规格、用户故事、验收标准、错误码、配置参数 |
| **PRD 评审报告** | `spec/PRD-review-report-v1.md` | 设计者、开发者 | PRD v1.0 的评审记录，含关键缺失、模糊点、技术可行性分析 |
| **实施计划** | `plan/IMPLEMENTATION_PLAN.md` | 开发者、项目经理 | 实施步骤（Phase 1~4）、验证清单、风险与缓解措施 |
| **用户指南** | `USER_GUIDE.md` | 最终用户 | 安装、调用方式、参数说明、使用示例、输出文件位置 |
| **运维手册** | `RUNBOOK.md` | 运维、技术支持 | 部署方式、常见问题、故障排除、版本管理与升级指南 |
| **开发记录** | `dev-log/` | 项目维护者 | 历次重大变更的详细记录（重构、设计决策、会话还原） |

---

## 快速定位

- **第一次使用 Skill** → 先看 [`USER_GUIDE.md`](USER_GUIDE.md)
- **遇到安装或运行问题** → 查阅 [`RUNBOOK.md`](RUNBOOK.md)
- **想修改或扩展 Skill** → 阅读 [`spec/PRD.md`](spec/PRD.md) 和 [`plan/IMPLEMENTATION_PLAN.md`](plan/IMPLEMENTATION_PLAN.md)
- **想了解某次重构的上下文** → 查看 [`dev-log/`](dev-log/)
