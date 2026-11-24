# 示例目录说明（生态贡献）

本目录提供一组**演示接口用、非真实案例**的最小调用示例，用于说明输入/输出契约与字段含义。

- `example_input.json`：虚构输入，含 `case_id` 与 `note`（注明"演示接口用，非官方案例"）。
- `example_output.json`：虚构输出，含 `case_id`、`status`、`artifact`（entities/relations）、`evidence_bindings`、`merge_decisions`、`conflicts`、3 个 `queries`、`uncertainties`、`human_review_items`、`change_log`、`self_check`，并标注为示意。

**重要**：示例中的论文、作者、数值、DOI 均为占位/虚构，不代表任何真实论文结论，也不构成预制作答答案或 benchmark 分数。实际输入中 primary/secondary 为两篇独立论文，位置为 JATS 段落 ID 而非 PDF 页码，根 DOI/作者不得互套。正式案例与白板由平台提供。请配合 `references/entity_relation_rules.md` 阅读以正确填写字段。
