# 鸿蒙适配 Skill Harness：工程、规则与路由三层取舍

## 项目定位

这个项目不是“给大模型写一个很长的鸿蒙 Prompt”，而是面向 Coding 场景设计一套可控的 Agent Harness：

> 用 Skill 路由控制鸿蒙能力是否进入上下文，用确定性工程工具构造最小且完整的代码证据，再用可评测、可回滚的规则 Skill 完成风险判断和修改。

三层分别回答三个问题：

| 层次 | 核心问题 | 主要方案 |
| --- | --- | --- |
| 工程侧 | Agent 应该读哪些代码 | 从卡片入口展开真实可达的共享组件；局部规则做有限穷举 |
| 规则侧 | 读到证据后，如何稳定判断和修改 | 参考 Microsoft SkillOpt，对规则 Skill 做受控文本优化 |
| Skill 路由侧 | 当前任务是否应该加载鸿蒙 Skill | 硬条件过滤、候选召回、listwise rerank、hard negative 和 `NO_SKILL` |

这三层是责任划分，不是在线执行顺序。真实执行顺序是：

```text
用户 Coding 任务
  → Skill 路由：选中鸿蒙 Skill 或拒绝加载
  → 工程取证：展开共享组件，构造最小代码证据
  → 规则判断：逐项判断风险并生成修改
  → Verifier：执行静态检查、测试和结果回归
```

故障也可以按层归因：

```text
鸿蒙 Skill 没有加载             → Skill 路由问题
Skill 已加载但关键文件没有进入证据 → 工程召回问题
正确证据已给到但判断或修改错误     → 规则 Skill 问题
修改逻辑正确但构建、测试不通过      → 执行或验证问题
```

---

## 30 秒版本

> 我把鸿蒙适配做成了 Coding Agent Harness 的三层架构。工程侧从卡片入口通过 Babel AST 和业务 Resolver 展开真实可达的共享组件，只在卡片目录内对细碎属性做有限穷举，避免全仓扫描和上下文膨胀。规则侧参考 Microsoft SkillOpt，把规则文档当成冻结模型之外的可优化状态，通过成功、失败轨迹生成有限的 add、delete、replace 修改，只接受在独立验证集上真正提升的版本。Skill 路由侧不只看名字和描述，而是先做适用条件过滤，再从全文索引召回候选并做 listwise 对比排序，同时加入表面相似但功能不同的 hard negative 和 `NO_SKILL` 拒绝选项。三层分别控制“要不要加载、加载后读什么、读到以后怎么做”。

## 两分钟版本

> 这个项目最开始更像一个大范围代码扫描器：先尽可能展开文件，再通过很多排除条件降低误报。实现大约有 2～3K 行，单卡平均扫描 50 多个文件。后来我没有继续堆过滤规则，而是从 Coding Agent Harness 的角度把问题拆成了三层。
>
> 第一层是工程侧，解决 Agent 应该看到什么代码。鸿蒙风险可能出现在卡片目录外的公共组件里，所以我从卡片入口出发，用 Babel Parser 和 Traverse 提取 import、export 以及可静态确定的动态 import，再由自定义 Resolver 处理扩展名、目录入口、alias 和 `index.json` 业务映射，递归得到真实可达的共享组件闭包。对只需要检查当前卡片目录、字面形式稳定的细碎属性规则，不强行全部 AST 化，而是在明确边界内做有限穷举。两路结果按真实路径归一化去重，只把命中代码、位置、规则和完整依赖链交给模型。
>
> 第二层是规则侧，解决正确证据已经给到以后，Skill 怎样稳定判断和修改。我参考 Microsoft SkillOpt 的思想，固定目标模型和执行 Harness，把规则文档作为外部可训练状态。先在训练轨迹中分析成功与失败，再生成有限的 add、delete、replace 规则修改；候选版本只有在独立 selection split 上严格提升才接受，失败修改进入 rejected buffer，避免同一个错误反复加入。当前的 Oracle Checklist 用于固定证据输入，帮助区分“工程没找到”和“规则不会改”，冻结测试集只用于最终验收。
>
> 第三层是 Skill 路由侧，解决表面相似但功能完全不同的 Skill 被误加载。第一阶段先根据任务类型、目标平台和仓库信号做硬性适用条件过滤，再从离线全文索引召回 Top-K；第二阶段读取候选的适用条件和 body 摘要，把候选放在一起做 listwise rerank，比较谁在功能上最适合，而不是逐个判断“好像相关”。评测集会专门加入同领域不同问题、同技术不同用途和能力描述过度泛化的 hard negative。若前置条件不满足、Top-1 分数过低或候选之间无法明确区分，就返回 `NO_SKILL`，不污染执行 Agent 的上下文。
>
> 最终这套 Harness 把模型前后的责任边界拆清楚了：路由层控制是否加载，工程层控制代码上下文，规则层控制判断策略，Verifier 控制修改能否通过。工程实现从约 2～3K 行下降到约 500 行，平均扫描文件从 50 多个下降到约 15 个；规则和路由则用独立指标评测，避免只看最终有没有生成 diff。

---

# 第一部分：工程侧——共享组件展开与最小代码上下文

## 1.1 工程侧解决什么问题

核心问题不是“怎样让 Agent 阅读更多文件”，而是：

> 怎样用确定性程序找到与当前卡片真实相关的代码，同时避免无边界扫描造成 token、延迟和误报增长？

鸿蒙兼容风险主要分成两类：

1. **跨文件、跨目录的共享组件风险**：卡片入口本身没有问题，但它引用的公共组件或传递依赖存在不兼容用法。
2. **卡片目录内的细碎属性风险**：属性名称或调用形式稳定，范围天然局限在当前卡片目录，不需要构造复杂的全仓语义分析。

因此工程侧没有强行使用一种分析方式解决所有问题，而是采用两条链路。

## 1.2 主链路：从卡片入口展开共享组件

```text
卡片入口
  → Babel Parser 生成 AST
  → Traverse 提取 import/export/静态动态 import
  → 业务 Resolver 解析真实路径
  → 递归展开传递依赖
  → visited 去重并防止循环
  → 得到入口真实可达的共享组件闭包
```

### Babel 与 Resolver 的职责边界

- Babel Parser/Traverse 判断“这是不是一条真实依赖声明”。
- Resolver 判断“这条模块标识符最终指向哪个文件”。

Resolver 需要处理：

- 相对路径。
- `.ts`、`.tsx`、`.js`、`.jsx` 等扩展名。
- 目录下的 `index` 文件。
- 项目 alias。
- `index.json` 中与当前卡片上下文相关的业务组件映射。

概念代码：

```ts
function visit(file: string) {
  const realFile = normalizeRealPath(file);
  if (visited.has(realFile)) return;

  visited.add(realFile);

  for (const specifier of extractDependencies(realFile)) {
    const dependency = resolveImport(specifier, realFile, cardRoot);

    if (!dependency) {
      unresolved.push({ importer: realFile, specifier });
      continue;
    }

    addEdge(realFile, dependency);
    visit(dependency);
  }
}
```

`visited` 防止 `A → B → C → A` 无限递归，依赖边仍然保留，因此可以还原完整引用链。

## 1.3 补充链路：卡片目录有限穷举

对于字面稳定、上下文要求低的属性规则：

1. 先确定当前卡片根目录。
2. 只允许指定源码后缀。
3. 排除测试、产物、缓存和第三方目录。
4. 使用确定性 matcher 找出候选位置。
5. 保留文件、行列、片段和命中规则。

这里的关键边界是：

> 有限穷举只负责提高局部候选召回，不代表属性名一出现就直接判错；最终仍由具体风险规则判断其使用方式。

## 1.4 两路结果如何合并

```text
最终候选文件
= 从入口真实可达的依赖文件
∪ 当前卡片目录中命中局部规则的文件
```

合并时按规范化真实路径去重，并保留来源：

```json
{
  "file": "SharedComponent.tsx",
  "sources": ["dependency", "property-search"],
  "rule_id": "harmony-property-usage",
  "location": { "line": 42, "column": 8 },
  "dependency_chain": [
    "CardEntry.tsx",
    "Container.tsx",
    "SharedComponent.tsx"
  ],
  "snippet": "..."
}
```

模型得到的不是一批原始文件，而是可以回答以下问题的证据包：

- 为什么这个文件与当前卡片有关？
- 哪条风险规则命中了？
- 风险代码具体在哪里？
- 当前卡片通过什么路径引用它？

## 1.5 核心工程取舍

| 方案 | 优点 | 问题 | 最终选择 |
| --- | --- | --- | --- |
| 全仓扫描 | 实现直接、候选多 | 无关文件、token、延迟和误报随仓库增长 | 不采用 |
| 所有规则都做 AST | 结构理解能力强 | 小规则也产生大量节点分支和维护成本 | 只用于结构问题 |
| 只扫描卡片目录 | 范围小、速度快 | 会漏掉目录外共享组件 | 作为补充链路 |
| 依赖闭包 + 局部穷举 | 同时覆盖跨目录结构关系与局部属性 | 需要明确规则边界并合并结果 | 最终方案 |

## 1.6 工程侧指标

- 共享组件/关键证据召回率。
- 单卡扫描文件数。
- 单卡分析耗时 P50/P95。
- 未解析依赖数量与比例。
- 平均依赖节点数、依赖边数。
- 交给模型的 token 数。
- 本地确定性规则的误报率。

按原有记录可以说明：

- 实现代码从约 2～3K 行下降到约 500 行。
- 单卡平均扫描文件从 50 多个下降到约 15 个。

代码量下降本身不是正确性指标；它代表删除了大量“先扩大、再逐层排除”的分支，仍需结合召回率和误报率回归。

---

# 第二部分：规则侧——参考 SkillOpt 的受控规则优化

## 2.1 规则侧解决什么问题

工程侧只保证目标文件和证据进入上下文。规则侧解决的是：

> 在正确证据已经给到的条件下，Agent 能否判断需要修改、选择正确规则，并生成命中正确位置的 diff？

因此不能看到最终没有 diff，就直接补 Prompt。需要先区分：

- **有没有读到**：目标文件和关键证据有没有被工程侧召回。
- **读到以后会不会改**：证据已经进入 Checklist 时，Agent 是否判断并修改正确。

## 2.2 对 Microsoft SkillOpt 的借鉴

SkillOpt 的核心思想是：

> 冻结目标模型和执行 Harness，把 Skill 文档本身当成可训练、可验证、可导出的外部文本状态。

对应到鸿蒙规则 Skill：

| SkillOpt 概念 | 鸿蒙项目中的对应物 |
| --- | --- |
| Frozen target model | 固定使用的 Coding 模型 |
| Frozen harness | 固定工程取证、工具和 Verifier |
| Rollout trajectory | 证据、工具调用、判断、diff、测试结果 |
| Scored result | 规则判断、patch 和回归指标 |
| Add/delete/replace edit | 对规则 Skill 的局部增加、删除、替换 |
| Textual learning rate | 单轮允许修改的规则数量/文本范围 |
| Selection split | 判断候选 Skill 是否真的提升的独立数据 |
| Rejected-edit buffer | 记录失败规则修改及造成的回退 |
| Best skill artifact | 通过验证并准备发布的版本化 Skill |

## 2.3 规则优化流程

```text
冻结目标模型、工程取证和 Verifier
  → 在训练集运行当前规则 Skill
  → 收集成功与失败轨迹
  → 按 minibatch 归纳重复失败模式
  → 提议有限的 add/delete/replace 修改
  → 受编辑预算约束生成候选 Skill
  → 在 selection split 上重新执行
      ├─ 严格提升：接受并版本化
      └─ 没有提升：拒绝并写入 rejected buffer
  → 最终 best skill 在冻结测试集验收
```

不能让 optimizer 每遇到一个 badcase 就重写整个 Skill，原因包括：

- 容易修复一个样本却破坏已有正确行为。
- 容易加入相互冲突的规则。
- 容易把具体文件名或具体样本写进规则，产生过拟合。
- 规则变化太大时，无法知道具体是哪项修改带来了提升或回退。

因此每轮修改应该是小步、局部、可审计的。

## 2.4 Oracle Checklist 的作用

Oracle Checklist 不是生产检索方案，而是诊断和规则优化工具：

```json
{
  "evidence_id": "E-1024",
  "rule_id": "harmony-property-usage",
  "file": "SharedComponent.tsx",
  "line": 42,
  "snippet": "..."
}
```

它暂时固定候选和证据输入，排除依赖发现、文件读取和上下文召回的影响：

- 完整 Harness 失败，Oracle Checklist 成功：优先排查 Skill 路由或工程召回。
- Oracle Checklist 已给出正确证据，仍判断/修改失败：进入规则 Skill 优化。

为了证明模型消费了证据，要求输出：

- 对应 `evidence_id`。
- 逐项风险判断。
- 使用的 `rule_id`。
- 是否修改以及不修改的理由。

## 2.5 数据切分与验收

```text
训练集 D_train
  用于产生轨迹、分析失败和提出规则编辑

选择集 D_sel
  用于判断候选 Skill 是否优于当前版本

冻结测试集 D_test
  只用于最终报告，不参与规则修改
```

规则侧可以统计：

- Oracle 证据条件下的风险判断 Precision/Recall/F1。
- 条件修改成功率：证据已召回时，diff 是否正确。
- 过度修改率：原本不应修改的代码是否被改动。
- diff 定位准确率。
- 构建、静态检查和测试通过率。
- Skill token 数和单轮编辑量。

候选 Skill 不应只以单一最终分数盲目接受。可以设置约束：

```text
候选 Skill 可以接受，当且仅当：
1. selection split 主指标严格提升；
2. Precision、过度修改率等安全指标没有越过阈值；
3. 构建和静态检查不回退；
4. Skill 长度和规则数量没有无边界增长。
```

## 2.6 面试真实性口径

如果没有完整实现独立 optimizer、自动编辑预算和 rejected buffer，应表述为：

> 我参考 SkillOpt 的受控文本优化思想设计规则调优流程，实际重点落在轨迹归因、局部规则修改、独立验证集门禁和冻结测试集回归。

不要表述为：

> 我完整复现了 SkillOpt，或者取得了论文中报告的提升。

论文指标只能说明方法研究结果，不能作为当前鸿蒙项目的项目指标。

---

# 第三部分：Skill 路由侧——避免加载表面相似但功能无关的 Skill

## 3.1 Skill 路由侧解决什么问题

路由层的目标不是“从候选中找一个语义最像的名称”，而是：

> 判断当前任务是否真的需要某项能力；如果需要，在多个表面相似的 Skill 中选择功能最匹配的一个；如果不需要，明确返回 `NO_SKILL`。

典型误召回：

| 用户真实任务 | 表面相似但错误的 Skill | 错误原因 |
| --- | --- | --- |
| 检查现有业务卡片的鸿蒙兼容风险 | ArkTS 原生组件开发 | 都提到鸿蒙和组件，但一个是兼容审查，一个是从零开发 |
| 展开卡片引用的共享组件 | 通用 npm 依赖安全审计 | 都提到依赖，但一个分析源码引用闭包，一个分析软件包漏洞 |
| 修复鸿蒙平台属性用法 | 多端 UI 视觉迁移 | 都提到跨平台适配，但输入、证据和产出不同 |

如果只把 Skill 名称和简短 description 交给 Agent，关键词重合很容易造成误加载。

## 3.2 Skill Manifest：先明确适用与排除条件

路由信息不能只有“这个 Skill 能做什么”，还要包含“什么时候不能用”。

示例：

```yaml
name: harmony-card-compatibility
description: 检查并修改业务卡片及其共享组件中的鸿蒙兼容风险

applies_when:
  - 当前任务属于 Coding、代码审查或代码修改
  - 用户目标是鸿蒙兼容性检查、适配或风险修复
  - 仓库中能够识别卡片入口、卡片目录或业务组件映射

requires:
  - 至少能够定位一个卡片入口或目标组件
  - 允许读取目标仓库源码

not_when:
  - 从零开发原生 ArkTS 页面或组件
  - 普通 React 重构且没有鸿蒙目标
  - 仅回答鸿蒙知识、API 或产品问题
  - 仅因为任务中出现“组件”“兼容”“依赖”等相似词

output_contract:
  - 命中的代码位置
  - 对应规则
  - 完整依赖证据链
  - 修改 diff 或明确的不修改理由
```

## 3.3 两阶段路由

```text
任务意图 + 仓库浅层信号
  → 硬性适用条件过滤
  → 第一阶段：从全文索引召回 Top-K
  → 第二阶段：对相似候选做 listwise rerank
  → 置信度、候选差距和 NO_SKILL 门禁
  → 只加载最终选中的 Skill
```

### 第一阶段：候选召回

第一阶段追求召回率，目标是不要过早漏掉正确 Skill。

可使用：

- 任务意图与 Skill 文本的 embedding 相似度。
- BM25 等关键词信号。
- 分类标签：Coding、平台、框架、输入和输出类型。
- 仓库浅层信号：文件类型、manifest、卡片入口、业务映射是否存在。

Router 可以离线索引 Skill 的 name、description、适用条件和 body；执行 Agent 不需要读取所有候选全文。候选 Skill 只有最终被选中后，正文才注入执行上下文。

### 第二阶段：listwise rerank

第一阶段召回的候选可能都“看起来相关”：

```text
用户任务：检查某张业务卡片引用的共享组件是否存在鸿蒙兼容风险

候选 A：鸿蒙卡片兼容性检查
候选 B：ArkTS 原生组件开发
候选 C：通用前端依赖分析
候选 D：多平台 UI 迁移
```

Pointwise 判断会逐个问：

```text
候选 A 与任务相关吗？相关。
候选 B 与任务相关吗？好像也相关。
候选 C 与任务相关吗？也有依赖分析。
候选 D 与任务相关吗？也属于跨平台。
```

四个候选可能都得到较高分。

Listwise rerank 会把候选放在一起比较：

```text
当前任务需要哪些必要能力？
哪个候选完整满足这些能力？
哪些候选只是关键词相似，输入或目标不同？
如果没有任何候选满足，是否应该返回 NO_SKILL？
```

期望输出：

```json
{
  "selected_skill": "harmony-card-compatibility",
  "ranking": [
    "harmony-card-compatibility",
    "frontend-dependency-analysis",
    "multi-platform-ui-migration",
    "arkts-component-development"
  ],
  "matched_requirements": [
    "卡片级代码分析",
    "共享组件依赖展开",
    "鸿蒙兼容规则检查"
  ],
  "rejected_candidates": [
    {
      "skill": "arkts-component-development",
      "reason": "用于原生组件开发，不负责现有业务卡片兼容审查"
    }
  ],
  "confidence": 0.91
}
```

一句话说明 listwise rerank：

> 第一阶段回答“哪些 Skill 可能相关”，第二阶段回答“在这些相似候选中，谁在功能上最适合”；它把判断从独立的像不像，变成候选之间的相对比较。

## 3.4 Hard Negative：专门测试表面相似、功能不同

普通随机负样本太容易，无法验证路由器是否真正理解 Skill 能力边界。应加入三类 hard negative：

1. **同领域、不同问题**：同样是鸿蒙，但一个负责兼容审查，一个负责原生应用开发。
2. **同技术、不同用途**：同样使用 AST，但一个展开源码依赖，一个做代码格式化或安全扫描。
3. **能力描述过度泛化**：描述声称“处理所有前端兼容问题”，实际没有卡片入口解析和共享组件证据链。

还可以从线上误召回中挖掘 hard negative：

```text
如果错误 Skill 经常出现在正确 Skill 的 Top-K 中，
就把这对 query/skill 加入路由回归集，要求后续版本显式拉开排名。
```

## 3.5 `NO_SKILL` 与拒绝加载

路由器不能被强迫每次都选一个 Skill。以下情况应返回 `NO_SKILL` 或请求澄清：

- 所有候选都未通过硬性 `applies_when/requires/not_when` 条件。
- Top-1 绝对分数低于阈值。
- Top-1 和 Top-2 差距过小，功能边界仍然不明确。
- 仓库缺少执行该 Skill 必需的入口或业务信号。
- 用户任务只是知识问答，不需要执行 Coding 流程。

这里要区分两个信号：

- **Top-1 分数低**：可能根本没有适用 Skill。
- **Top-1 与 Top-2 差距小**：可能存在候选歧义，需要二次判别或澄清。

`NO_SKILL` 是一等路由结果，不是异常兜底。

## 3.6 小规模 Skill 库的工程取舍

研究论文使用专门训练的 bi-encoder 和 cross-encoder 处理大规模 Skill 库，但当前项目的 Skill 数量如果不大，没有必要训练独立的 1.2B Router。

更合理的轻量方案：

```text
少量 Skill
  → 结构化 applies_when/not_when 硬过滤
  → embedding/BM25 召回 Top-3 或 Top-5
  → LLM 根据完整适用条件做 listwise 排序
  → 阈值 + NO_SKILL
```

只有在 Skill 数量、QPS 或路由成本明显增长以后，才考虑：

- 离线预计算全量 Skill embedding。
- 训练轻量 bi-encoder。
- 训练专用 cross-encoder reranker。
- 对历史误召回做 hard-negative mining。

这体现的工程取舍是：

> 借鉴论文的检索、重排和 hard-negative 原则，但根据 Skill 库规模选择最小可用实现，而不是为了复现论文堆模型。

## 3.7 路由侧指标

- Activation Precision：加载的 Skill 中有多少真正适用。
- Activation Recall：应该加载时是否成功加载。
- False Activation Rate：无关任务上误加载 Skill 的比例。
- `NO_SKILL` Accuracy：没有适用 Skill 时能否正确拒绝。
- Hard-negative Hit@1：相似干扰项存在时，正确 Skill 是否排名第一。
- Top-K Recall：正确 Skill 是否进入第一阶段候选。
- 路由延迟和 token 成本。

不能只统计“正确 Skill 存在时的 Hit@1”，否则无法衡量路由器是否知道什么时候不应该加载任何 Skill。

---

# 三层如何共同评测

## 分层指标

| 阶段 | 主要指标 | 失败时优先优化什么 |
| --- | --- | --- |
| Skill 路由 | Activation Precision/Recall、hard-negative Hit@1、`NO_SKILL` Accuracy | Manifest、召回、rerank、阈值与负样本 |
| 工程取证 | 关键证据召回率、扫描文件数、未解析依赖、token | Parser、Resolver、搜索边界 |
| 规则判断 | Oracle 条件下的判断 F1、条件修改成功率、过度修改率 | 规则表达、示例、编辑策略 |
| 执行验证 | 构建/测试通过率、最终 diff 正确率 | 修改执行、Verifier、回滚策略 |

## 可观测事件

每次执行至少记录：

```json
{
  "route": {
    "selected_skill": "harmony-card-compatibility",
    "candidates": ["..."],
    "confidence": 0.91,
    "rejected_reasons": ["..."]
  },
  "context": {
    "entry": "CardEntry.tsx",
    "files_scanned": 15,
    "unresolved_dependencies": [],
    "evidence_ids": ["E-1024", "E-1025"]
  },
  "rule_execution": {
    "consumed_evidence_ids": ["E-1024", "E-1025"],
    "rule_ids": ["harmony-property-usage"],
    "patch_generated": true
  },
  "verification": {
    "static_check": "passed",
    "tests": "passed"
  }
}
```

这样最终任务失败时，不需要只看一份最终 diff 猜原因。

---

# 高频追问

## 1. 为什么把它叫 Agent Harness，而不是代码扫描脚本？

> 因为确定性扫描只是工程取证的一部分。完整系统还控制 Skill 是否加载、代码上下文如何构造、规则如何版本化和评测、修改后如何验证。模型能力不变，但模型外的路由、上下文、工具、规则和 Verifier 共同决定最终行为，这些组成了 Harness。

## 2. 为什么工程侧不能直接把整个仓库交给模型？

> 全仓上下文会把不可达组件、废弃文件和其他卡片带入判断，token、延迟和误报随仓库增长。依赖闭包可以给出“为什么这个文件与入口有关”的证据，局部穷举只在明确业务边界内补召回。

## 3. 为什么规则侧不能出现 badcase 就加一条 Prompt？

> 单样本修补容易过拟合，也可能破坏已有正确行为。更稳妥的是从一批成功和失败轨迹归纳重复模式，限制每轮编辑量，再通过独立 selection split 决定是否接受。

## 4. Listwise rerank 到底解决什么？

> 第一阶段返回的候选通常都在同一领域，逐个判断可能全部得到高相关分。Listwise 把候选放在一起，围绕任务的必要能力做相对比较，专门排除关键词相似但输入、目标或执行流程不同的 Skill。

## 5. 为什么不只把 Skill description 写得更详细？

> 更清晰的 description 有帮助，但相似 Skill 的关键差异可能藏在适用条件、执行步骤、输入输出契约和正文中。路由器可以使用离线全文信号，执行 Agent 仍只加载最终选中的 Skill，兼顾区分能力与上下文成本。

## 6. 路由器使用 Skill body，会不会违背渐进式加载？

> 不违背。全文由独立 Router 离线索引或在候选重排时读取，不代表把所有 Skill body 注入执行 Agent。渐进式加载控制的是执行上下文；最终只有通过路由门禁的 Skill 被加载。

## 7. 如果没有任何适用 Skill 怎么办？

> `NO_SKILL` 是正式类别。硬性前置条件失败、Top-1 分数低或候选无法明确区分时，不加载 Skill，必要时请求用户补充目标，而不是从低质量候选里强行选一个。

## 8. SkillRouter 论文的方法是否完整解决了误加载？

> 它重点解决大规模候选池中的召回与排序，尤其是相似候选之间的功能区分；项目里还需要额外加入 `NO_SKILL`、适用条件门禁和无关任务回归集，才能评测“根本不应该加载 Skill”的情况。

## 9. 规则优化和路由优化会不会互相污染？

> 会，所以要分开评测。规则 Skill 优化时通过 Oracle Checklist 固定正确证据和已选 Skill，排除路由与工程召回；路由评测则固定 Skill 内容，专门测是否选对。分层通过后再跑端到端回归。

## 10. 如果继续优化，会做什么？

- 对 AST 和 Resolver 结果做内容哈希缓存。
- 基于 Git diff 做增量依赖分析。
- 从线上误召回自动挖掘 hard negative。
- 版本化 Skill manifest、规则正文和评测集。
- 为路由、工程召回、规则判断和 Verifier 建独立看板。
- 对动态路径和运行时注册增加低置信度与人工兜底。

---

# 面试真实性边界

- 工程侧可以讲实际实现和已有测量：Babel AST、自定义 Resolver、局部有限穷举、代码量和扫描文件数变化。
- 如果没有训练独立路由模型，就说“embedding/BM25 召回 + LLM listwise 重排”，不要说训练了 SkillRouter 论文模型。
- 如果没有完整复现 SkillOpt，就说“参考 SkillOpt 的受控文本优化思想”，并明确自己实际实现了哪些环节。
- 论文中的 Hit@1、性能提升和模型规模属于论文结果，不能当成项目结果。
- 路由层新增后应重新统计 Activation Precision、False Activation Rate 和 `NO_SKILL` Accuracy，不能沿用原工程扫描准确率。
- 修改候选生成或规则 Skill 后，必须重新跑同一冻结测试集，不能只展示几个成功案例。

---

# 参考资料

1. Microsoft Research, [SkillOpt: Executive Strategy for Self-Evolving Agent Skills](https://www.microsoft.com/en-us/research/publication/skillopt-executive-strategy-for-self-evolving-agent-skills/)
2. SkillOpt 论文全文：[arXiv:2605.23904](https://arxiv.org/html/2605.23904v2)
3. [SkillRouter: Skill Routing for LLM Agents at Scale](https://arxiv.org/html/2603.22455v5)
4. ACL 2026, [Do LLMs Know Tool Irrelevance? Demystifying Structural Alignment Bias in Tool Invocations](https://aclanthology.org/2026.acl-long.1473/)
5. Microsoft Agent Framework, [Agent Skills 与 Progressive Disclosure](https://learn.microsoft.com/en-us/agent-framework/agents/skills)

---

# 最后收口

> 这个项目最关键的不是用了 AST、Prompt 或向量检索中的某一个技术，而是把 Coding Agent 的失败拆成了三个可独立优化的层次：Skill 路由负责避免错误能力进入上下文，工程取证负责构造最小但完整的代码证据，规则优化负责让冻结模型在正确证据上稳定判断和修改。不同问题交给最适合的确定性程序、检索模型或语言模型处理，再通过分层评测和 Verifier 把它们组合成一套可观测、可回归的 Agent Harness。
