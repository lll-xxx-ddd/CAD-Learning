# CADDesigner: Conceptual CAD Model Generation with a General-Purpose Agent — 阅读笔记

> **论文**：Fengxiao Fan, Jingzhe Ni, Xiaolong Yin, Sirui Wang, Xingyu Lu, Qiang Zou, Ruofeng Tong, Min Tang, Peng Du.  
> **版本**：发表于 *Computer-Aided Design*, Vol. 198, 104087, 2026。  
> **核心问题**：如何让通用大模型 Agent 从**抽象文本 / 草图**出发，经需求澄清、CAD 代码生成、执行与视觉反馈，稳定地产生可编辑的概念 CAD 模型。

## 一句话总结

CADDesigner 的关键不是再训练一个专用 CAD 模型，而是把 CAD 建模组织成一个 **ReAct Agent + LLM 友好的显式 CAD 表示（ECIP）+ 结构化知识库 + 执行/视觉闭环纠错** 的系统，使通用大模型能在较复杂、需求不完整的概念设计场景中持续生成和修正 CAD 代码。

---

# 1. Introduction

## 1.1 现有方法的局限

论文将已有方案的主要问题概括为四类：

1. **传统 CAD 建模门槛高、迭代成本大**  
   OnShape、AutoCAD、SolidWorks、CATIA 等通常依赖人工草绘、拉伸和逐步建模，要求用户掌握专业 CAD 操作；在早期概念设计中，需求本身又常常是不完整、抽象的，因此从想法到参数化模型的转换较慢且容易出错。

2. **基于参数化命令序列的学习方法表达能力受限**  
   DeepCAD、SkexGen 等方法主要依赖监督学习生成 CAD 命令序列，但受训练数据多样性和输出表示能力限制，通常偏向 sketch/extrude 等基础操作，对复杂建模流程的泛化不足。

3. **微调 LLM/VLM 的 CAD 方法成本高且受数据瓶颈制约**  
   现有 LLM/VLM 方法能从文本、图像等输入生成 CAD 代码，但微调开源大模型需要较大 GPU 资源；同时高质量、多样化 CAD 训练数据稀缺，限制了代码多样性和生成质量。

4. **现有 CAD SDK 并非为 LLM 代码生成设计**  
   - CadQuery 的 Fluent API 依赖**隐式上下文/内部状态传递**，LLM 需要在长链式调用中持续追踪中间状态；
   - build123d 虽然上下文更显式，但大量依赖 `@`、`+`、`-` 等符号运算，复杂操作的语义对 LLM 不够透明。  
   因而，LLM 容易在状态依赖、类型、参数和操作语义上产生歧义。

## 1.2 本文创新点

论文的创新可概括为两层：**Agent 框架**和**CAD 代码表示**。

### (1) CADDesigner：面向概念 CAD 的通用 Agent

- 支持**文本、草图及二者组合**作为输入；
- 通过对话式需求分析，将粗粒度用户意图细化为可建模的设计描述；
- 集成需求细化、代码生成、命令执行、视觉反馈等工具；
- 利用执行错误和多视图渲染结果进行迭代修正；
- 成功案例可进入结构化知识库，为后续生成提供可复用经验。

### (2) ECIP：Explicit Context Imperative Paradigm

作者在 CadQuery 之上实现一层 LLM-oriented 的轻量 API（项目中称 **SimpleCADAPI**），核心思想是：

> **每一步 CAD 操作都显式传入对象和参数，并显式返回新的建模状态。**

相比 CadQuery 常见的链式隐式状态，ECIP 将中间对象变成明确变量，并进一步加入：

- 显式返回类型命名；
- 原子操作级结构化错误信息；
- 语义 Tag / Metadata 与几何查询机制；
- 可复用、自扩展的复合操作；
- 面向 RAG 的 API 文档与案例知识库。

**本质上，论文不是单纯提升 LLM 本身，而是重新设计“LLM 如何表达和操作 CAD”。**

---

# 2. Method

## 2.1 整体方法

CADDesigner 采用 **ReAct-style Agent**，由一个中央 CAD Orchestrator Agent 控制整个建模循环。它不是固定的 hard-coded workflow，而是由系统提示给出流程规则，Agent 在运行时根据当前状态选择工具，因此用户可以中途追加约束、修改目标或纠正方向。

### 论文整体框架图（Figure 2）

![CADDesigner overall framework](https://562590763.github.io/CADDesigner/files/figure2.png)

*Figure 2. CADDesigner 的 Intelligent CAD Orchestrator Agent。蓝色箭头表示 Action/Invocation，橙色虚线表示 Observation/Feedback。*

其主链路可以写成：

```mermaid
flowchart LR
   A[User Input] -->|T1| B[Detailed Design]
   B -->|T2| C[CAD Code]
   C -->|T3| D[CAD Model]
   D -->|T4¹| E[Multi-view Render]
```

随后视觉反馈模块根据渲染结果与目标设计给出通过/失败信号及诊断反馈；若失败，则反馈重新进入代码生成与执行阶段，形成闭环。

---

## 2.2 四个核心 CAD 工具

### T1：Requirement Refinement Tool

**输入**：文本和/或草图等初始信息。  
**输出**：包含具体参数的详细设计描述。

作用是解决概念设计输入“不完整、模糊”的问题：Agent 将粗略描述扩展为较明确的几何结构、尺寸和建模需求；用户也可以进一步修改，最终得到确认后的 \(D_{final}\)。

### T2：Code Generation Tool

将最终设计描述转换为可执行 CAD 代码。其生成不是纯自由生成，而是由**结构化知识库 \(K\)** 约束：

- **Anno**：API 函数语义、参数格式、返回类型、使用说明；
- **Case**：基础/历史建模案例。

因此代码生成可视为：

\[
G':(D_{final},K)\rightarrow C'
\]

代码执行失败时，结构化错误会返回 T2，指导下一轮代码重写。

### T3：Command Execution Tool

负责执行 Python CAD 脚本并生成 CAD 模型，同时返回：

- shell 日志 / 错误；
- 几何元数据，如体积、尺寸、拓扑相关信息。

这些信息不仅用于“代码能否运行”的判断，也能为尺寸和约束验证提供结构化信号。

### T4：Visual Feedback Tool

先从模型生成 **6 个视角**：Front-Top-Left、Back-Bottom-Right、Back-Top-Left、Front-Bottom-Right、Top、Right。

随后包含两个子步骤：

1. **Visual Question Generation**：依据目标设计和多视图图像生成针对性检查问题；
2. **Visual Feedback Generation**：回答这些问题，并输出：
   - \(P\in\{0,1\}\)：是否终止；
   - \(F\)：结构化视觉诊断反馈。

若 \(P=0\)，反馈返回代码生成阶段继续修正；若 \(P=1\)，模型交给用户最终确认。

---

## 2.3 ReAct Agent 与辅助工具

中央 Agent 维持两类核心状态：

- **Working Memory & Context State**：当前目标、历史日志、API 知识、最近错误等；
- **Reasoning & Planning Engine**：在“Observe → Analyze → Plan → Action”循环中决定下一步工具调用。

另外有两个辅助工具：

- **SketchPad**：以 key-value 方式保存图片路径、代码片段、执行结果等中间信息，避免把大量历史内容反复塞入上下文；支持自动摘要、标签搜索和过期清理。
- **File Operation Tools**：负责模型流程中的文件读写与引用管理。

---

## 2.4 ECIP：显式上下文命令式 CAD 表示

CadQuery 常见写法可抽象为隐式状态更新：

\[
C_{k+1}=f_k(C_k,p_k)
\]

其中上下文 \(C_k\) 常隐藏在链式调用内部。ECIP 则要求：

\[
S_{k+1}=g_k(S_k,p_k)
\]

并把每个 \(S_k\) 显式保存为变量。这样 LLM 不需要“猜测当前 workplane / shape 状态”，更容易插入普通 Python 控制流、定位错误和局部修改。

### 一个直观例子：底座上开圆孔

例如，要创建一个 80 × 50 × 10 的底座，并在顶部开一个直径为 12 的圆孔。使用 CadQuery 时，工作平面和当前对象主要通过链式调用隐式传递：

```python
import cadquery as cq

model = (
   cq.Workplane("XY")
   .box(80, 50, 10)
   .faces(">Z")
   .workplane()
   .hole(12)
)

cq.exporters.export(model, "base.step")
```

在这段代码中，`.faces(">Z").workplane()` 会切换到顶面工作平面，后面的 `.hole(12)` 依赖前面链式调用产生的上下文。对于较长的建模过程，LLM 需要持续追踪当前 workplane、对象类型和链式调用结果。

用 ECIP 表达时，可以把每一步的输入和输出都保存为显式变量。下面的函数名是帮助理解 ECIP 思路的简化示意，不代表论文中 API 的完整名称：

```python
from simplecad import *

base: Solid = create_box_rSolid(
   length=80,
   width=50,
   height=10,
)

top_face: Face = select_face_rFace(
   solid=base,
   query="face.normal == (0, 0, 1)",
)

hole_profile: Wire = create_circle_rWire(
   center=(40, 25),
   radius=6,
   plane=top_face,
)

hole_tool: Solid = extrude_rSolid(
   profile=hole_profile,
   direction=(0, 0, -1),
   distance=10,
)

result: Solid = boolean_cut_rSolid(
   target=base,
   tool=hole_tool,
)

export_step(result, "base.step")
```

此时建模状态的依赖关系是明确的：

```text
base -> top_face -> hole_profile -> hole_tool -> result
```

如果孔的位置错误，可以直接修改 `hole_profile`；如果布尔运算失败，也能定位到 `boolean_cut_rSolid` 的输入，而不必回溯整条调用链。因此，ECIP 的重点不是简单增加 API 数量，而是把原本隐藏在调用链中的上下文、中间结果和对象类型显式化。

### ECIP / SimpleCADAPI 的关键设计

- **Core Module**：封装 Vertex、Edge、Wire、Face、Shell、Solid、Compound 等几何对象，以及坐标系、工作平面、Tag 和 Metadata。
- **API Module**：提供 50+ CAD 函数，覆盖创建、变换、拉伸/旋转/放样/扫掠、布尔、圆角/倒角、阵列、选择与导出等操作。
- **Query Language (QL)**：基于语义标签、元数据和几何属性进行可序列化选择，降低“按索引选面/边”的脆弱性。

### LLM-friendly 设计

**1. 结构化错误**

\[
ErrMsg=(ErrCau,ErrLoc,CorrAct)
\]

即明确给出**错误原因、错误位置、建议修复动作**，帮助 Agent 在迭代中快速恢复。

**2. 显式返回类型**

函数命名采用类似：

```text
ActionName_rReturnType
```

例如函数名本身即可提示返回 `Solid`、`Wire` 等类型，降低几何对象类型混淆。

**3. Self-Evolving Composite Operations**

将一串原子 CAD 操作封装为更高层复合操作，并保存为后续可直接复用的新操作，例如螺钉、法兰等常见结构，从而积累建模知识。

---

## 2.5 Knowledge Base

知识库 \(K\) 通过 RAG 为 T2 提供准确 CAD 语义，重点解决 LLM 容易：

- 误解函数参数；
- 混淆返回类型；
- hallucinate 不存在的 API；
- 忽视特定几何约束。

论文对 API 文档采用结构化 Markdown 表示，并按二级标题进行**语义友好的 chunking**，每个 chunk 还追加 API 名、章节标签和代表性关键词作为 anchor，提高检索的完整性与相关性。

---

# 3. Results

## 3.1 实验设置

### 实现

- 硬件：8-core Intel CPU；Python 3.12；
- Agent 框架：作者定制框架；
- RAG：RAGFlow；
- Embedding：`bge-large-zh-v1.5`；
- 检索：向量相似度 0.6 + 关键词相似度 0.4；Top-k = 3；
- 主 Agent + T2 Code Generation：**Claude-4-Sonnet**；
- T1 Requirement Refinement + T4 两个视觉子模块：**Gemini-2.5-Flash**；
- temperature = 0.7，top-p = 0.9。

**重要实验边界**：所有实验都关闭了用户反馈环 \(R\) 和 \(Sat\)，因此实验结果**不包含用户确认后在线积累案例所带来的提升**。

### 数据集

基于 **DeepCAD**（约 178K 参数化 CAD 模型）：

- 先去除重复模型；
- 从训练集随机选 **1K** 模型构建初始 RAG Case 库，并转换为 ECIP 代码；
- **200-model 子集**：按 CAD 命令数量分层采样（1–10、11–20、21–30、31–40、41+），用于消融实验；
- **1K-model 子集**：从去重测试集随机采样，用于与已有 Text-to-CAD 方法比较；
- 每个模型提供固定视角渲染图；文本输入使用 DeepCAD 的 abstract-level descriptions。

### 指标

**几何质量**：

- IoU ↑
- Chamfer Distance (CD) ↓
- Hausdorff Distance (HD) ↓

评测前统一归一化到 \([-0.5,0.5]^3\)，用 ICP 对齐；IoU 体素大小为 0.02；CD/HD 从模型表面均匀采样 2048 点。

**生成过程**：

- Pass@1 ↑：第一次就生成有效代码的比例；
- AVG Re ↓：平均重试次数；
- SUC ↑：最终得到可执行、可渲染 CAD 的比例；
- Tokens、Latency：推理消耗与耗时。

> 注意：SUC 只代表“语法/执行有效”，不保证生成模型与输入语义一致。

---

## 3.2 核心实验结果

### 3.2.1 ECIP 设计消融

| 方法 | Pass@1 ↑ | AVG Re ↓ | SUC ↑ |
|---|---:|---:|---:|
| **ECIP** | **0.45** | **1.86** | **100.0%** |
| w/o structured error | 0.41 | 2.62 | 81.5% |
| w/o return type | 0.32 | 2.30 | 90.5% |

**精炼分析：**

- 去掉**结构化错误**后，首次生成下降不大，但重试显著增加、最终成功率降到 81.5%：说明它主要负责**错误后的快速恢复与收敛**。
- 去掉**显式类型信息**后，Pass@1 从 0.45 降到 0.32：说明类型设计主要改善**第一次生成时的语义准确性**。
- 两者作用互补：**Type 降低犯错概率，Error 提高犯错后的修复能力。**

### 3.2.2 Agent 组件与输入模态消融

| Variant | IoU ↑ | CD ↓ | HD ↓ |
|---|---:|---:|---:|
| w/o Case | 0.2650 | 0.1350 | 0.4690 |
| w/o T1 | 0.2726 | 0.1251 | 0.5595 |
| w/o T4 | 0.2522 | 0.1192 | 0.4793 |
| Text only | 0.3041 | 0.1236 | 0.4154 |
| Image only | 0.3518 | 0.1167 | 0.4262 |
| **Text + Image** | **0.3893** | **0.0693** | **0.2553** |

去掉 API Function Annotations（Anno）时，Agent 无法稳定识别可用操作及其使用方式，实验中无法生成有效模型。

**精炼分析：**

- Case、需求细化 T1、视觉反馈 T4 都能提升生成质量；其中去掉 T4 后 IoU 降幅最大，说明**视觉闭环对整体结构正确性非常关键**。
- Image-only 的 IoU/CD 优于 Text-only，说明抽象文本对几何细节描述不足，图像能提供更强空间信息；但其 HD 略差于 Text-only。
- **Text + Image 在三个几何指标上均最好**，说明文本和视觉具有明显互补性：图像提供几何结构，文本补充图像难以确定的语义/约束。

### 3.2.3 不同 LLM Backend

| Backend | IoU ↑ | CD ↓ | HD ↓ |
|---|---:|---:|---:|
| **Claude-4-Sonnet** | **0.3041** | **0.1236** | **0.4154** |
| Gemini-2.5-Pro | 0.2951 | 0.1571 | 0.4902 |
| GPT-5.2-Codex | 0.2914 | 0.1604 | 0.5021 |

三个后端整体差距不大，Claude-4-Sonnet 最好。作者据此认为，在能力相近的 LLM 间，ECIP + 知识约束框架能保持相对稳定的 CAD 代码生成表现，系统性能并非完全依赖单一底座模型。

### 3.2.4 ECIP vs. CadQuery vs. build123d

| Representation | IoU ↑ | CD ↓ | HD ↓ | Pass@1 ↑ | AVG Re ↓ | SUC ↑ | Tokens (M) ↓ | Latency (s) ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| **ECIP** | **0.3041** | **0.1236** | **0.4154** | 0.46 | **1.86** | **100.0%** | **0.98** | 436 |
| CadQuery | 0.2827 | 0.1818 | 0.5186 | 0.50 | 2.01 | 87.5% | 1.40 | 541 |
| build123d | 0.2617 | 0.1570 | 0.5126 | **0.59** | 1.91 | 96.0% | 1.35 | **363** |

**精炼分析：**

- ECIP 的几何质量、最终成功率和迭代收敛最好，同时 Token 量最低；
- build123d 首次执行成功率最高且延迟最低，但几何一致性较差；
- 说明“能生成可运行代码”与“生成几何正确的模型”并不等价；
- ECIP 的优势来自**代码表示和 LLM 可用性**，而不是换了不同几何内核——它本身仍是基于 CadQuery/OpenCASCADE 的轻量表示层。

### 3.2.5 与已有 Text-to-CAD 方法比较（1K）

| Method | IoU ↑ | CD ↓ | HD ↓ | SUC ↑ |
|---|---:|---:|---:|---:|
| Text2CAD | 0.1831 | 0.1475 | 0.5680 | 96.6% |
| cadrille | 0.0274 | 0.2162 | 0.5817 | 98.2% |
| CADCodeVerify | 0.2348 | 0.2329 | 0.4892 | 86.1% |
| **CADDesigner** | **0.2769** | **0.1097** | **0.4347** | **100.0%** |

CADDesigner 在该 1K abstract-text 测试中四项指标均最好。

论文特别强调：cadrille 的 SUC 很高，但可视化结果经常与输入语义无关，说明**语法正确 ≠ 设计意图对齐**。CADDesigner 的优势之一是其操作空间更丰富，支持 revolve、pattern 等高层 CAD 操作；对于旋转对称、重复结构等模型，比主要依赖 extrusion 的表示更容易得到忠实几何。

---

## 3.3 推理成本

随着 CAD 命令数量从 1–10 增长到 41+，Tokens 和 Latency 均持续增加，但论文观察到增长较平稳，没有突然爆炸。

按模块分解后：

- **T2 Code Generation 是 Token 与 Latency 的主要来源**；
- T1、T4 占比较小；
- T3 不消耗 LLM Token，但会贡献一定执行时间；
- 主 Agent 本身开销较小。

因此，系统当前的效率瓶颈主要集中在**代码生成阶段**。

---

## 3.4 当前模型局限性

论文 Section 5.7 明确给出两点主要局限：

1. **交互延迟仍然较高**  
   虽然复杂度增加时延迟增长相对可控，但在复杂模型和多轮迭代场景中，端到端响应时间仍不可忽略；其中 T2 是主要瓶颈。

2. **工业设计领域知识仍不足**  
   当前系统尚未充分建模真实工业设计中的：
   - manufacturing constraints；
   - tolerance requirements；
   - assembly semantics；
   - standardized engineering conventions。  
   因而，在特定工业领域的复杂模型生成与验证上仍需要进一步加强。

此外，从实验设置本身还应注意一个范围限制：**论文关闭了用户反馈与在线 Case 累积环节**，所以实验只验证了当前固定知识库条件下的生成能力，并未实证“持续使用后知识库自增长”能带来多大长期收益。

---

# 4. Conclusion

## 4.1 论文最终结论


实验表明：

- 完整 ECIP 的最终执行成功率达到 **100%**，结构化 Error 和 Type annotation 都有明确贡献；
- 文本 + 图像输入在 200-model 消融中达到 **IoU 0.3893 / CD 0.0693 / HD 0.2553**，明显优于单模态；
- 在 1K abstract-text 测试上，CADDesigner 达到 **IoU 0.2769 / CD 0.1097 / HD 0.4347 / SUC 100%**，优于 Text2CAD、cadrille 和 CADCodeVerify；
- ECIP 相比 CadQuery / build123d 表现出更好的几何一致性和最终生成可靠性。

## 4.2 结果与局限的综合判断

**优势**：CADDesigner 很适合**快速原型和早期概念设计**——此时用户意图通常不完整，交互式需求澄清和视觉反馈比单次生成尤其重要。

**局限**：主要问题是**多轮 Agent 推理带来的延迟**，以及**制造、装配、公差和工程规范等领域约束不足**。
- 数据集只用了DeepCAD结果不够泛化
---

# 总结
1. **CADDesigner 的提升不是只来自更强 LLM，而来自 Agent + CAD 表示层的共同设计。**
2. ECIP 提升没那么大，可以暂时不考虑
3. **视觉反馈 T4 是高质量建模的重要环节；Text + Image 的互补性非常明显。**

---

## 参考

- Paper (arXiv v6): https://arxiv.org/abs/2508.01031
- Project Page: https://562590763.github.io/CADDesigner/
- DOI: https://doi.org/10.1016/j.cad.2026.104087

> 本笔记以论文 v6 的正文、Figure 2、Table 2–6、Section 5.7 和 Conclusion 为主进行压缩整理；额外分析仅用于解释论文结果，不引入论文未支持的实验事实。
