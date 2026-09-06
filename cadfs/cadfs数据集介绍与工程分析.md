# CADFS 数据集介绍与工程分析

> **分析对象**：CADFS 发布版的处理后核心数据（FeatureScript FP/RP、英文步骤标注、STEP）与辅助索引。  
> **本地统计日期**：2026-09-02。  
> **重点**：数据数量、文件格式、建模操作分布、跨模态对应关系和 15 种操作的真实案例。数据构建过程仅作背景说明，不在本文展开。

## 1. 一页读懂 CADFS

CADFS 是一个面向“文本/图像 → 可编辑 CAD 程序”的大规模真实世界数据集及配套框架。它不只保存最终三角网格，而是以 Onshape 的 FeatureScript 表达设计历史：草图、拉伸、旋转、圆角、倒角、抽壳、孔、布尔、阵列等操作及其参数、选择对象和依赖关系都保留在程序里。论文将规模概括为约 **450k/451k** 个真实 CAD 设计、覆盖 **15 种建模操作**；本地对正式压缩包逐条计数得到 **450,307 个 FeatureScript 文件**。

从工程角度，CADFS 最值得注意的不是“有 15 个类别”，而是以下四点：

1. **15 种操作是可共现的操作标签，不是互斥分类类别。** 每个设计通常由多个操作顺序构成。
2. **程序、自然语言和 B-rep 可以按同一模型 ID 对齐。** 例如 `00025852` 同时对应 `.txt` FeatureScript、英文步骤标注和 `.step` 几何文件。
3. **分布很长尾。** 全量 RP 数据中 Extrude 出现在 94.13% 的模型里，Fillet 为 23.15%，Boolean 为 1.28%，Delete Body 仅为 0.80%。
4. **它适合生成“可编辑设计历史”，但不等于所有样本都可无条件编译。** 两位小数版本可能因舍入改变几何约束；自动生成的文本标注也存在漏项、对象指代和格式异常。

官方入口：

- 论文：[CADFS: A Big CAD Program Dataset and Framework for Computer-Aided Design with Large Language Models](https://arxiv.org/abs/2605.01925)
- 项目页：[CADFS project page](https://voyleg.github.io/cadfs/)
- 数据：[VladPyatov/CADFS on Hugging Face](https://huggingface.co/datasets/VladPyatov/CADFS)
- 代码：[VladPyatov/CADFS on GitHub](https://github.com/VladPyatov/CADFS)
- 数据许可：CC BY 4.0；代码许可：MIT

数据只用一句话概括其来源：作者从真实 CAD/ABC 与 Onshape 相关数据中恢复干净 FeatureScript，并配套生成文本、图像和几何表示；本文不讨论该构建流水线。

## 2. 本次实际下载了什么

本次下载位置为 `repos/CADFS/cadfs/`。按需求只下载处理后的核心数据和辅助索引，没有下载约 72.8 GB 的原始 STL、多视图图像与 ABC 源数据。

### 2.1 核心压缩包

| 压缩包 | 本地文件数 | 压缩包大小 | ZIP 内文件解压总量 | 主要扩展名 | 用途 |
|---|---:|---:|---:|---|---|
| `dataset/featurescript_fp.zip` | 450,307 | 786,054,431 B（749.64 MiB） | 3,899,977,960 B（3.63 GiB） | `.txt` | 全精度 FeatureScript，尽量保留原参数 |
| `dataset/featurescript_rp.zip` | 450,307 | 688,118,023 B（656.24 MiB） | 3,521,836,282 B（3.28 GiB） | `.txt` | 数值保留两位小数的 FeatureScript，训练/评测主要版本 |
| `dataset/text_annotations.zip` | 449,883 | 411,749,548 B（392.68 MiB） | 893,563,255 B（852.17 MiB） | `.txt` | 英文逐步设计历史描述 |
| `dataset/step.zip` | 450,307 | 7,448,943,349 B（6.94 GiB） | 39,148,251,055 B（36.46 GiB） | `.step` | 可交换的 B-rep 几何 |
| **合计（四个 ZIP）** | — | **9,334,865,351 B（9.33 GB / 8.69 GiB）** | — | — | 本次核心下载 |

表中的“本地文件数”和“ZIP 内文件解压总量”来自 ZIP central directory 的实际遍历，不是数据卡上的四舍五入口径。前三个 ZIP 各有 101 个目录条目，STEP ZIP 有 102 个；表中没有把目录算作数据文件。

保留了全部原始 ZIP；`featurescript_fp/`、`featurescript_rp/` 和 `text_annotations/` 已解压以便抽查与使用。完整 STEP 保留为 ZIP，只单独提取本文 15 个案例，避免再占用整个 STEP 解压后的空间。

### 2.2 辅助索引与测试集

| 文件/集合 | 实测数量 | 含义 |
|---|---:|---|
| `misc/unique_models.json` | 382,609 | 按相似 FeatureScript 或 B-rep 去重后的模型 ID；占 450,307 的 84.97% |
| `misc/stage1_indices.json` | 196,084 | Stage-1（以 Sketch/Extrude 为主）索引；占 43.54% |
| `misc/DeepCAD_train_val_test_split.json` | 161,240 / 8,946 / 8,052 | DeepCAD 的 train / validation / test 划分 |
| `test_data/CADFS_test.json` | 9,347 | CADFS 专用测试 ID；排除 DeepCAD train/validation/test，约占正式程序包 2.08% |
| `test_data/CADFS_test/CADFS_text_test.jsonl` | 9,347 行 | 已整理好的文本条件测试记录 |

本地 CADFS 测试目录中，FeatureScript、文本标注和 `.step` 文件都严格为 9,347 个，与测试 ID 一一对应。训练或评测仍应以 split/JSONL 的 ID 为准，不要仅通过未过滤扩展名的目录遍历推断样本集。

## 3. 为什么会看到 451k、450,307、382,609 等不同数字

这些数字表示不同统计口径，并不矛盾：

| 数量 | 口径 | 应如何使用 |
|---:|---|---|
| 451k | 论文对设计规模的千位近似表述 | 写背景、与其他数据集比较 |
| 450k | 摘要/项目页的更粗略表述 | 非精确概述 |
| **450,307** | 两个正式 FeatureScript ZIP 中的实际 `.txt` 数 | 本地文件处理、覆盖率计算的基准 |
| **449,883** | 实际文本标注 `.txt` 数 | 文本任务的原始可用上限；比程序少 424 个，覆盖率 99.906% |
| **382,609** | `unique_models.json` 的去重 ID 数 | 需要控制近重复/数据泄漏时使用 |
| **196,084** | `stage1_indices.json` 的 ID 数 | 两阶段训练中 Stage-1 子集口径 |
| 约 170k / 405k | 论文训练阶段的近似规模 | 还会受测试排除、去重选择和 8192-token 过滤影响，不能直接等同于索引长度 |
| **9,347** | `CADFS_test.json` 和测试 JSONL 的实际记录数 | CADFS 15 操作测试集 |

因此，最稳妥的写法是：“论文报告约 451k 个设计；当前发布压缩包含 450,307 个程序，其中 449,883 个有文本标注，去重索引保留 382,609 个。”

## 4. 数据在磁盘上是什么样的

### 4.1 目录与命名规则

核心模态遵循同一种路径规则：

```text
cadfs/
├── dataset/
│   ├── featurescript_fp.zip
│   ├── featurescript_fp/{chunk_id:04}/{model_id:08}.txt
│   ├── featurescript_rp.zip
│   ├── featurescript_rp/{chunk_id:04}/{model_id:08}.txt
│   ├── text_annotations.zip
│   ├── text_annotations/{chunk_id:04}/{model_id:08}.txt
│   ├── step.zip
│   └── step/{chunk_id:04}/{model_id:08}.step
├── misc/
│   ├── unique_models.json
│   ├── stage1_indices.json
│   └── DeepCAD_train_val_test_split.json
└── test_data/
    ├── CADFS_test.json
    └── CADFS_test/
        ├── CADFS_text_test.jsonl
        ├── featurescript_rp/{chunk}/{id}.txt
        ├── text_annotations/{chunk}/{id}.txt
        └── step_abc/{chunk}/{id}.step
```

`chunk_id` 是八位 `model_id` 的前四位。例如模型 `00397823` 位于 chunk `0039`：

```text
featurescript_rp/0039/00397823.txt
text_annotations/0039/00397823.txt
step/0039/00397823.step
```

### 4.2 FeatureScript：可执行的设计程序

RP 文件是 UTF-8/ASCII 风格纯文本，通常具有固定框架：声明 FeatureScript 版本、导入 Onshape 标准库、定义单位和查询常量，然后按设计顺序调用 `newSketch`、`extrude`、`fillet` 等函数。

```featurescript
FeatureScript 1511;
import(path : "onshape/std/geometry.fs", version : "1511.0");
import(path : "onshape/std/common.fs", version : "1511.0");
const mm = millimeter;
...
var sketch = newSketch(context, id + "F0", { "sketchPlane" : qUnion([Q0])});
skCircle(sketch, "E0", {"center" : v(0, 0) * mm, "radius" : 60.37 * mm});
skSolve(sketch);
extrude(context, id + "F1", {"entities" : qUnion([Q1]), "depth" : 25.4 * mm});
```

对 450,307 个 RP 文件全量扫描得到：

| 指标 | 最小 | 中位数 | 平均 | P90 | P99 | 最大 |
|---|---:|---:|---:|---:|---:|---:|
| 文件字节数 | 1,307 | 4,171 | 7,820.97 | 15,054 | 56,407 | 3,558,299 |
| 文本行数 | 32 | 65 | 96.00 | 172 | 509 | 15,090 |

这说明“典型样本”不长，但尾部非常重：最大程序约 3.56 MB、15,090 行。若直接把所有原始程序送进 LLM，极端长样本会远超常见上下文窗口，因此官方 JSONL 会过滤“输入 + 输出超过 8192 tokens”的记录。

版本也不是完全统一：`FeatureScript 1511;` 占 395,884 个（87.91%），其余样本分布在多个版本。执行程序时不能假定所有文件都使用同一标准库版本。

### 4.3 FP 与 RP 的差别

- **FP（full precision）**：数值参数保留较高精度，归档体积和解压体积都更大，适合几何复现和精度敏感分析。
- **RP（rounded precision）**：参数保留两位小数，文本更短，是官方训练/评测工作流的主要输入。

RP 不能被简单理解为“更干净且一定可编译”。舍入可能让原本精确相交、相切、共面或满足约束的几何变成退化/不相交，从而导致编译或几何生成失败；反过来，数据卡也提醒某些程序只在 RP 版本能成功。实际应用中应分别记录“代码可编译”“几何可生成”和“与参考 STEP 几何一致”三种状态。

### 4.4 文本标注：英文步骤式设计历史

文本标注不是类别标签，而是类似 CAD 教程的逐步自然语言：

```text
Step 1 - Sketch
Create a new sketch on the Front plane. Draw a circle ...

Step 2 - Extrude NEW
Extrude the circular sketch region ... creating a solid cylindrical body.
```

对 449,883 个标注文件全量扫描得到：

| 指标 | 最小 | 中位数 | 平均 | P90 | P99 | 最大 |
|---|---:|---:|---:|---:|---:|---:|
| 文件字节数 | 45 | 1,560 | 1,986.21 | 3,871 | 7,660 | 38,265 |
| 英文词/数字 token（正则粗计） | 8 | 321 | 406.72 | 791 | 1,566 | 7,110 |
| `Step N -` 步骤数 | 0 | 4 | 6.45 | 13 | 31 | 225 |

步骤复杂度分布：

| 步骤数 | 文件数 | 占比 |
|---:|---:|---:|
| 0（不符合标准步骤标题） | 92 | 0.020% |
| 1–2 | 133,689 | 29.716% |
| 3–5 | 126,340 | 28.083% |
| 6–10 | 118,607 | 26.364% |
| 11–20 | 55,696 | 12.380% |
| 21+ | 15,459 | 3.436% |

这里的 92 个“0 步”不是空文件，而是没有采用标准 `Step N -` 标题。随机抽到的 `00859595.txt` 是一段对原标注的否定性审查意见，要求重写 55 个 FeatureScript 操作，而不是最终的逐步标注。这是一个可直接观察到的数据质量边界：正式使用前至少应做格式校验，而不能只判断文件是否存在。

### 4.5 STEP：最终 B-rep 几何

STEP 文件保存程序执行后的边界表示（B-rep），可在 FreeCAD、Open CASCADE、SolidWorks 等工具中读入。它适合：

- 计算几何指标或渲染参考图；
- 验证 FeatureScript 执行结果是否与参考形状一致；
- 将“设计程序正确性”和“最终几何相似性”分开评测。

STEP 只保存交换后的几何拓扑/曲面，不等价于 FeatureScript 的完整设计历史。因此，训练可编辑 CAD 生成器时不应把 STEP 当作 FeatureScript 的替代品，而应把它当作同一模型的几何监督或验证目标。

### 4.6 JSONL：直接用于 SFT/评测的聊天记录

文本条件记录的实际结构是：

```json
{
  "messages": [
    {"role": "system", "content": "You are CAD code generation model."},
    {"role": "user", "content": "Step 1 - Sketch\n..."},
    {"role": "assistant", "content": "FeatureScript 1511;\n..."}
  ],
  "cad_file_id": "00858269"
}
```

图像条件记录比它多一个 `images` 字段，用户消息以 `<image>` 开头并包含 mesh bounds、中心和缩放等几何上下文：

```json
{
  "messages": [...],
  "images": ["path/to/0085/00858269.png"],
  "cad_file_id": "00858269"
}
```

字段含义：

- `messages`：标准对话式监督数据；user 是条件，assistant 是目标 FeatureScript。
- `cad_file_id`：跨 FeatureScript、标注、图像、STL/STEP 和 split 对齐的主键。
- `images`：图像条件样本的文件路径列表；下载后通常还要用官方脚本改写为本地路径。

官方提供的训练 JSONL 位于 `train_data/`，已过滤近重复并限制输入加输出不超过 8192 tokens；本次按“核心处理后数据”范围没有额外下载训练 JSONL 和图像包。

## 5. 同一个模型如何跨三种表示对应

以真实测试模型 `00025852` 为例：

```text
FeatureScript RP: dataset/featurescript_rp/0002/00025852.txt
文本标注:         dataset/text_annotations/0002/00025852.txt
测试 STEP:        test_data/CADFS_test/step_abc/0002/00025852.step
JSONL 主键:       "cad_file_id": "00025852"
```

它的 FeatureScript 依次包含两个同心圆、环形套筒拉伸、中心圆柱加料以及 counterbore 孔等调用；自然语言标注以 8 个步骤描述同一设计历史；STEP 则表示最终 B-rep。三者各自回答不同问题：

| 表示 | 保留的主要信息 | 适合任务 |
|---|---|---|
| FeatureScript | 参数、顺序、实体查询、特征依赖 | 程序生成、设计历史恢复、可编辑性研究 |
| 文本标注 | 人类可读的意图和操作步骤 | 文本到 CAD、语义对齐、指令微调 |
| STEP | 最终曲面、边、拓扑与实体 | 几何验证、渲染、距离/拓扑指标 |

一个合理的闭环评测是：模型根据文本生成 FeatureScript → 编译/执行 → 导出 STEP → 与参考 STEP 比较几何，同时单独比较操作序列是否一致。

## 6. 15 种操作的全量分布

下表对 **450,307 个 `featurescript_rp` 文件**做正则匹配，统计“至少出现一次该操作的模型数”。同一模型会进入多行，因此百分比不会相加到 100%。Sketch 用 `newSketch(`；其他操作用对应的 FeatureScript 调用（如 `fillet(context`、`booleanBodies(context`）识别。

| 操作 | 至少出现一次的模型数 | 占全部程序 | CADFS 测试集模型数 | 测试集占比 |
|---|---:|---:|---:|---:|
| Sketch | 450,307 | 100.000% | 9,347 | 100.00% |
| Extrude | 423,869 | 94.129% | 8,238 | 88.14% |
| Revolve | 51,148 | 11.358% | 1,798 | 19.24% |
| Sweep | 9,616 | 2.135% | 326 | 3.49% |
| Loft | 8,565 | 1.902% | 309 | 3.31% |
| Construction Plane | 35,480 | 7.879% | 1,188 | 12.71% |
| Fillet | 104,245 | 23.150% | 3,639 | 38.93% |
| Chamfer | 42,320 | 9.398% | 1,432 | 15.32% |
| Shell | 19,356 | 4.298% | 696 | 7.45% |
| Hole | 24,709 | 5.487% | 894 | 9.56% |
| Boolean | 5,773 | 1.282% | 187 | 2.00% |
| Delete Body | 3,591 | 0.797% | 123 | 1.32% |
| Circular Pattern | 7,191 | 1.597% | 232 | 2.48% |
| Mirror | 12,070 | 2.680% | 411 | 4.40% |
| Transform | 9,692 | 2.152% | 327 | 3.50% |

测试集明显提高了多种高级操作的比例。例如 Fillet 从全量的 23.15% 提高到 38.93%，Revolve 从 11.36% 提高到 19.24%，Loft 从 1.90% 提高到 3.31%。这使 9,347 个测试样本更适合检验复杂操作能力，但不能把测试分布当成自然训练分布。

### 6.1 共现关系

最常见的共现组合为：

| 操作对 | 共现模型数 | 占全部程序 |
|---|---:|---:|
| Sketch + Extrude | 423,869 | 94.129% |
| Extrude + Fillet | 100,735 | 22.370% |
| Extrude + Chamfer | 40,335 | 8.957% |
| Extrude + Construction Plane | 30,958 | 6.875% |
| Extrude + Revolve | 30,769 | 6.833% |
| Extrude + Hole | 23,754 | 5.275% |
| Fillet + Chamfer | 19,001 | 4.220% |
| Extrude + Shell | 17,652 | 3.920% |
| Revolve + Fillet | 15,406 | 3.421% |
| Construction Plane + Fillet | 11,712 | 2.601% |
| Loft + Construction Plane | 7,716 | 1.713% |

这反映了真实设计历史的层级结构：Sketch/Extrude 是主体构造，Fillet、Chamfer、Hole、Shell 更像后处理特征，Construction Plane 常用于为 Loft 等操作提供非默认参考面。对建模而言，稀有操作的难点不仅是样本少，还包括“先生成什么实体，随后如何准确查询并引用它”。

## 7. 15 种操作的真实 case

下面每个案例都在本地 CADFS 测试包中核对了 FeatureScript、文本标注和 STEP 三种文件。路径写成测试包的实际目录；完整核心 STEP ZIP 解压后可用同一个 `{chunk}/{id}` 规则定位。

### 7.1 Sketch — `00397823`

- **测试集频率**：9,347 / 9,347（100.00%）
- **操作含义**：在指定平面建立二维草图，创建后续实体特征所需的轮廓或路径。
- **自然语言标注摘录**：在 Front plane 新建草图，以原点为圆心绘制半径 60.37 mm 的圆；随后将圆形区域拉伸 25.4 mm。
- **关键 FeatureScript**：

```featurescript
var sketch = newSketch(context, id + "F0", { "sketchPlane" : qUnion([Q0])});
skCircle(sketch, "E0", {"center": v(0, 0) * mm, "radius": 60.37 * mm});
skSolve(sketch);
```

- **文件**：`featurescript_rp/0039/00397823.txt`；`text_annotations/0039/00397823.txt`；`step_abc/0039/00397823.step`

### 7.2 Extrude — `00250785`

- **测试集频率**：8,238 / 9,347（88.14%）
- **操作含义**：沿草图平面法向将区域或边拉伸为实体/曲面。
- **自然语言标注摘录**：在 Front plane 绘制半径 38.1 mm 的圆，选择圆边并以 SURFACE 方式拉伸 38.1 mm，得到圆柱曲面。
- **关键 FeatureScript**：

```featurescript
extrude(context, id + "F1", {
    "bodyType" : ToolBodyType.SURFACE,
    "surfaceEntities" : qUnion([Q0]),
    "depth" : 38.1 * mm
});
```

- **文件**：`featurescript_rp/0025/00250785.txt`；`text_annotations/0025/00250785.txt`；`step_abc/0025/00250785.step`

### 7.3 Revolve — `00210709`

- **测试集频率**：1,798 / 9,347（19.24%）
- **操作含义**：让草图区域绕轴旋转，形成回转实体或曲面。
- **自然语言标注摘录**：绘制圆心 `(-51.22, 0)`、半径 13.86 mm 的圆和一条竖直轴线，将圆的内部区域绕该轴完整旋转 360°，得到环面实体。
- **关键 FeatureScript**：

```featurescript
revolve(context, id + "F1", {
    "entities" : qUnion([Q0]), "axis" : qUnion([Q1]),
    "revolveType" : RevolveType.FULL
});
```

- **文件**：`featurescript_rp/0021/00210709.txt`；`text_annotations/0021/00210709.txt`；`step_abc/0021/00210709.step`

### 7.4 Sweep — `00816123`

- **测试集频率**：326 / 9,347（3.49%）
- **操作含义**：让截面沿路径运动，生成管、杆或更一般的扫掠体。
- **自然语言标注摘录**：Front plane 上绘制从 `(0,0)` 到 `(492.4,-86.82)` 的直线路径，Right plane 上绘制半径 0.5 mm 的圆，沿直线扫掠成细管。
- **关键 FeatureScript**：

```featurescript
sweep(context, id + "F2", {
    "profiles" : qUnion([Q0]),
    "path" : qUnion([Q1])
});
```

- **文件**：`featurescript_rp/0081/00816123.txt`；`text_annotations/0081/00816123.txt`；`step_abc/0081/00816123.step`

### 7.5 Loft — `00546176`

- **测试集频率**：309 / 9,347（3.31%）
- **操作含义**：在两个或更多截面之间插值，构成平滑过渡体。
- **自然语言标注摘录**：在 Top plane 绘制三角形，在 Front plane 绘制终止于 `(0,4.04)` 的线并选取端点；从三角区域 Loft 到该点，形成尖锥状实体。
- **关键 FeatureScript**：

```featurescript
loft(context, id + "F2", {"sheetProfilesArray" : [
    { "sheetProfileEntities" : qUnion([Q0]) },
    { "sheetProfileEntities" : qUnion([Q1]) }
]});
```

- **文件**：`featurescript_rp/0054/00546176.txt`；`text_annotations/0054/00546176.txt`；`step_abc/0054/00546176.step`

### 7.6 Construction Plane — `00374842`

- **测试集频率**：1,188 / 9,347（12.71%）
- **操作含义**：创建偏置或按几何定义的参考平面，为后续草图/特征提供定位基准。
- **自然语言标注摘录**：先生成半径 17.46 mm、长度 152.4 mm 的圆柱；再相对端面沿反法向偏置 27.94 mm，创建 152.4 mm × 152.4 mm 的构造平面。
- **关键 FeatureScript**：

```featurescript
cPlane(context, id + "F2", {
    "entities" : qUnion([Q0]), "cplaneType" : CPlaneType.OFFSET,
    "offset" : 27.94 * mm, "oppositeDirection" : true,
    "width" : 152.4 * mm, "height" : 152.4 * mm
});
```

- **文件**：`featurescript_rp/0037/00374842.txt`；`text_annotations/0037/00374842.txt`；`step_abc/0037/00374842.step`

### 7.7 Fillet — `00307793`

- **测试集频率**：3,639 / 9,347（38.93%）
- **操作含义**：用圆角过渡替换尖锐边。
- **自然语言标注摘录**：创建半径 7.5 mm、高 20 mm 的圆柱；对上下两条圆边施加半径 7 mm 的圆角，启用切向传播并禁用 edge overflow。
- **关键 FeatureScript**：

```featurescript
fillet(context, id + "F2", {
    "entities" : qUnion([Q0, Q1]), "radius" : 7 * mm,
    "tangentPropagation" : true, "allowEdgeOverflow" : false
});
```

- **文件**：`featurescript_rp/0030/00307793.txt`；`text_annotations/0030/00307793.txt`；`step_abc/0030/00307793.step`

### 7.8 Chamfer — `00013090`

- **测试集频率**：1,432 / 9,347（15.32%）
- **操作含义**：用斜面切去锐边。
- **自然语言标注摘录**：创建半径 1.22 mm、高 7.75 mm 的圆柱；对两端圆边施加等距 0.25 mm 倒角并启用切向传播。
- **关键 FeatureScript**：

```featurescript
chamfer(context, id + "F2", {
    "entities" : qUnion([Q0, Q1]),
    "width" : 0.25 * mm,
    "tangentPropagation" : true
});
```

- **文件**：`featurescript_rp/0001/00013090.txt`；`text_annotations/0001/00013090.txt`；`step_abc/0001/00013090.step`

### 7.9 Shell — `00773260`

- **测试集频率**：696 / 9,347（7.45%）
- **操作含义**：移除选定面并把实体变成具有给定壁厚的薄壁体。
- **自然语言标注摘录**：将半径 11.46 mm 的圆拉伸 60.96 mm，选择起止两个端面并以 2.54 mm 厚度抽壳，得到两端开口的薄壁管。
- **关键 FeatureScript**：

```featurescript
shell(context, id + "F2", {
    "entities" : qUnion([Q0, Q1]),
    "thickness" : 2.54 * mm
});
```

- **文件**：`featurescript_rp/0077/00773260.txt`；`text_annotations/0077/00773260.txt`；`step_abc/0077/00773260.step`

### 7.10 Hole — `00161681`

- **测试集频率**：894 / 9,347（9.56%）
- **操作含义**：按孔型、终止方式、直径、攻丝和定位参数创建工程孔特征。
- **自然语言标注摘录**：创建半径 31.75 mm、高 6.35 mm 的圆柱；在中心放置直径 25.4 mm 的反向贯穿孔，并设置 tapped through。
- **关键 FeatureScript**：

```featurescript
hole(context, id + "F2", {
    "style" : HoleStyle.SIMPLE, "endStyle" : HoleEndStyle.THROUGH,
    "oppositeDirection" : true, "holeDiameter" : 25.4 * mm,
    "locations" : qUnion([Q0]), "scope" : qUnion([Q1]),
    "isTappedThrough" : true
});
```

- **文件**：`featurescript_rp/0016/00161681.txt`；`text_annotations/0016/00161681.txt`；`step_abc/0016/00161681.step`

### 7.11 Boolean — `00075961`

- **测试集频率**：187 / 9,347（2.00%）
- **操作含义**：对实体执行并集、差集或交集。
- **自然语言标注摘录**：生成一个水平圆柱和一个偏置 19.05 mm 的竖直小圆柱，从前者中减去后者，得到偏置的垂直贯穿孔。
- **关键 FeatureScript**：

```featurescript
booleanBodies(context, id + "F4", {
    "operationType" : BooleanOperationType.SUBTRACTION,
    "tools" : qUnion([Q0]), "targets" : qUnion([Q1])
});
```

- **文件**：`featurescript_rp/0007/00075961.txt`；`text_annotations/0007/00075961.txt`；`step_abc/0007/00075961.step`

### 7.12 Delete Body — `00737786`

- **测试集频率**：123 / 9,347（1.32%）
- **操作含义**：从 Part Studio 中删除选定实体，常用于清理构造体或中间结果。
- **自然语言标注摘录**：三个圆形区域拉伸成三个圆柱后，删除由第三个圆（中心 `(-81.24,168.53)`）生成的实体，仅保留前两个圆柱。
- **关键 FeatureScript**：

```featurescript
deleteBodies(context, id + "F2", {
    "entities" : qUnion([Q0])
});
```

- **文件**：`featurescript_rp/0073/00737786.txt`；`text_annotations/0073/00737786.txt`；`step_abc/0073/00737786.step`

### 7.13 Circular Pattern — `00143647`

- **测试集频率**：232 / 9,347（2.48%）
- **操作含义**：绕轴等角度复制特征、面或实体。
- **自然语言标注摘录**：将有中心孔的环形实体绕内孔圆边指定的轴做 PART 阵列，每个实例相差 12°，共 30 个实例，并以 ADD 方式加入模型。
- **关键 FeatureScript**：

```featurescript
circularPattern(context, id + "F2", {
    "operationType" : NewBodyOperationType.ADD,
    "entities" : qUnion([Q0]), "axis" : qUnion([Q1]),
    "angle" : 12 * degree, "instanceCount" : 30
});
```

- **文件**：`featurescript_rp/0014/00143647.txt`；`text_annotations/0014/00143647.txt`；`step_abc/0014/00143647.step`

### 7.14 Mirror — `00094392`

- **测试集频率**：411 / 9,347（4.40%）
- **操作含义**：跨指定平面镜像复制特征、面或实体。
- **自然语言标注摘录**：生成半径 36.09 mm、长度 204.44 mm 的圆柱，以远端盖面为镜像平面并以 ADD 方式加入镜像副本。
- **关键 FeatureScript**：

```featurescript
mirror(context, id + "F2", {
    "operationType" : NewBodyOperationType.ADD,
    "entities" : qUnion([Q0]),
    "mirrorPlane" : qUnion([Q1])
});
```

- **文件**：`featurescript_rp/0009/00094392.txt`；`text_annotations/0009/00094392.txt`；`step_abc/0009/00094392.step`

### 7.15 Transform — `00100501`

- **测试集频率**：327 / 9,347（3.50%）
- **操作含义**：平移、旋转或复制所选实体。
- **自然语言标注摘录**：生成半径 39.56 mm、高 25.4 mm 的圆柱；保留原件并复制，将副本沿 X 平移 1270 mm、沿 Y 平移 50.8 mm。
- **关键 FeatureScript**：

```featurescript
transform(context, id + "F2", {
    "entities" : qUnion([Q0]),
    "transformType" : TransformType.TRANSLATION_3D,
    "dx" : 1270 * mm, "dy" : 50.8 * mm, "dz" : 0 * mm,
    "makeCopy" : true
});
```

- **文件**：`featurescript_rp/0010/00100501.txt`；`text_annotations/0010/00100501.txt`；`step_abc/0010/00100501.step`

## 8. 数据特点与工程分析

### 8.1 优势

1. **表达能力比 Sketch–Extrude 序列更丰富。** 15 种操作覆盖回转、放样、扫掠、参考面和多种工程后处理，能表示更接近真实零件的设计历史。
2. **FeatureScript 是代码而不是匿名 token。** 参数有单位，操作有明确函数名，实体通过 query 选择；输出可读、可编辑，并能在 Onshape 环境中执行。
3. **同一 ID 提供程序—语言—几何对齐。** 可以同时训练语义遵循、程序生成和几何一致性。
4. **规模足以支持 LLM/VLM 微调。** 即使采用 382,609 个去重索引，仍显著大于很多只包含简单命令的 CAD 程序数据集。

### 8.2 长尾是首要建模难题

如果随机抽取训练数据，模型几乎总能看到 Sketch/Extrude，却很少看到 Delete Body、Boolean、Circular Pattern、Loft 和 Sweep。一个只按总体 loss 优化的模型可能通过反复 Sketch + Extrude 获得不错的平均表现，却不会稳定生成稀有高级特征。

可采用的策略包括：

- 按“模型是否包含某操作”做多标签分层抽样，而不是把每个模型硬分到单一类别；
- 对稀有操作过采样或设置 operation-aware loss；
- 评测时分别报告 15 种操作的出现召回、参数正确率和实体引用正确率；
- 专门构造包含复杂共现链的子集，例如 Construction Plane → Sketch → Loft、Extrude → Hole → Fillet。

### 8.3 编译成功不等于设计正确

FeatureScript 的评测至少需要四层：

1. **语法/编译有效性**：代码能否通过编译；
2. **几何生成有效性**：执行后能否得到非空 B-rep；
3. **几何一致性**：生成 STEP 与参考 STEP 的形状、尺度、拓扑是否接近；
4. **设计历史一致性**：操作类型、顺序、参数和实体引用是否符合条件描述。

例如，用许多拉伸近似一个圆周阵列，最终几何可能接近，但设计历史并不等价；而漏掉 Fillet 可能保持主体几何，却损失关键制造特征。只看 Chamfer Distance 会掩盖这些问题。

### 8.4 自动文本标注存在可预期误差

标注是模型生成的英文设计步骤，优点是规模大、与 FeatureScript 的表达结构接近；风险是：

- 可能漏掉某个操作或把操作顺序说错；
- 可能将 query 指向的边/面描述成错误对象；
- 数值、方向、NEW/ADD/REMOVE、blind/through 等属性可能不一致；
- 少量文件可能是审查意见而非规范步骤，本文严格格式扫描发现 92 个此类候选；
- FeatureScript 有 450,307 个，但标注少 424 个，天然不是完整一一对应。

因此应把自然语言看作“弱监督/自动标注”，而不是人工验证的绝对真值。训练前可做操作序列对齐：从标注标题抽取操作名，与 FeatureScript 函数序列比较，对缺失、额外或顺序异常样本降权或剔除。

### 8.5 去重与数据泄漏

原始程序中有约 15.03% 未进入 `unique_models.json`。如果随机按文件划分，几何或代码近重复可能跨越训练集和测试集，导致评测过高。建议：

- 优先用官方 split 和 `unique_models.json`；
- 用 `cad_file_id` 联结所有模态后再切分，避免同一模型不同表示落入不同集合；
- 如建立新 benchmark，应先按几何/程序簇分组，再按簇划分。

### 8.6 适合与不适合的任务

**适合：**

- 文本到 FeatureScript 生成；
- 单图/多视图到可编辑 CAD 程序重建；
- CAD 设计历史补全、修复和解释；
- 操作序列识别、参数预测、实体引用学习；
- 程序执行后与 STEP 的几何一致性评测；
- 稀有高级特征的生成能力研究。

**需要谨慎：**

- 把 15 种操作当作互斥类别做普通分类；
- 未编译验证就把所有 RP 文件当作可执行真值；
- 把自动英文标注当成人工严格校对的说明书；
- 忽略 8192-token 过滤后，直接把论文训练规模等同于压缩包文件数；
- 只用最终 STEP 评估是否恢复了正确设计意图。

## 9. 推荐的最小使用流程

如果目标是训练或评估文本到 CAD 模型，建议：

1. 以 `unique_models.json` 和官方 split 选 ID；
2. 联结 `text_annotations/{chunk}/{id}.txt` 与 `featurescript_rp/{chunk}/{id}.txt`；
3. 校验标注格式并比较标注操作序列与 FeatureScript 调用；
4. 过滤输入加输出超过模型上下文的样本，或使用长上下文/分段策略；
5. 训练时对稀有操作做多标签分层采样；
6. 推理后编译 FeatureScript，导出 STEP；
7. 同时报告编译率、几何生成率、几何指标和 15 操作分项指标。

若只是想先观察数据，可从本文 15 个 case 开始；它们都在 9,347 个测试样本内，文件较小且三种表示齐全。

## 10. 本地校验记录与统计口径

本次校验执行了以下只读检查：

- 对四个 ZIP 读取 central directory，核对文件数、扩展名、压缩与解压字节数；三个较小 ZIP 还逐成员通过 CRC 测试；
- `step.zip` 的实际大小与 Hugging Face 元数据同为 7,448,943,349 B，SHA-256 也与远端内容哈希一致：`42c79eae931641fc888dd9355d353c7fea8f725a4c23316fe644b06aa24984f1`；
- 对 450,307 个 RP FeatureScript 逐文件扫描长度、行数、版本声明、15 种操作出现情况和两两共现；
- 对 449,883 个文本标注逐文件扫描大小、粗略词数和 `Step N -` 步骤数；
- 读取三个 JSON 索引并统计长度；
- 核对 9,347 行测试 JSONL 的字段；
- 核对 15 个固定 case 的 FeatureScript、标注和 STEP 均存在，并检查关键操作调用、参数和单位。

操作计数是“**包含该调用的模型数**”，不是调用总次数；步骤数由行首 `Step + 数字 + -` 正则统计；词数是英文/数字正则粗计，不能等同于任一模型 tokenizer 的 token 数。所有百分比均以本地 ZIP 的实际文件数为分母。

## 参考资料

1. Pyatov, V. et al. [CADFS: A Big CAD Program Dataset and Framework for Computer-Aided Design with Large Language Models](https://arxiv.org/abs/2605.01925), CVPR 2026.
2. [CADFS 官方项目页](https://voyleg.github.io/cadfs/).
3. [CADFS Hugging Face 数据卡与文件](https://huggingface.co/datasets/VladPyatov/CADFS).
4. [CADFS 官方代码仓库](https://github.com/VladPyatov/CADFS).

```bibtex
@inproceedings{pyatov2026cadfs,
  title  = {CADFS: A Big CAD Program Dataset and Framework for Computer-Aided Design with Large Language Models},
  author = {Pyatov, Vladislav and Bobrovskikh, Gleb and Galochkin, Saveliy and Boldyrev, Nikita and Voynov, Oleg and Filippov, Alexander and Ferrer, Gonzalo and Wonka, Peter and Burnaev, Evgeny},
  booktitle = {Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition},
  year   = {2026}
}
```
