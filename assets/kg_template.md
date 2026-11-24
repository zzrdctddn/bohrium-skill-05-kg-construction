# 文献知识图谱（可编辑模板）

> 用后删除本说明。方括号为占位。位置用 JATS 段落 ID，非 PDF 页码。

## 语料登记
- 论文1：`source_id=[paper-1]`，DOI=[…]，作者=[…]，许可证=[…]
- 论文2：`source_id=[paper-2]`，DOI=[…]，作者=[…]，许可证=[…]
- 注意：根 DOI/作者属于论文1，不得套到论文2；同篇多格式不算跨篇证据。

## 实体表
| entity_id | name（原文表述） | type | source_id | paragraph_id | 原文短引 | mention_type | 证据强度 |
|---|---|---|---|---|---|---|---|
| E1 | […] | [method] | [paper-1] | [p-12] | [……] | self_reported | strong |
| E2 | […] | [material] | [paper-2] | [p-5] | [……] | paraphrased | weak |

## 关系表
| relation_id | subject | predicate（受控） | object | 条件 | source_id | paragraph_id | 原文短引 | 证据强度 |
|---|---|---|---|---|---|---|---|---|
| R1 | E1 | uses | E2 | […] | [paper-1] | [p-12] | [……] | strong |

## 对齐与冲突
| 候选 | 决策 | 依据 |
|---|---|---|
| [E1, E5] | [merge/keep_separate] | [……] |

冲突：[R1 vs R7] 描述=[…] 处置=[…]

## 查询（至少 3 个，实际执行）
1. 查询：[……] → 子图节点=[…]，边=[…]，证据=[短引+位置]，无法确定=[…]
2. 查询：[……] → …
3. 查询：[……] → …

## 待人工确认
- [对齐/冲突/弱证据项逐条列出]
