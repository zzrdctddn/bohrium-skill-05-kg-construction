---
name: evidence-constrained-kg-builder
description: 从一批科研论文抽取实体与关系，为每个节点和每条边绑定原文证据与来源定位，正确处理跨论文别名归一与冲突，并支持至少 3 个可回原文的查询。当用户要求"构建文献知识图谱""抽取实体关系""别名归一/实体对齐""绑定证据""查询图谱""增量加入论文"时使用。输出含实体表、关系表、证据绑定、对齐与冲突记录、查询结果子图与结构化 result.json。
version: 1.0.0
license: MIT
---

# 文献知识图谱构建：证据约束的图谱构建（Evidence-Constrained Graph Construction, ECGC）

> 面向真实论文集合，把"画一张关系图"从凭常识连边，变成**每个节点和每条边都能回到论文具体位置的证据工程**。核心不是图好看，而是让每个实体和关系回答：抽得准吗？连得对吗？对得齐吗？查得到、回得去吗？

## 1. 用途与适用边界

**解决什么**：基础 Agent 构建文献图谱的常见失败——实体是模型凭常识补出的概念而非论文真实对象、把"综述转述/共现/推测"当直接关系、别名缩写大小写处理成重复节点或错误合并、把两篇论文的条件/样本/数值揉成一条结论、节点和边回不到原文位置、查询答不上或答了没证据。本 Skill 用"证据约束 + 来源绑定 + 受控对齐 + 可回查查询"消除这些失败。

**触发场景**：从一批论文抽取研究者/材料/方法/任务/指标/结论实体；建立有证据支持的关系；处理别名/缩写/大小写/同名冲突；为每个节点和边绑定来源；执行至少 3 个真实查询并给出证据；新增论文时保留既有节点与编号。

**不该用**：论文无可读正文且不可访问时（报告材料缺口，不凭标签补造实体）；要求"完整通用 KG 系统/自动实体链接服务"时（本 Skill 只做证据约束的可回查最小成果）；只做单篇摘要结构化时（转用文献摘要 Skill）。

**输入**：`/input/case.json` 与 `task.md`。材料可能包含 `corpus-manifest.json`（各论文 DOI、作者、来源哈希、许可证）、`material.json`（正文段落汇总索引）、`primary-material.json`（原论文）、`secondary-material.json`（新增论文）。**输出**：`/app/output/result.json`（字段见第 4 节）+ 可编辑的 `knowledge_graph.md`。

## 2. 核心方法（ECGC = Evidence-Constrained Graph Construction）

本 Skill 的方法名为 **ECGC = Evidence-Constrained Graph Construction**，四个不可跳过的机制逐一对应四个关键问题：

1. **机制 E｜证据约束抽取（Evidence-Constrained Extraction）→ 抽得准**。实体必须是论文里真实出现的科研对象（研究者、材料、方法、任务、指标、结论），保留原文位置与表述原文（`source_id` + `paragraph_id` + 原文短引），区分作者自述与他人转述；**禁止凭常识补全概念**。
2. **机制 C｜受控关系抽取（Controlled Relation Extraction）→ 连得对**。只建立有证据支持的关系，关系类型（使用/比较/优于/支持/矛盾等）、方向与适用条件与原文一致；**禁止把共现、综述转述、推测当直接证据**。
3. **机制 G｜受控实体对齐（Governed Entity Alignment）→ 对得齐**。处理别名、缩写、大小写、全称简称、同名冲突：给合并依据，既不产生重复节点，也不把不同对象错误合并。**不能强迫每个别名都合并；不能把不同论文的条件、样本、数值结果合成一个结论；相同通用方法/材料术语不代表两篇用了相同物理样品、模型实例、数据集或实验条件。**
4. **机制 CQ｜来源绑定与可回查查询（Source-Bound Queryable Graph）→ 查得到、回得去**。每个节点和每条边保留 `source_id`、`paragraph_id`、原文短引与证据强度；提供至少 3 个**实际执行的**查询，给出结果子图与支撑证据，无法确定的关系明确列出。

另外贯穿一条铁律：**抽取与核验分离**——先构建图，再以"审阅者"身份独立执行第 6 节自检，两遍不可合并。

> **输入修订特别约定（务必遵守）**：现在提供的是两篇独立、真实、许可可再分发的论文文本，不是把同一篇的 JSON 与 Markdown 当两篇。根 DOI/title/authors 属于第一篇，不得套到第二篇。位置是组织方给的 **JATS 段落 ID，不是实际 PDF 页码**；本包无 PDF 页码，不得把自建"第 1 页"称为原 PDF 页。正文行内公式/上下标被线性化，原 `secondary-publisher.xml` 留供核对；不提供图像像素、补充附件、底层临床/实验数据，不推断其中数值。**同篇多种文件格式不算跨篇证据。**

## 3. 工作流（SOP）

严格按顺序执行，每步都有检查点；检查点不过则回退该步。

**Step 0 · 输入解析与语料登记**
- 读 `corpus-manifest.json` 与 `task.md`，为每篇论文登记独立 `source_id`、DOI、作者、来源哈希、许可证；确认 primary/secondary 是两篇独立论文。
- 检查点：根 DOI/作者是否误套到第二篇？是否把同篇多格式当跨篇？若是，纠正并记 `change_log`。

**Step 1 · 实体抽取（机制 E）**
- 逐段抽取研究者、材料、方法、任务、指标、结论等实体；每个实体记 `entity_id`、原文表述、类型、`source_id`、`paragraph_id`、原文短引；标 `self_reported`（作者自述）或 `paraphrased`（他人转述）。
- 检查点：实体是否在论文中真实出现？是否凭常识补了概念？位置是否为 JATS 段落 ID 而非自建页码？缺证据的实体标 `unverified`。

**Step 2 · 关系抽取（机制 C）**
- 只在有原文证据时建边；为每条边记关系类型（受控词表）、方向、适用条件、`source_id`、`paragraph_id`、原文短引、证据强度。
- 检查点：是否把共现/转述/推测当直接关系？方向是否与原文一致？不满足则不建边，进 `uncertainties`。

**Step 3 · 实体对齐与冲突处置（机制 G）**
- 对跨篇候选同一概念，逐对给出合并依据（全称=缩写、大小写规范、同 DOI 等）；记录"不得合并"的冲突（不同对象、不同条件/样本/数值）。
- 检查点：是否强迫所有别名合并？是否把不同论文条件/样本/数值揉成一条？保留 `merge_decisions` 与 `conflicts`，合并/不合并都要有依据。

**Step 4 · 来源绑定与证据强度**
- 为每个节点和每条边绑定 `source_id` + `paragraph_id` + 原文短引，标注证据强度（strong/weak）。
- 检查点：每个实质节点/边是否都有独立论文的来源与短引？同篇多格式是否被误当跨篇证据？

**Step 5 · 查询执行（机制 CQ）**
- 设计并执行至少 3 个真实查询（如"方法 X 使用了哪些材料？""哪些结论与 Y 矛盾？""指标 Z 在两篇中如何被报告？"），给出结果子图节点/边 ID、支撑证据与无法确定项。
- 检查点：查询是否真的在图上执行并回填证据？无结果查询是否如实说明而非编造？

**Step 6 · 增量更新（可选）与独立核验**
- 新增论文时保留既有节点与编号，说明新增/合并/冲突处置；以审阅者身份逐项走 `references/checklist.md`。
- 写 `knowledge_graph.md`（可编辑）+ `result.json`（结构化结果，含 `case_id` 与全部输出契约字段）。

## 4. 输出契约（result.json）

```json
{
  "case_id": "<原样带回>",
  "status": "success | partial | blocked",
  "artifact": {
    "entities": [
      {"entity_id": "E1", "name": "<原文表述>", "type": "researcher|material|method|task|metric|conclusion", "source_id": "<论文source_id>", "paragraph_id": "<JATS段落ID>", "quote": "<原文短引>", "mention_type": "self_reported|paraphrased", "evidence_strength": "strong|weak"}
    ],
    "relations": [
      {"relation_id": "R1", "subject": "E1", "predicate": "uses|compares|outperforms|supports|contradicts|other", "object": "E2", "condition": "<适用条件>", "source_id": "<...>", "paragraph_id": "<...>", "quote": "<原文短引>", "evidence_strength": "strong|weak"}
    ]
  },
  "evidence_bindings": [
    {"node_or_edge_id": "E1|R1", "source_id": "<...>", "paragraph_id": "<...>", "quote": "<...>", "evidence_strength": "strong|weak"}
  ],
  "merge_decisions": [
    {"candidates": ["E1", "E5"], "decision": "merge|keep_separate", "basis": "<合并/不合并依据>"}
  ],
  "conflicts": [
    {"entities_or_relations": ["R1", "R7"], "description": "<冲突描述>", "handling": "<保留/标注方式>"}
  ],
  "queries": [
    {"query": "<真实查询>", "result_subgraph": {"nodes": ["E1"], "edges": ["R1"]}, "supporting_evidence": ["<原文短引+位置>"], "undetermined": ["<无法确定项>"]}
  ],
  "uncertainties": ["<材料缺失/冲突/无法判断项>"],
  "human_review_items": ["<需人工确认的对齐/冲突项>"],
  "change_log": [{"step": 3, "action": "...", "reason": "..."}],
  "self_check": {
    "entities_grounded": true,
    "relations_evidenced": true,
    "alignment_justified": true,
    "queries_with_evidence": true
  }
}
```

字段原则：`status` 诚实反映完成情况；`partial` 必须写清 `uncertainties`；无证据的实体/关系不写成事实，进 `uncertainties`；不得编造来源元数据、数值或 benchmark 分数；材料缺失、无法判断、运行失败分别说明。

## 5. 质量自检清单

执行前完整读取 `references/checklist.md`，逐项打钩。关键硬项：

- [ ] 实体均为论文真实对象，保留原文表述与 JATS 段落位置
- [ ] 区分作者自述/他人转述，未凭常识补概念
- [ ] 关系类型/方向/条件与原文一致，未把共现转述推测当直接证据
- [ ] 别名/缩写/大小写/同名冲突处理有依据，未强迫合并、未误并不同对象
- [ ] 未把不同论文的条件/样本/数值合成一个结论
- [ ] 每个节点和边都能回到 source_id + paragraph_id + 原文短引
- [ ] 至少 3 个实际执行的查询，附证据与无法确定项
- [ ] 抽取与核验分离，所有改动有 change_log

## 6. 异常处理与边界

| 情况 | 处理 |
|---|---|
| 文献缺失/PDF 无法解析 | `status=blocked` 或 `partial`，说明缺口，不凭标签补造实体 |
| 实体同名/别名冲突 | 给出合并依据；依据不足则 `keep_separate` 并进 `human_review_items` |
| 关系证据冲突 | 不强行建边，记入 `conflicts`，进人工复核 |
| 查询无结果 | 如实返回空子图并说明，不编造关系 |
| 跨篇"共同术语" | 不据此推断相同样品/模型/数据集/条件，标 `keep_separate` |
| 行内公式/上下标线性化 | 保留线性化原文，注明核对 `secondary-publisher.xml` |
| 无 PDF 页码 | 一律用 JATS 段落 ID，不自建页码 |
| 根元数据误套第二篇 | 逐篇独立登记，纠正并记 change_log |

## 7. 最小调用示例（示意，非真实案例）

**输入**（`case.json` 片段）：
```json
{"case_id":"DEMO-NOT-A-REAL-CASE","task":"从两篇论文构建证据约束图谱并执行查询","corpus":["primary-material.json","secondary-material.json","corpus-manifest.json"],"queries":["方法X使用了哪些材料?","指标Z在两篇中如何报告?"]}
```
**调用**：加载本 Skill 后，用户说"用 evidence-constrained-kg-builder 构建这两篇的图谱"。

**示意输出**（节选）：
```json
{"case_id":"DEMO-NOT-A-REAL-CASE","status":"success","artifact":{"entities":[{"entity_id":"E1","name":"（示意）方法X","type":"method","source_id":"paper-1","paragraph_id":"p-12","quote":"（示意短引）","mention_type":"self_reported","evidence_strength":"strong"}],"relations":[{"relation_id":"R1","subject":"E1","predicate":"uses","object":"E2","source_id":"paper-1","paragraph_id":"p-12","quote":"（示意短引）","evidence_strength":"strong"}]},"queries":[{"query":"（示意）方法X使用了哪些材料?","result_subgraph":{"nodes":["E1"],"edges":["R1"]},"supporting_evidence":["（示意）p-12 短引"],"undetermined":[]}],"self_check":{"entities_grounded":true,"relations_evidenced":true,"alignment_justified":true,"queries_with_evidence":true}}
```
> 上述为演示接口用的虚构示例，不代表任何真实案例结果；示例中的论文、作者、数值均为虚构。

## 8. 资源清单

- `references/method.md` —— ECGC 方法细则与反例库
- `references/checklist.md` —— 逐项自检清单（覆盖抽得准/连得对/对得齐/查得到回得去）
- `references/schema.json` —— 输出字段字典（draft-07）
- `references/entity_relation_rules.md` —— 实体/关系类型定义、证据强度、别名对齐与冲突处置规则
- `assets/kg_template.md` —— 图谱节点/边/查询可编辑模板
- `examples/` —— 最小调用示例（标注为示例，非真实案例）
