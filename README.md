# 论文汇总

| 论文名 | 会议/期刊 | 介绍 | 评论 | 代码链接 |
| --- | --- |  --- | --- | --- |
| Text-to-CAD Generation with Chain-of-Thought and Geometric Reward | NeurIPS 2025 | CAD-Coder 将 Text-to-CAD 从“预测低层 CAD 命令序列”改写为“生成可执行的 Python/CadQuery 程序”，再用 SFT + CoT 冷启动 + GRPO 几何奖励同时优化代码有效性与 3D 几何准确性。  | 1.CAD-Coder 没有真正的 Agent。2.输入确实基本只有文本；3.训练分布主要还是被 Sketch + Extrusion 限制 |  |
| CADDesigner: Conceptual CAD Model Generation with a General-Purpose Agent | Computer-Aided Design 2026（ 中科院三区，if=3 ) | CADDesigner 将概念 CAD 建模组织为 ReAct Agent，通过需求细化、CAD 代码生成、执行和多视图视觉反馈形成闭环，并结合结构化知识库与显式 CAD 表示 ECIP 提升代码生成和模型修正能力。 | 1.实验主要基于 DeepCAD，数据分布相对单一，泛化性仍需进一步验证；2.多轮 Agent 推理存在延迟，制造、公差和装配等工程约束不足。 | [项目主页](https://562590763.github.io/CADDesigner/) |

# 数据集


# 指令
## 生成md
这篇文章你写一个阅读笔记.md，包括introduction写出现有方法的局限以及本文创新点；2.method总分方式介绍方法；，要包括整体框架图，并进行介绍3.results介绍实验设置，实验结果与精炼分析，当前模型的局限性分析（如果有）；4.conclusion展示结果与局限。内容请按照论文中的来，可以适当补充但不要编写没有的东西。整篇文章要精炼抓住重点
