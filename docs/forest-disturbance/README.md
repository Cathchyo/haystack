# 森林扰动智能解译

本目录用于维护森林扰动监测、数据口径和智能周报相关的产品与技术文档。

## 文档索引

| 编号 | 文档 | 内容 |
|---|---|---|
| 00 | [周报系统 PRD](./00-weekly-report-prd.md) | 产品目标、范围、功能需求和验收标准 |
| 01 | [数据图层语义](./01-data-layer-semantics.md) | `slgr_confirm`、`slgr_detected` 和 `slgr_detected_rec` 的业务含义与使用规则 |
| 02 | [输入数据规范](./02-input-data-contract.md) | 原始字段结构、标准字段映射、日期与空间处理规则 |
| 03 | [周报事实与分析规则](./03-weekly-report-facts.md) | 从周报目标反推必须计算的数据及分析规则 |
| 04 | [周报模板](./04-weekly-report-template.md) | 最终 Markdown 周报的固定结构和写作规则 |
| 05 | [知识库规范](./05-knowledge-base.md) | 知识库内容、边界、条目要求和模型使用规则 |

## 文档约定

- 文件按 `编号-主题.md` 命名，编号用于表示推荐阅读顺序。
- 数据口径变更时，先更新对应数据语义文档，再同步更新 PRD 和处理代码。
- “算法确认”与“人工/现场核实”必须使用不同术语，不能混用。
- 所有新增字段或状态都应记录来源、判定规则和适用范围。

## 推荐阅读顺序

1. [数据图层语义](./01-data-layer-semantics.md)
2. [输入数据规范](./02-input-data-contract.md)
3. [周报事实与分析规则](./03-weekly-report-facts.md)
4. [周报模板](./04-weekly-report-template.md)
5. [知识库规范](./05-knowledge-base.md)
6. [周报系统 PRD](./00-weekly-report-prd.md)
