# 文献知识图谱构建 Skill（evidence-constrained-kg-builder）

一个把论文集合转成"带证据、可回查、可查询图谱"的 Agent Skill。核心：**证据约束抽取 + 受控关系抽取 + 受控实体对齐 + 来源绑定与可回查查询 + 抽取核验分离**（ECGC）。

## 目录结构
- `SKILL.md` —— Skill 本体（YAML frontmatter、工作流、输出契约）
- `references/` —— ECGC 方法细则、自检清单、字段 schema、实体/关系规则
- `assets/` —— 图谱可编辑模板
- `examples/` —— 最小调用示例（已标注为示例，非真实案例）
- `GITHUB_URL.txt` —— 提交时填 GitHub 公开链接（当前为 REPLACE_ME 占位）

## 使用方式
1. 将本目录作为 Skill 安装到 Agent（平台约定路径 `/skills/weekly-skill`）。
2. 输入 `/input/case.json` 与 `task.md`，下指令如"从这两篇论文构建图谱并回答查询"。
3. 按 SOP 执行：语料登记 → 实体抽取 → 关系抽取 → 对齐冲突处置 → 来源绑定 → ≥3 个查询 → 核验。
4. 输出到 `/app/output/result.json` 与 `knowledge_graph.md`。

## 依赖
仅需文本处理能力与 `node`（JSON 校验）；无外部图数据库依赖。实体/关系类型与对齐规则内置于 `references/`。

## 适用与限制
- 适用：多论文实体关系抽取、别名归一与冲突处置、证据绑定、可回查查询、增量更新。
- 限制：两篇论文必须按独立来源登记，根元数据不得互套；位置用 JATS 段落 ID，不自建 PDF 页码；无证据的实体/关系不写成事实；不编造数据、来源与 benchmark 分数。

## Runtime contract
Skill 加载到 `/skills/weekly-skill`，输入 `/input/case.json` 与 `task.md`，输出到 `/app/output/result.json`（含全部输出契约字段与 `case_id`）。资源 ≤50 个 UTF-8 文本文件、单文件 ≤500KB、总计 ≤2MB、无软链。执行预算 64 次模型请求 / 900 秒 / 每次 16384 token。输入修订：primary/secondary 为两篇独立论文；`corpus-manifest.json` 列 DOI/作者/哈希/许可证；位置为 JATS 段落 ID 非 PDF 页码；不推断补充附件/底层数据中的数值。

## 评分对应
实际效果(40)、使用增益(25)、创新增益(15)、可靠体验(10)、生态贡献(10) 的支撑材料分别见 SKILL.md 第 3/2/2/6/8 节与 `references/`；对照由平台用固定案例与白板完成，作品不自带答案与分数。
