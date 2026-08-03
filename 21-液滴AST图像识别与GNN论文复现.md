# 液滴 AST 图像识别与 GNN 论文复现：软件开发岗位面试稿

> 适用岗位：新凯来软件开发工程师
> 讲述定位：复杂图像软件 pipeline、模块集成、故障定位、可复现实验
> 结论边界：开发期研究系统，不是临床 AST 产品

## 面试时的项目定位

不要把项目讲成：

> 我用了 Cellpose、GNN 和很多模型做耐药分类。

应该讲成：

> 我把一个原始显微图像到耐药证据的复杂任务，拆成分割、质量控制、前景提取、结构建模和整图聚合等可独立验证的阶段；又通过论文结构复现和参数受控实验，判断复杂模型是否真的带来增益，并在整图结果不理想时定位失败发生在哪一层。

对于软件开发岗位，项目价值主要是：

1. 将研究脚本重构为模块化、可配置的 Python pipeline。
2. 为每个阶段定义输入、输出、中间产物和失败检查。
3. 通过日志、manifest、hash、checkpoint replay 和可视化定位问题。
4. 用对照实验做工程决策，而不是只追求复杂模型。
5. 诚实区分开发期指标、独立测试和最终业务准确率。

## 先给零背景面试官解释问题

### 一句话问题定义

> 给定抗生素作用 2 到 4 小时后的微液滴显微图像，系统需要自动找出图中的上百个液滴，提取每个液滴内部的细菌生长形态，判断它更像耐药 RES 还是敏感 SUS，再把多滴证据汇总成整张图的结果。

第一次出现时解释：

- RES：resistant，耐药。
- SUS：susceptible，敏感。
- 一个 image/timepoint：某个样本在一个时间点拍摄的一张整图。
- 一个 droplet：整图中的一个微液滴，是单滴模型的预测单位。

### 为什么困难

1. 液滴边界有暗环、粘连、反光，分割结果需要修复和质量控制。
2. 液滴内部有网格参考点、气泡、边缘反射等伪影，亮点不一定是细菌信号。
3. 一张图虽然有上百滴，但这些液滴来自同一实验样本，并不是上百个独立生物样本。
4. 不同抗生素、时间点和实验批次的图像分布不同，容易发生 domain shift。
5. 单滴预测正确不等于整图聚合正确，两层都需要独立验证。

## 90 秒项目介绍

> 我做的是一套微液滴显微图像的耐药证据分析 pipeline。输入是一张包含上百个液滴的原始 TIF 图像，目标是先判断每个液滴中是否存在耐药相关的生长形态，再聚合成整图结果。
>
> 工程上我把流程拆成五层。第一层用 Fiji 做对比度预处理，再用 Cellpose-SAM 分割液滴，并对暗边和实例边界做修复；第二层按面积、圆度、长宽比等规则做质量控制并裁出单滴；第三层融合液滴局部和整图级前景检测，去除参考网格、小碎片和部分反光；第四层把液滴里的前景连通区域变成图节点，用 7 个形态特征和空间邻接预测单滴 RES 概率；第五层再比较固定数量、比例阈值和多 seed 投票等整图聚合方法。
>
> 为了判断 GNN 的复杂结构是否真的有价值，我参考 Nature Communications 2025 的 tissue GNN 工作，在同一数据 split、参数预算和随机种子下运行了 4 种药物、8 个结构变体、3 个 seed，共 96 次训练。结果显示 layerwise pooling 没有稳定优于 final-layer mean，空间边对不同药物的作用方向也不一致。因此我没有默认采用最复杂结构，而是保留更稳定的 final-layer mean 和单药模型。
>
> 后续整图未见样本检查结果并不理想，但通过固定模型和聚合阈值对照，我排除了“只调最后阈值就能解决”的假设，把主要问题定位到单滴跨批次泛化、图像污染和权威真值不足。这个项目最能体现的是复杂 pipeline 的模块化、复现审计和故障隔离能力，而不是一个已经可临床使用的分类器。

## 10 分钟讲述顺序

### 1. 问题、输入和输出：1 分钟

```text
输入：抗生素作用后 2h / 4h 的微液滴显微整图
中间：上百个液滴及其内部生长形态
输出：单滴 RES 证据分数；开发期整图 RES/SUS 判定
```

强调当前标签是人工或文件记录整理出的证据标签，不是每个液滴独立培养得到的临床真值。

### 2. 当前 pipeline：2～3 分钟

```text
原始 TIF
→ Fiji 16-bit 对比度处理
→ Cellpose-SAM 液滴实例分割
→ 暗边修复、面积与形状 QC
→ 单液滴裁剪
→ 局部 Triangle 前景 + 整图 Triangle 参考融合
→ 网格/参考点、小组件和边缘反射清理
→ 连通组件特征与 component graph
→ 单滴 RES 概率
→ 多滴聚合与整图结果
```

#### Stage 1：液滴分割与修复

- Fiji 对原始图做统一对比度处理。
- Cellpose-SAM 输出实例 mask。
- 根据局部暗边对实例做向内或向外修复。
- 用面积、圆度、长宽比、bbox 填充率做宽松 QC。
- 每张图输出原始/修复 mask、overlay、repair records 和 QC CSV。

软件工程意义：分割不是一个黑盒函数，原始结果、修复过程和最终 QC 都有独立产物，方便定位是哪一步破坏了边界。

#### Stage 2：裁滴与前景融合

- 根据修复 mask 裁出单滴。
- 对过小 bbox 做动态筛除。
- 同时计算局部前景和整图前景。
- 两路都检测到的组件直接保留；仅整图检测到的组件需要通过面积、局部强度和背景条件。
- 去除孤立单像素、小组件和部分边缘反光。

软件工程意义：局部算法保留细节，整图算法提供统一参考；两路融合比只依赖单个阈值更容易解释和调试。

#### Stage 3：component graph

一个液滴对应一张图：

- 节点：清理后前景的 connected component。
- 节点属性：固定 7D 形态特征，包括面积比例、宽高比例、长宽比、填充率和圆度等。
- 边：组件质心构造的无向空间 4-NN。
- 时间：2h/4h one-hot，在 graph readout 后拼接一次。
- 模型：当前保留每种抗生素独立训练。

#### Stage 4：整图聚合

冻结单滴模型后比较：

- H0：预测为 RES 的液滴数 `K >= 10`。
- H1：按有效液滴数换算为全局匹配比例。
- H2：只根据训练集的 `drug × hour` 平均液滴数计算分组比例。

三种方法都不使用未见图标签调阈值。

### 3. 为什么引入论文与 GNN：1 分钟

参考论文：Ali et al., Nature Communications 2025，研究空间组织是否能够提高 tissue-level phenotype prediction。

论文不是简单声称 GNN 必然更好，而是比较：

- 直接平均节点属性的 pseudobulk。
- 保留节点但不使用空间边的模型。
- 使用空间邻接的 GNN。
- density、node identity shuffle 和 permutation 等结构基线。

项目映射：

| 论文 | 液滴项目 |
| --- | --- |
| 一张组织图像 | 一个液滴 |
| 一个已分割细胞 | 一个前景连通组件 |
| 单细胞分子属性 | 组件 7D 形态属性 |
| 细胞空间邻接 | 组件质心 4-NN |
| 组织图像表型 | 单滴 RES/SUS 证据标签 |

必须说清：这是架构与消融逻辑的迁移复现，不是用论文原始数据完整复现论文。

### 4. 论文结构复现实验：2 分钟

固定条件：

- 2,733 个标注 droplet。
- 17 个 image/timepoint `sample_id` 组。
- 4 种药物。
- 3 个随机种子：13、37、101。
- 固定 grouped development split。
- 目标参数预算约 12,289。

八个主要变体覆盖：

1. raw-feature mean / pseudobulk。
2. Gaussian weighted neighbor mean + final-layer mean。
3. 参数匹配的 GIN final mean。
4. 作者发布代码版 `h1/h2` layerwise mean-concat。
5. 论文 Methods 文字版 `h0/h1/h2`。
6. 参数匹配 no-edge 对照。
7. same-width final。
8. same-width layerwise。

总计：

```text
4 drugs × 8 variants × 3 seeds = 96 次训练
65,592 条逐 droplet 预测
96 个 embedding 文件
graph failure = 0
```

为了保证复现：

- 保存 split manifest 和 SHA-256。
- 保存参数合同、checkpoint、history、prediction 和 embedding。
- checkpoint 重载后做概率 parity 检查。
- 从导出预测独立重算 AP、nAP、ROC-AUC、Brier 等指标。
- paired delta 的独立复算误差达到机器精度量级。

### 5. 实验结果与工程决策：1～2 分钟

核心结果：

- Layerwise GIN 相比 raw mean 有提升，但同时改变了容量、编码器和邻居通信，不能把提升单独归因于 topology。
- 真正的 layerwise 与 final-layer 参数匹配对照没有稳定优势。
- Methods 的 `h0/h1/h2` 版本也没有稳定胜过发布代码的 `h1/h2`。
- observed 4-NN edges 对 CIP 为正，对 Ceft 为负。
- Ceft+CIP development holdout 的原始 AP 宏平均变化约为 `-0.0065`，接近于零。

最终决策：

1. 固定 7D morphology 特征，不因为复杂度增加强度和绝对位置特征。
2. 以 final-layer mean 为主，layerwise embedding 只作辅助分析。
3. 保留按药物独立模型，不让共享模型直接替换模型库。
4. 所有拓扑结论逐药报告，不能只报一个被 prevalence 放大的综合指标。

这段的面试重点：

> 面对论文和复杂模型，我没有把“实现成功”当成“方案有效”，而是通过控制参数、split 和 seed 做因果更清晰的对照，并根据结果选择更稳、更简单的结构。

### 6. 整图未见样本与失败定位：1 分钟

主要检查范围：

- 10 张未见 image/timepoint bags。
- 文件名/data-loader 标签为 6 RES、4 SUS。
- 实际独立生物单元少于 10，且没有权威 BMD/MIC 真值。

结果：

- H0、H1、H2 多数票都只与当前文件标签一致 4/10。
- H2 相比 H0 的 balanced accuracy 增量为 0。
- 30 个主要 bag-seed call 中，H0/H1/H2 基本没有改变最终 call。

故障定位：

- Ceft 出现同药同时间 RES/SUS 排序方向冲突。
- PT 的外部系列整体被压到低分区。
- CIP 对时间点和 seed 敏感。
- Ceft 4h SUS 图存在大量气泡相关前景污染。

所以不能通过继续调聚合阈值解决。下一步应优先：

1. 对齐 MIC/BMD 和 breakpoint，建立权威 `image_truth_manifest.csv`。
2. 收集新的独立 batch/isolate。
3. 加强输入和分割 QC。
4. 冻结真值后再校准 soft pooling 和 INDET/NO_CALL。

正确表述：

> 第一次整图外部检查没有达到可用水平，但它排除了聚合阈值是主要瓶颈的假设，并把问题定位到输入污染、跨批次泛化和真值质量。

错误表述：

> 临床准确率是 40%。

因为这 10 张图没有权威临床真值，也不是 10 个完全独立生物重复。

## 早期 CNN / ViT 可复现实验怎么讲

只作为项目演进，控制在 30 秒：

> 早期我比较过 PCA+Logistic Regression、Feature-Stem CNN 和 ViT。Feature-Stem 在 RGB 外增加灰度、CLAHE、Laplacian 和 Sobel 通道，跨组平均 AUC 提高约 0.034、accuracy 提高约 0.053、F1 提高约 0.024；冻结 backbone 的快速 ViT 实验没有超过 Feature-Stem CNN。这说明更大的 backbone 不会自动解决图像质量和数据独立性问题，后续工作因此更强调显式分割、结构表示和受控验证。

边界：ViT 只是单 seed、8 epoch、冻结 backbone 的 quick benchmark，不能概括成 ViT 架构普遍不适合。

## 为什么这个项目匹配软件开发工程师

### 需求到实现

- 从整图耐药判断需求拆出单滴分割、前景识别、结构建模和整图聚合。
- 明确每阶段目标、非目标和接口产物。

### 模块化与可配置

- Python 包按 preprocessing、model、thresholds、evaluation、labeling、experiments 分层。
- 路径和实验作业集中到 YAML 配置。
- 迁移环境主要修改 base directory 和 Fiji executable。

### 集成与故障隔离

- Fiji、Cellpose、OpenCV、scikit-image、PyTorch 等组件串成完整流程。
- 每个阶段保留 CSV、mask、overlay、模型和 summary。
- 可以区分分割错误、前景错误、构图错误、模型错误和聚合错误。

### 测试与复现

- grouped split 防止同一 image/timepoint 的液滴跨分区。
- 多 seed、checkpoint replay、独立重算、参数合同和 SHA-256 审计。
- 不把开发 holdout 包装成独立 biological test。

### 软件质量取舍

- 复杂结构没有稳定收益时，选择更简单、更稳定的 final-layer mean。
- 共享模型未通过安全门时不替换单药模型库。
- 整图失败时停止阈值过拟合，转向补真值和独立数据。

## 高频追问

### 为什么用 GNN，不直接用 CNN？

> 一个液滴中的前景组件数量可变，组件之间又可能存在空间关系，图结构可以显式表示这些对象和关系，也更方便做节点级可视化。但我没有假设 GNN 一定更好，所以设置了 raw mean、final mean 和 no-edge 对照；结果也说明 topology 不是所有药物都有效。

### 为什么选择 7D morphology？

> 我比较了 7D 形态、9D 形态加强度、12D 再加绝对位置，使用验证集和 one-standard-error 规则选择。7D 的 validation nAP、AP 和 ROC-AUC 最好，增加强度和位置反而没有改善，所以选择更简单的 7D。

### 如何防止数据泄漏？

> split 按完整 `sample_id` 分组，同一张图的液滴不跨 split；模型选择只看 validation。但审计仍发现 biological-series overlap，而且 holdout label 参与过 split construction，所以我明确称它为 development holdout，而不是最终独立测试。

### 为什么按药物独立训练？

> 不同药物的形态—标签关系和 topology 增量不同。共享模型在 Ceft、CIP、PT 有正向趋势，但 Genta 只有一个 test 正例 sample、5 个相关正例且 seed 波动很大。当前不足以安全替换单药模型，所以保留单药模型并把共享方案留作后续验证候选。

### 如果 pipeline 某一步偶发失败，怎么定位？

> 先用 sample ID 固定原始输入、版本和配置，再查看 stage summary 判断失败首次出现在哪一层。分割层看 raw/repaired mask 和 QC；前景层看 local/global/fused mask；构图层看 node count、graph failure；模型层通过 checkpoint replay 比较概率；聚合层从逐滴 flag 重算 K 和最终 call。修复后把该 sample 加入回归集。

### 你对性能做了什么？

> 当前工作的重点是正确性和可复现，性能优化主要体现在批处理、缓存参考 mask、避免重复构图和生成可重用中间产物。若进入生产化阶段，我会先用 profiling 拆分 Fiji、分割、前景处理、构图和推理耗时，再决定并行化、GPU batching、内存复用或中间产物缓存，而不是先做无证据优化。

### 当前项目最大的不足是什么？

> 不是模型结构不够复杂，而是独立 biological group 太少、整图真值没有和 MIC/BMD 对齐、部分图像有明显 artifact，导致模型和聚合结果无法升级为临床结论。当前工程 pipeline 已经能稳定运行和定位问题，但业务有效性还需要新的权威数据验证。

## 三句必须守住的边界

1. 这是有监督的 droplet graph learning，不是无监督 GNN。
2. `test` 字段只是 development holdout，不是独立 biological test。
3. 整图 4/10 是与当前文件标签的一致性，不是临床准确率。

## 面试前建议准备的三张图

1. 整图预测 overlay：展示系统如何把单滴结果映射回原图。
2. 单滴 top-scoring contact sheet：展示模型证据和 artifact。
3. H0/H1 下采样稳定性图：展示为什么稳定性不等于正确性。

讲图时固定顺序：

```text
图里是什么
→ 颜色/坐标代表什么
→ 我观察到什么
→ 它支持什么结论
→ 它不能证明什么
```

## 30 秒收尾

> 这个项目让我形成了一套处理复杂工程问题的方法：先把端到端任务拆成有明确输入输出的阶段，再通过可观测中间产物定位故障；对模型或架构选择使用参数受控对照，不因技术复杂就默认有效；最后严格区分开发指标和最终业务结论。我认为这套模块化、测试、故障定位和可追溯方法，可以迁移到复杂的工业和装备软件中。
