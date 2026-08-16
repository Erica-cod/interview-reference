# 合合信息 AI 全栈开发：字节 AI Native 经历高压追问小抄

> 目标岗位：[27 届校招-AI 全栈开发工程师](https://intsig.zhiye.com/campus/detail?jobAdId=b63e05c4-5fc0-41fa-8db3-1e7a17836efc)
>
> 项目证据：[20 分钟答辩主讲稿](/Users/bytedance/Downloads/20min-defense-speech.md:5)｜[转正答辩文档](https://bytedance.larkoffice.com/wiki/P5LdwVldliCD8XkN7bacbBF8nZg)｜[代码改造 Skill 评测思路](https://bytedance.sg.larkoffice.com/docx/L7dldJbIyou1bYxnkSql8OlVguf)｜[主 Workflow Skill](/Users/bytedance/LinaMono/lina-mono/lina-mono/.agents/skills/lina-search-card-workflow-b-profile-v2/SKILL.md:1)

## 0. 先背这五句话

1. **我做的不是一个“会写代码的 Agent”，而是一套面向搜索卡片交付的 AI Native 研发控制面。**
2. **人的研发目标先被编译成 Spec、修复计划和验收合同；Agent 负责找证据、生成候选修改并执行，独立验证结果决定能不能交付。**
3. **确定性工具负责范围和状态，模型负责语义判断，测试负责外部裁决，人负责目标、权限和高风险准入。**
4. **测试不是编码完成后的 QA 兜底，而是开发控制环的一部分：编码前确认覆盖，编码中 Red/Green，编码后做全量回归和真实终端验收。**
5. **我的完成标准从“AI 给出了结果”变成“代码、配置、服务端数据和真机用户结果都有独立证据”。**

一句话总定位：

> 我在字节把鸿蒙搜索卡片适配中反复出现的代码、配置和服务端问题，沉淀成规则、评测集和可组合 Skill，再通过公共研发主干、领域 Profile、交付状态机和真机验收，把“人串流程”改造成“人给目标，AI 在可验证边界内闭环交付”。

## 1. 为什么这段经历正好匹配岗位

岗位不是只要求“会用 Copilot”，而是同时要求全栈交付、AI Native 研发、Agent 项目和研发流程沉淀。我的字节经历可以分别证明：

| 岗位要求 | 字节经历证据 | 面试落点 |
| --- | --- | --- |
| 端到端产品交付 | 搜索卡片交付横跨前端代码、TCC 配置、服务端数据和鸿蒙真机 | 我对用户可见结果负责，不把边界停在前端 Diff |
| AI 辅助编码 | 规则扫描、修复计划、Spec-to-Code、CR、验证和 MR 交付 | AI 不是聊天插件，而是研发主流程里的执行者 |
| 标准化研发范式 | 历史 MR 净化、范围/业务双维评测、Bad Case 分层、Skill 与规则库分离 | 我能量化效果，并把工程缺陷和规则缺陷分开优化 |
| Agent 项目交付 | 按 Business 拆 deliveryUnit，每个单元绑定 Demand、分支、MR、Pipeline | 长流程有状态、有隔离、有失败恢复 |
| 架构与技术方案 | 公共 Spec 主干 + 轻量 Domain Profile + 原子 Skill | 共性流程复用，领域知识按需注入 |
| 产品与协作 | 从 QA 入测阻塞和上线延误中定义问题；配置、服务端和 QA 联合闭环 | 从真实交付问题出发，不为做 Agent 而做 Agent |

这段经历主要证明 **AI Native 工程、复杂链路治理和可验收交付**。如果面试官继续追后端框架、数据库、SSE、鉴权和部署，再接 AI 兴趣教练项目中的 React、BFF/Hono、MongoDB、Redis、OIDC、SSE 和容器化，不要硬把一个项目承担岗位的全部证明责任。

## 2. 30 秒、90 秒和 3 分钟项目介绍

### 2.1 30 秒版本

> 我在字节独立负责抖音搜索卡片的鸿蒙适配。这个任务表面是前端迁移，实际要同时保证服务端有数据、TCC 配置正确、代码兼容鸿蒙，最后在真机真实 query 下能出卡。我的核心工作是把一线经验沉淀成可评测的规则 Skill，再把代码扫描、Spec/Coding、配置诊断和真机验收串成公共交付主干。模型只负责语义判断和候选修改，依赖范围、状态、权限和验收由确定性系统控制。最终形成了从研发目标到真实终端证据的 AI Native 交付闭环。

### 2.2 90 秒版本

> 我在字节独立负责抖音搜索卡片的鸿蒙适配，支撑了日常 117 张卡和高考专项 50 处交付，覆盖 71.2% 的高 PV 场景。这个任务并不只是改前端：服务端要有数据，TCC 要满足鸿蒙出卡条件，前端代码要兼容，最后还要在真机真实 query 下正常出卡。
>
> 我的工作经历了三步。第一步，我把高频鸿蒙适配经验沉淀成规则 Skill，并建立了真实历史任务回放：先从历史 MR 中清理掉顺手重构和无关修复，得到预期文件范围；再反推只描述业务目标的真实需求，让执行模型只看到 Skill、需求和改造前代码。最终同时看文件范围的准确率、召回率、噪音率，以及关键业务目标的一致性得分，而不是把某个扫描 P/R 当成 MR 正确率。
>
> 第二步，我把完成标准从“代码改完”扩展到“真机能出卡”。对目标卡片主动检查 TCC；遇到不出卡时用同一 LogID 联合定位配置和服务端最早断点。这个能力扫描约 2,000 张存量卡，定位并处理 24 处配置遗漏，扫描加人工复核约 3 小时。
>
> 第三步，我把这些单点能力编排成公共研发主干：明确需求直接生成 repair plan，高层适配需求先做依赖展开和领域规则判断；再按 Business 拆交付单元，进入 Spec、Coding、整体回归、本地产物验证和真实 query 验收。我的核心方法是：人定义目标和验收合同，确定性工具控制范围与状态，AI 做语义和实现，独立证据决定是否交付。

### 2.3 3 分钟版本

> 我想讲的不是一个“AI 自动写代码”的 Demo，而是如何把 AI 放进真实研发交付。
>
> 搜索卡片迁到鸿蒙有四层正确性：服务端有数据、TCC 配置符合出卡条件、客户端代码兼容、真机用户行为正确。早期我只盯前端代码，AI 扫描通过后就认为完成；高考专项里有一张卡因为漏查 TCC 延后一天上线，后续批量不出卡又影响 QA 入测。这让我意识到，流程跑完不等于任务正确，生成代码的模型也不能同时担任裁判。
>
> 第一阶段我先建立外部评判标准。代码改造没有唯一标准答案，所以我不逐行比 Diff，而是从真实历史 MR 构造回放集：强模型拿完整上下文净化人工 MR，删除顺手修 Bug、重构和格式化，产出预期修改文件集；再反向生成只描述业务目标的自然需求。执行阶段固定普通模型，只允许看到 Skill、需求和改造前代码；结束后由独立 Judge 将任务拆成关键业务目标，联合评估文件范围和业务结果。文件侧看准确率、召回率和噪音，业务侧看关键目标完成、硬约束、少改多改和额外语义变化；两类门禁同时通过才算 PASS。
>
> Bad Case 也不再统一归因于 Prompt。我固定模型、Skill 和 Case 做一个反事实上下文实验：原始入口漏检时，只额外把共享组件和调用关系单独喂给 AI；如果同一错误簇多次回放都由失败转成功，就把它作为 Context Defect 的强信号，优先改依赖展开、组件发现和加载顺序。只有正确文件和证据已经进入上下文，模型仍然漏判、误判或给错适配动作，才进入鸿蒙规则调优。规则侧参考微软 2026 年 SkillOpt，把 Rule ID 当文本参数：每个优化 step 最多做 4 个原子 add/delete/replace，后期衰减到 2 个；候选在独立验证集上必须严格提升，持平也拒绝，被拒修改进入负反馈，最终只发布 `best_skill`。
>
> 第二阶段我把正确性扩展到全链路。进测前可以按卡片主动检查 schema、channel、SURL 等鸿蒙配置；发生不出卡时，使用同一个 LogID 联合排查 search_api 和 search_service，从 Trace、Loader/Packer、过滤原因和下游响应中找最早断点。确认是下游空就交给责任服务，命中配置线索再进入 TCC 修复；诊断权限不会自动扩大成写配置、Push 或部署权限。这样既避免 Agent 猜根因，也避免一次只修表象。
>
> 第三阶段我把流程抽象成公共 Spec 主干加轻量 Domain Profile。需求动作明确时走 `explicit`；只有“鸿蒙适配”“大屏适配”这类高层目标时走 `rule_driven`，先加载对应规则形成逐卡 repair plan。随后按 Business 拆 deliveryUnit，一个单元对应一个 Demand、分支、MR 和 Pipeline，共享文件指定唯一 Owner。编码后 `local_dev` 验证当前本地产物，涉及 TCC 时还必须跑 `query_search` 验证真实环境，两种证据不能互相替代。
>
> 在这套架构上，我进一步把正确性变成验收合同：编码前将需求条款映射成 Case 和真机计划并冻结；Coding Agent 只能根据失败证据把 Case 从 Red 推到 Green，不能自己修改验收标准；重构后再跑同一套回归和真机验收。失败时分成 Spec、Case、代码、工程上下文、鸿蒙规则和配置/执行环境，进入不同回路，而不是所有问题都回到 Coding 继续打补丁。
>
> 所以这个项目真正的价值不是 AI 生成了多少代码，而是我把模型的不确定性放在可控位置，并用状态、权限、可追踪测试和真实终端证据把它变成可交付的工程能力。

## 3. 系统架构：面试时按这条链路讲

```text
研发目标 / PRD
    ↓
需求合同：目标、非目标、业务不变量、验收标准
    ↓
planningMode
    ├─ explicit：改动点明确，直接形成 repair plan
    └─ rule_driven：高层目标
          ↓
       确定性依赖展开
       searchScope + chain + unresolved
          ↓
       Domain Profile
       ruleSkill / configSkill / presenter
    ↓
按 Business 拆 deliveryUnit
一单元 = 一 Demand + 一分支 + 一 MR + 一 Pipeline
    ↓
RFC / Spec → 任务拆解 → Coding → 任务级 Eval → 整体回归 → CR
    ↓
本地产物验证 local_dev
    ↓
配置 / 真实环境验证 query_search
    ↓
证据包：需求覆盖、Diff、测试结果、配置结果、真机结果
    ↓
人工授权 Push / 部署 / 高风险准入
```

### 3.1 哪些地方用模型，哪些地方不用

| 环节 | 主要执行者 | 原因 |
| --- | --- | --- |
| 需求归纳、规则语义判断 | 模型 | 输入开放、存在语义差异 |
| 依赖范围、ID、状态迁移 | 确定性代码 | 要可重复、可追踪、可恢复 |
| 生成候选修改 | 模型 + Coding Worker | 需要理解局部业务代码 |
| Lint、类型检查、单测、契约测试 | 工具 | 有明确 oracle，不需要模型猜 |
| TCC/服务端链路判断 | 工具证据 + 模型归纳 | 日志和 Trace 是事实源，模型只做归纳 |
| 真机出卡与固定交互 | 自动化工具 | 验证用户真实可见结果 |
| 测试覆盖确认、写配置、Push、部署 | 人工门禁 | 错误成本或权限风险高 |

一句话收口：

> 外层是确定性 Workflow，内层在需要语义判断和代码生成的节点使用 Agent；不是把整个研发过程交给一个自由循环的模型。

### 3.2 协议化状态机和七道闸口

```text
spec_unapproved
→ spec_approved
→ repo_aligned
→ demand_created
→ execution_ready
→ implementing
→ verified
→ self_tested
→ deployed

任意阶段证据不足 → blocked
```

状态不是靠 Agent 自述推进，而是靠 guard：

- RFC 未批准，不编码。
- Repo 未对齐，不拆任务。
- 没有真实 Demand、Branch 和文件边界，不进入实现。
- 当前任务没有独立 Eval 证据，不进入下一任务。
- 整体需求覆盖、关键命令和回归风险未通过，不进入 `verified`。
- 设备结果不能关联到当前本地产物，不进入 `self_tested`。
- 自测未通过，不 Push；代码、TCC、Push、部署分别授权。

这里要强调“协议化”：Skill、artifact 和 guard 共同定义状态契约；如果面试官追问事务、事件持久化和并发恢复，可以顺势说下一层平台化就是把同一契约下沉成可执行控制面，而不是重新设计业务流程。

## 4. 核心立场：怎么证明 Agent 改的代码是对的

### 4.1 先纠正问题定义

不要回答“Agent 可以证明自己改对了”。更强的回答是：

> 代码正确不是模型的一个布尔判断，而是相对于一份验收合同成立。Agent 只能产出候选，独立、冻结、可重复的 oracle 才能给出交付结论。

“正确”至少同时包含：

```text
需求是 ABC
→ A、B、C 都实现了：没有少改
→ 没有偷偷变成 ABCD：没有多改
→ 原有行为没有退化：没有回归
→ 当前产物在目标环境生效：不是只改了源码
→ 用户真实路径成立：不是只跑通命令
```

### 4.2 不是“把问题扔给测试”，而是“把判断编译成测试”

面试官：

> 所以你实际上没有在开发阶段判断代码改对没有，而是改完以后交给测试兜底？

推荐回答：

> 不是。我把测试设计放在编码之前，把测试执行放在编码循环里，把真实环境验收放在进入 QA 之前。以前的问题是编码 Agent 先自由实现，再用同一套逻辑自检；改进后，需求和历史风险先被编译成冻结 Case，Coding Agent 只根据失败证据修改，直到同一批 Case 从 Red 到 Green。跑绿之后还要经过配置、服务端和真实终端验收。所以我不是放弃开发阶段的判断，而是把昂贵、不可重复的模型主观判断，替换成可执行、可回归的工程证据。

为什么这样更经济：

> 如果每次修改都让模型重新加载整个仓库、业务背景、调用链和隐含约束，再用长链推理判断自己是否正确，token、延迟和不稳定性都很高。稳定约束适合一次性编译成测试、规则和门禁；后续编码只消费明确失败信号。

### 4.3 TDD 式研发闭环

```text
已批准 Spec
→ 测试设计：自动化 Case + 真机计划
→ 覆盖确认：需求/规则是否都有对应 oracle
→ 冻结验收合同
→ 自动化 Case 先失败（Red）
→ Coding Agent 实现
→ 同一批 Case 通过（Green）
→ Refactor
→ 同一批 Case + 全量回归再次通过
→ local_dev 验证当前本地产物
→ query_search / 真机验收真实环境
→ 汇总证据，等待 Push / 部署授权
```

编码阶段只有一个核心目标可以这样说：

> 在不修改验收合同、不越出批准范围、不破坏既有行为的前提下，把冻结 Case 全部跑绿。

“全绿”是必要条件，不是唯一条件；还要经过覆盖确认、Diff 范围审计和真实环境验收。

### 4.4 六层正确性合同

| 层级 | 要回答的问题 | 主要证据 |
| --- | --- | --- |
| L0 需求覆盖 | 每条需求、非目标和不变量是否有 oracle | `requirement_id → case_id` 追踪矩阵 |
| L1 范围覆盖 | 该读的文件、组件和配置是否进入上下文 | `searchScope`、依赖链、`unresolved`、证据召回率 |
| L2 代码行为 | 实现是否满足正向、反向和边界行为 | 单测、契约测试、类型检查、Lint、历史回归 |
| L3 集成链路 | 配置、服务端数据和接口契约是否成立 | TCC 检查、Trace、下游响应、集成测试 |
| L4 当前产物 | 验证的是不是本次源码构建出来的产物 | `local_dev`、构建标识、预览结果 |
| L5 用户结果 | 真实 query、真机出卡、样式和交互是否正确 | `query_search`、截图、组件树、固定交互证据 |

### 4.5 验收 Case 的最小结构

```json
{
  "requirement_id": "R-07",
  "precondition": "AB 开启，Harmony 环境，目标卡命中",
  "input": "固定 query / 固定数据",
  "action": "进入搜索页并触发卡片",
  "expected": "使用新版格式且卡片正常出卡",
  "invariants": ["AB 关闭保持旧行为", "其他端不受影响"],
  "scope_hint": ["候选调用点", "配置项", "共享组件"],
  "verification": ["unit", "local_dev", "query_search"]
}
```

Case 必须同时覆盖：

- 正向路径：ABC 都出现。
- 反向路径：不满足前置条件时不能误触发。
- 边界路径：阈值、空数据、异常状态、超时。
- 兼容路径：AB 关闭、其他平台、旧接口。
- 不变量：不能新增 D、不能删除既有 B。
- 历史回归：过去真实 Bad Case 不能复发。

### 4.6 测试 Case 本身怎么保证没写错、没写漏

面试官继续追问：

> 你只是把正确性问题转移给测试，那怎么证明 Case 是完整的？

回答：

> Case 不由 Coding Agent 自己定义。我先建立 `需求/规则 ID → Case → 计划项 → 变更文件 → 验收证据` 的双向追踪。每条需求至少有正向、反向或边界 oracle；每个实际 Diff 也必须反查到需求或技术约束。编码前由人或独立 Test Designer 做覆盖确认并冻结；Coding Agent 对测试目录只读。关键任务再保留隐藏回归集，防止针对可见 Case 写特例。新的线上 Bad Case 会先成为候选 Case，经过历史回放和人工评审后再进入正式回归集。

再补一层：

> 对关键断言可以做故障注入或 mutation：故意删除分支、反转条件或跳过配置，Case 必须能失败。这样验证的是测试有没有判别力，不只是有没有数量。

### 4.7 Coding Agent 会不会偷偷改测试

> 测试设计和编码是两个权限域。测试在编码前冻结并产生版本/hash，Coding Worker 只拿到实现目录写权限；测试变更必须由独立 Test Owner 提交并重新审核。最终 Eval 使用冻结版本和隐藏回归集，避免“修改实现的同时把标准改松”。

## 5. 失败后怎么归因：不能所有问题都回到 Coding

| 观测 | 归因 | 回路 |
| --- | --- | --- |
| 需求明确，但没有对应 Case | Case 覆盖缺口 | 回到测试设计 |
| 需求和 Case 都没描述该行为 | Spec / 领域规则缺口 | 回到需求澄清或规则候选 |
| Case 已覆盖，代码行为不满足 | Coding 错误 | 回到实现 |
| 共享组件单独输入能识别，入口扫描却漏掉 | Context Defect | 修依赖展开、组件发现和上下文装配，不改 Prompt |
| 正确文件和证据已进入上下文，仍然漏判或误判 | Policy Defect | 进入有编辑预算和验证门的鸿蒙规则训练 |
| 编译、设备、权限、网络或工具调用失败 | Execution Defect | 修环境或工具，不污染工程检索和规则 |
| `local_dev` 通过，真实 query 失败 | TCC、服务端或环境问题 | 进入配置/Trace 诊断 |
| 所有功能 Case 通过，但多改文件 | 计划或范围越界 | 阻断 MR，回到 repair plan |
| 重构后功能通过但结构膨胀 | 非功能约束缺口 | 复杂度、重复代码、Diff 预算门禁 |

一句话：

> 失败回流不是一个无脑 loop，而是一套路由；只有确认是实现问题才让 Coding Agent 重试。

## 6. 知识怎么组织，为什么不直接做向量数据库

### 6.1 四层知识结构

```text
第一层：公共 Workflow
  状态、权限、交付对象、验证主干

第二层：Domain Profile
  Harmony / Big Screen 等领域路由

第三层：原子 Skill / 版本化规则库
  规则判断、TCC、真机、服务端诊断

第四层：请求级证据
  目标 Bundle、代码片段、依赖链、LogID、Trace、测试结果
```

主 Skill 不携带所有领域知识，只描述公共主干和路由规则；Profile 只注入领域特有的 `ruleSkill`、可选 `configSkill` 和终端验收方式。知识与执行流程分离，规则可以独立版本化、评审和回归。

### 6.2 Skill 很大，如何保证加载正确模块

> 我不让模型靠语义相似度自由猜模块，而是先把路由条件结构化。`planningMode` 判断改动点是否明确：明确走 `explicit`，高层目标走 `rule_driven`；`domainProfile` 再由目标平台和任务类型确定。未命中已注册 Profile 就阻断，不允许模型凭通用经验补规则。相似模块有明确匹配条件、优先级和版本；加载结果必须回传规则 ID、证据 ID 和所消费的 Profile，路由因此可观测、可回放。

### 6.3 怎么知道这次加载到的资料就是你希望的

> “上下文里存在”不等于模型消费了。我要求每个结论绑定 `rule_id`、`evidence_id`、`file#line` 和引用链，并逐项返回 `modify / no_change / blocker`。先做反事实消融：固定模型、Skill 和 Case，只追加共享组件与调用关系；同一错误簇稳定出现 Retrieval Uplift 时优先优化检索和依赖范围，证据齐全仍判断错才进入鸿蒙规则训练。规则训练按 SkillOpt 式 textual learning rate 控制每步最多修改的原子条目数。Oracle Checklist 和 Evidence Recall 只用于归因，正式能力分数仍由修改范围与业务结果裁决。

### 6.4 为什么这里不用 RAG

> 这类核心知识不是海量非结构化文档，而是强结构、强版本、强权限的研发规则。平台、planningMode、Business、Bundle 和规则 ID 都是确定性键，Markdown/Profile 的渐进式加载比向量相似召回更可控、更便于 Code Review。RAG 会额外引入切分、召回、权限过滤和版本一致性问题，反而可能把关键规则召回漏掉。因此稳定规范走版本化 Skill 和规则库；只有历史事故、长尾排障经验这类规模大、表达不统一的资料，才进入带引用的 Hybrid RAG。

### 6.5 产品知识、会话状态和长期记忆为什么分开

| 类型 | 存储方式 | 原因 |
| --- | --- | --- |
| 稳定产品/领域规则 | Git 版本化规则库 | 可评审、可回滚、与代码版本一致 |
| 当前任务状态 | RFC、`demand_id`、分支、MR、Pipeline、状态机 | 必须精确，不适合相似检索 |
| 请求级证据 | 结构化结果和 artifact | 需要可追踪和复现 |
| 历史事故与长尾经验 | Hybrid RAG + citation | 量大、表达多样、按问题检索 |

因此“为什么不把用户历史对话也放进 RAG”的回答是：

> 用户历史属于当前 Session 的状态和偏好，不等同于稳定知识。当前任务需要精确恢复的字段进入 checkpoint/state store；只有跨任务长期复用、且能通过权限和来源过滤的历史事实才进入检索层。不能因为都叫上下文就全部塞进一个向量库。

## 7. 修改范围怎么找全，如何证明不是模型猜的

### 7.1 范围发现

当前搜索卡工作流先从目标 Bundle 出发，递归读取每层 `index.json.usingComponents`，解析：

- `./`、`../` 相对组件；
- `@components/*`、`@common/*`；
- 受支持 workspace package 的 `exports` 入口；
- 循环、重复和未解析引用。

输出不是一句“我认为这些文件相关”，而是：

```json
{
  "dir": "目标组件目录",
  "chain": ["CardEntry", "Container", "SharedComponent"],
  "ref": "@components/shared/index",
  "unresolved": []
}
```

`searchScope` 只证明目录可达，不直接等于规则命中。领域 Skill 必须继续用规则和代码证据形成结论。

这条机械链路刻意只承担“声明式组件图”责任，不把 JS/TS import、TTML include、样式 import 等所有语言依赖假装成已覆盖；这些非组件依赖由 Repo 对齐、目标调用点检查、CR 和回归测试继续补齐。职责窄但证据强，比一个无法说明覆盖边界的“全仓智能扫描”更可靠。

### 7.2 面试官：修改边界本身也是模型判断的，怎么证明它对

> 我把“发现候选范围”和“决定是否修改”拆开。候选范围由确定性组件图、需求声明和配置索引生成，模型不能随意缩小；模型只在候选范围内做语义判断。修改后再做反向审计：每个变更文件必须映射到 requirement/case/技术约束，每条需求也必须有变更或 `no_change` 证据。超出批准范围的 Diff 直接阻断，`unresolved` 不允许静默丢弃。

### 7.3 共享组件和跨 Business 怎么避免互相踩

> 交付先按 Business 拆成 deliveryUnit，每个单元有独立 Demand、分支、MR 和 Pipeline；共享文件提前指定唯一 Owner，其他单元只引用其结果，不跨分支重复修改。这样状态和权限不会在长流程中漂移，失败也能定位到具体交付单元。

### 7.4 静态范围仍找不到怎么办

> 运行时拼接路径、动态注册和服务端下发组件不能假装被静态分析完全覆盖。系统把它们保留为 `unresolved` 或低置信度项，触发逐级扩展、日志/Trace、运行时组件树或人工确认。边界显式化比“模型应该能理解”更可靠。

## 8. Skill 如何评测和调优：终局看范围与业务，失败再分轨归因

### 8.1 先把历史 MR 变成信息隔离的评测样本

代码改造没有唯一标准 Diff，历史人工 MR 也不天然等于干净 GT。我的样本构造分四步：

1. **固定起点**：冻结改造前 Repo 版本，所有执行从同一 baseline 开始。
2. **净化人工 MR**：让强模型读取历史 MR、代码仓库和改造背景，删掉顺手修 Bug、重构、格式化等与目标无关的变更，产出 `expected_file_set` 和关键业务目标。
3. **反向生成真实需求**：从历史结果生成“完成某项鸿蒙适配”这种业务输入，避免显式泄漏“改 A/B/C 文件、增加 D 分支”这类答案。
4. **隔离执行与裁判**：执行模型只能看到当前 Skill、用户需求和改造前 Repo；完成后，独立 Judge 才拿完整上下文对比 AI 结果与净化后的人工结果。

这套设计同时控制两种污染：人工 MR 里的无关改动不会被当成答案，执行 Agent 也不能从评测输入里直接抄修改清单。因为需求仍由历史结果反推，所以准确说法是“控制显式答案泄漏”，不是宣称绝对无泄漏。

### 8.2 面试官一定会问：具体评测指标是什么

设 `E` 为净化后的预期修改文件集，`A` 为 AI 实际修改文件集，`n(S)` 为集合元素个数。文件级不比较代码行是否完全相同，而看范围是否收敛：

| 指标 | 公式 | 回答的问题 |
| --- | --- | --- |
| 改造范围准确率（评测文档称“改造范围覆盖率”） | `n(A∩E) / n(A)` | AI 改过的文件中，有多少确实应该改 |
| 相关文件召回率 | `n(A∩E) / n(E)` | 应该改的文件中，AI 找到了多少 |
| 改动噪音率 | `n(A−E) / n(A)`，并同时报告额外文件数 | 有没有大量无关修改；它与范围准确率互补 |
| 漏改率 | `n(E−A) / n(E)` | 有没有少改、漏改关键模块 |

空集合口径提前固定：`E≠∅, A=∅` 时范围准确率和召回率都记 0；`E=∅, A=∅` 是正确 `no_change`；`E=∅, A≠∅` 时噪音率记 100% 并判范围失败，避免分母为空时临场改口径。

文件对上只是必要条件，第二维是 **业务结果一致性**：

> 飞书方案直接定义的是“文件范围 + 业务结果一致性”双维框架，以及范围准确率、相关文件召回率和噪音控制；漏改率、下面的权重/状态值和交付硬门禁，是我针对鸿蒙代码改造做的工程化落地，不冒充文档原始公式。

1. Judge 先把需求拆成带权重的关键业务目标和硬约束，例如核心配置、关键模块路由、新场景可运行、兼容性和禁止副作用。
2. 每个目标按 `完成 / 部分完成 / 未完成` 计 `1 / 0.5 / 0`，得到 `Goal Completion = Σ(wᵢ×statusᵢ) / Σwᵢ`。
3. 再对少改、多改、额外业务语义变化和兼容性回归扣分，形成 `0–100` 的 Business Consistency Score。
4. 任一关键目标未完成、硬约束失败、编译/回归失败，都直接判 FAIL；不能靠其他项平均分高把关键失败盖过去。

最终采用双门禁，不把两类指标混成一个漂亮但不可解释的平均数：

```text
PASS
= 文件范围门禁通过
∧ 关键业务目标与硬约束全部通过
∧ 编译、Case、历史回归和真实环境证据通过
```

其中范围门禁不是机械要求 AI 与人工文件集完全相同。净化阶段由强模型提出关键文件子集 `E_critical ⊆ E`，经人工复核后冻结；`E_critical` 召回必须为 100%。`A−E` 中每个额外文件都必须反向映射到 requirement 或技术约束，并由业务 Judge 确认属于等价实现，否则计为噪音并判失败。这样既容纳不同正确实现，又不放过无依据的多改。

如果追问“强模型 Judge 自己会不会漂”：Judge 不自由写结论，而是固定模型版本、Prompt、推理参数和 JSON rubric，逐个返回 `goal_id / status / evidence / missing / extra_semantics`；先用人工标注的 PASS/FAIL 锚点校准，阈值附近或两次裁决冲突的样本再人工仲裁。编译、测试、文件集合和真机证据仍由确定性工具计算，不能被语义分覆盖。

评测方案文档里的三个判分示例用于说明不同失败如何进入语义判定，它们不是鸿蒙 Skill 的总体成功率：

| Case | 结果 | 业务语义分 | 判定信号 |
| --- | --- | ---: | --- |
| 样例 A | PASS | 93/100 | 主链完成度高，方向高度一致 |
| 样例 B | FAIL | 78/100 | 主方向正确，但范围偏移并引入额外业务语义 |
| 样例 C | FAIL | 81/100 | 核心能力基本完整，但兼容性控制不稳定 |

标准回答：

> 我的 Skill 主评测不是“单测通过率”，而是两个终局维度。第一是文件级范围：准确率、召回率、噪音率和漏改率；第二是业务结果一致性：把需求拆成关键目标和硬约束，计算 0 到 100 的语义分。关键目标、编译回归或真实环境任一硬门禁失败都判 FAIL。这样既允许 AI 和人工使用不同实现方式，又能识别“文件都碰到了但业务没完成”以及“功能做出来但偷偷多改”的情况。

### 8.3 Skill 调优先做失败分型，不是看到失败就改 Prompt

```text
失败样本
├─ Context Defect：共享组件、依赖或调用链没有进入上下文
├─ Policy Defect：证据齐全，但鸿蒙规则缺失、冲突、歧义或动作错误
└─ Execution Defect：编译环境、设备、权限、网络或工具执行失败
```

三类问题分别改工程检索、规则 Skill 和执行环境。正式终局指标仍是上一节的范围与业务一致性；下面的 Evidence Recall、定位率等只是诊断哪一层该优化。

#### A 轨：工程侧调优——共享组件反事实实验

> 我发现有些入口文件扫描不出问题，但把它依赖的共享组件单独喂给同一个 AI，模型就能判断正确。于是我固定模型、Skill、需求和推理参数，只做一个变量：A 组使用原上下文，B 组额外加入同版本共享组件、定义位置和调用关系。如果同一错误簇在多次回放中稳定出现 Retrieval Uplift，这是 Context Defect 的强归因信号；我优先改上下文供给，再用完整门禁复核，而不凭单个 Case 就断言规则一定没问题。

工程侧因此优化：依赖递归展开、共享组件发现、版本匹配、加载顺序、最小 Evidence Pack 和 `unresolved` 显式上报。用以下诊断指标看改动是否有效：

- `Evidence Recall@K`：人工标注的关键共享组件证据是否进入前 K 条；
- `Evidence Validity`：证据是否真实存在、版本匹配、依赖可达；
- `First-pass Localization Rate`：第一轮是否定位到正确组件、文件和符号；
- `Retrieval Uplift`：只增加共享组件证据后，范围门禁和业务 PASS 的提升；
- `Context Cost`：上下文文件数、token、工具调用和定位轮数，避免靠塞全仓换召回。

#### B 轨：鸿蒙规则/Prompt 调优——把 Skill 当成可训练文本参数

规则不写成长篇散文，而原子化为：

```text
rule_id
+ trigger / 适用范围
+ evidence / 判断证据
+ required_action / forbidden_action
+ exception
+ verifier
```

我参考 Microsoft 2026 年的 [SkillOpt](https://arxiv.org/html/2605.23904v2)，把调规则从“凭感觉重写 Prompt”改成训练式优化：

| 模型训练概念 | 鸿蒙规则 Skill 中的对应物 |
| --- | --- |
| Parameter | 带 Rule ID 的原子鸿蒙适配规则 |
| Forward | 冻结目标模型、Harness 和 Judge，在一批历史 Case 上跑完整轨迹 |
| Gradient 信号 | 成功/失败轨迹分组后的共性反思，不针对单例背答案 |
| Textual learning rate `Lₜ` | 每个 optimization step 最多允许应用的原子规则修改数 |
| Validation | 独立 selection 集上的范围与业务双维门禁 |
| Negative feedback | 被拒 edit 及其掉分进入当前 epoch 的 rejected-edit buffer |
| Slow/meta update | epoch 边界抽取同一批训练样本，分别运行前一 epoch 与当前 epoch 的 Skill，比较改善、回归、持续失败和稳定成功 |
| Checkpoint | `current_skill`、`best_skill`；部署只发布历史最优版本 |

一次训练闭环是：

1. 固定目标模型、执行 Harness、Judge、代码 baseline 和数据切分。
2. 在 train split 跑 rollout，把成功和失败轨迹分开做 reflection minibatch，提炼可泛化错误簇。
3. 候选只允许原子 `add / delete / replace`。`Lₜ` 前期每 step 最多 4 条，后期按 schedule 衰减到 2 条；这里限制的是 **每个 step 的上限**，不是每个 epoch 为了凑数必须改 N 条。
4. Candidate 在独立 selection split 上回放。Business Consistency 必须严格高于 current，关键目标、范围准确率/召回率和历史回归不得退化；持平也拒绝。
5. 通过才晋级为 current 并更新 best；失败 edit 进入 rejected buffer，避免同一 epoch 重复犯错。
6. epoch 边界抽取同一批训练样本，用前后两个 epoch 的 Skill 各跑一次并做 slow/meta update；test split 不参与选规则，只在 release checkpoint 做最终报告。

这套训练协议解决三个追问：小步有界更新便于归因；独立验证门防止过拟合；rejected buffer、epoch-wise slow/meta update 和 best checkpoint 防止越改越长、灾难性遗忘和坏版本累积。论文实验默认配置是 4 个 epoch、rollout batch 40、reflection minibatch 8、`Lₜ=4` 并余弦衰减到最低 2；迁移到鸿蒙规则时沿用的是这套控制机制，batch 大小按历史任务量设置。鸿蒙场景另外保留一套不参与调参的 frozen regression set，这是我的工程加固，不把它说成 SkillOpt 原生的固定 anchor。

时间线最稳的口径是：**我原有实践已经是“小步修改 + 固定回放”，SkillOpt 发布后，我用 textual learning rate、严格验证门、拒绝缓冲区和 best checkpoint 把这套方法正式标准化。** 不要说成论文发表前就照着 SkillOpt 实现。

### 8.4 已有业务数据与评测样例

| 数据 | 能证明什么 | 不能偷换成什么 |
| --- | --- | --- |
| 日常 117 张 + 高考 50 处，覆盖 71.2% 高 PV 场景 | 真实业务交付规模 | Agent 自动化成功率 |
| 文档判分示例 93 PASS、78 FAIL、81 FAIL | 说明范围漂移和兼容性失败如何进入语义判定 | 鸿蒙 Skill 的总体成功率或 Judge 有效性证明 |
| 约 2,000 张存量卡定位 24 处遗漏，约 3 小时含复核 | 批量配置治理价值 | 全链路 Agent 成功率 |
| 12 张卡约 1 小时，QA 前完成真机出卡验证 | 真实业务闭环跑通 | 普遍四倍提效结论 |

### 8.5 第二、第三个需求怎么证明稳定

> 数据按需求族、组件族和时间切成 train、selection 和 frozen test，近重复需求不能跨集合。train 用于错误聚类和提出 edit，selection 决定 Candidate 是否严格晋级，frozen test 只在发布检查点使用。对非确定节点固定模型、Skill、Harness 和参数重复运行，同时报告范围 P/R、Business Consistency、最差 Slice、回归率、pass@1、方差和成本。新需求先进入 Shadow 集，既要通过自己的业务目标，也要带着鸿蒙场景额外维护的 frozen regression set 一起通过。

如果面试官拿大屏和时间格式继续问：

> 两个场景分别构成不同 Slice：大屏重点看返回值契约，时间格式重点看 AB 接入和调用点覆盖。任何一轮规则更新即使总体平均分上涨，只要关键 Slice 或 frozen regression set 回归，就不允许晋级；因此稳定性不是“多跑几次都结束了”，而是同一版本在独立任务族上持续满足范围和业务双门禁。

## 9. 图片中那组高压追问：可直接背的回答

### Q1：这些知识你是怎么组织的？

> 公共 Workflow 只保留状态、权限和交付主干；平台差异放在 Domain Profile；规则、TCC、真机和服务端诊断拆成原子 Skill；每次请求只加载命中的 Profile 和最小代码证据。稳定知识版本化，任务状态结构化，历史事故才做检索。

### Q2：Skill 很大，两个模块相似时怎么保证选对？

> 先用结构化元数据路由，不靠语义相似度猜。`explicit/rule_driven` 决定 repair plan 的产生方式，目标平台决定 Profile，Business 决定交付单元。未命中注册项就阻断；命中后必须返回所用 Profile、规则 ID 和证据 ID，路由可以回放。

### Q3：怎么知道 Agent 真加载了你希望的资料？

> 不看“上下文里有没有”，看它是否消费。每个结论必须引用 `evidence_id`、规则 ID、文件位置和依赖链。失败时固定模型、Skill 和 Case，只追加共享组件及调用关系做 A/B；同一错误簇多次稳定翻转是 Context Defect 的强信号，优先改组件发现和加载，证据齐全仍误判才进入 Policy Defect 调优。Evidence Recall 只用于这一步归因，终局仍由范围与业务双门禁裁决。

### Q4：第一个需求跑通，第二、第三个怎么办？

> 第一个需求只进入 train，不直接变成成功率。样本按需求族、组件族和时间隔离成 train、selection、frozen test；规则 Candidate 只有在 selection 上严格提升且鸿蒙 frozen regression set 零关键回归才晋级，test 只在 release checkpoint 使用。新需求还要进入 Shadow 集，重复运行并分别报告范围 P/R、业务一致性、最差 Slice、回归率、方差和成本。

### Q5：你的输入是需求、输出是 MR，怎么判断 MR 正确？

> 需求先变成验收合同。每条 requirement 映射 Case、计划项、变更文件和最终证据；每个 Diff 也必须反查到 requirement 或技术约束。MR 只有在冻结 Case、历史回归、范围审计、本地产物和真实环境门禁全部通过后才进入可交付状态。

### Q6：这些只能证明流程执行完，不能证明代码改对。

> 同意“流程完成不等于正确”，所以我的状态机不是用“工具调用成功”推进，而是用外部证据推进。比如 `query_search` 证明真实配卡生效，却不能证明当前本地产物；`local_dev` 证明本次构建，却不能替代真实环境。所有任务都要有代码与当前产物证据；涉及 TCC 或要求检查真实出卡时，再叠加 `query_search`，相关证据不能互相替代，缺少就不能进入对应交付状态。

### Q7：如何确定影响范围找全了？

> 候选范围先由确定性组件图、需求清单和配置索引生成，保留完整 chain 与 unresolved；模型只在候选内判断。修改后再做双向追踪和全量回归。范围无法静态确定时进入运行时组件树、LogID Trace 或人工门禁，不用模型置信度掩盖未知。

### Q8：影响范围本身也是 AI 判断的，怎么证明 AI 判断对？

> 我没有把边界完全交给 AI。结构关系先由脚本、Resolver、需求声明和配置索引形成候选，模型只做语义判断；最终拿 AI 文件集 `A` 对比净化后的预期集 `E`，明确计算 `n(A∩E)/n(A)`、`n(A∩E)/n(E)`、噪音率和漏改率。文件范围过门后还必须通过关键业务目标和硬约束，所以“范围找对”也不会被偷换成“代码改对”。

### Q9：测试全通过，但需求其实没完成怎么办？

> 这属于 Case 覆盖问题，不是 Coding 问题。通过 requirement-to-case 矩阵检查每个条款，通过负向和不变量 Case 防止 ABC 变 AB 或 ABCD；QA 在真机前先对比 AI Case 与测试计划；新的漏项回灌为候选 Case。编码 Agent 无权通过补丁修改验收标准。

### Q10：为什么不把用户历史对话也放到 RAG？

> 稳定产品知识、当前 Session 状态和历史检索资料的语义不同。产品规则要版本化，状态要精确 checkpoint，历史事故才适合 RAG。用户最近对话优先作为 Session 状态和原文；只有跨会话、可授权、可溯源的长期事实才进入检索层。

### Q11：为什么不用一个大 Agent 全做完？

> 规划、编码、验收和权限控制存在职责冲突。一个 Agent 同时生成代码和修改测试标准，会形成自证；同时掌握配置写入和部署权限，错误半径也太大。我采用确定性外层 Workflow 和职责隔离的 Worker，简单任务仍走短路径，避免为了多 Agent 增加 token 和调试成本。

### Q12：为什么不用 LangGraph 或工作流引擎？

> 选型先看核心问题。这里团队已经有 Spec-to-Code、CLI 和原子 Skill，主要缺口是领域路由、结果交接和验收编排；用 Skill 主干复用成本更低。状态图扩大到需要持久 run、复杂补偿和大规模并发时，再把同一状态契约迁到成熟引擎，而不是一开始就引入框架复杂度。

### Q13：长流程怎么防止状态漂移？

> 不依赖聊天记忆传状态。每个 deliveryUnit 固定 `demand_id/branch/mr/pipeline`，阶段输出结构化落盘；状态转移有 guard 和 evidence；共享文件唯一 Owner；聚合只读各单元最终结果。对话可以丢，状态和证据不能丢。

### Q14：如何避免无限循环和越改越臃肿？

> 首先失败分类，不是所有失败都回 Code；其次限制重试预算和 Diff 预算。规则侧再用 textual learning rate 约束每 step 的原子 edit 数，Candidate 未在独立 selection 上严格提升就不替换 current，被拒 edit 进入负反馈，最终只发布 best checkpoint。这样不是先把坏 Prompt 覆盖上线再回滚，而是坏候选根本不能晋级。

### Q15：QA 的价值是不是被 Agent 替代了？

> 没有。AI 和 RD 把低风险、可重复的检查前移，并把已覆盖项、未覆盖项和高风险项整理成证据；QA 重点验证测试设计完整性、设备交互、视觉表现和复杂环境。目标是减少盲测与重复定位，不是转移质量责任。

### Q16：这个项目为什么算全栈，不只是前端自动化？

> 因为交付对象跨越客户端代码、配置控制面、搜索服务数据链路、研发平台对象和真实终端。我的责任不是把所有服务端代码都归为自己写，而是能沿端到端链路定义接口、证据和责任边界，并让对应能力在同一交付合同下协作。

### Q17：如何迁移到合合信息的大客户 Agent 项目？

> 把 Harmony Profile 换成客户领域 Profile，公共主干不变：需求合同、领域规则、API/数据源工具、权限、状态、评测和交付证据仍然复用。客户差异进入 Profile 和工具适配层，不复制整套 Agent；敏感操作按租户、角色和环境设置门禁，所有结论保留来源和审计链。

### Q18：你这个 Skill 最具体的评测指标是什么？

> 两个终局维度。文件侧设净化后预期集合 `E` 和 AI 实际集合 `A`，算范围准确率 `n(A∩E)/n(A)`、相关文件召回率 `n(A∩E)/n(E)`、噪音率和漏改率；业务侧由独立 Judge 把需求拆成关键目标与硬约束，算 0–100 的 Business Consistency。关键目标、编译回归、当前产物或真实环境任一硬门禁失败就判 FAIL，不让平均分掩盖关键错误。评测方案文档中的判分示例是 93 PASS、78 FAIL、81 FAIL，用来说明范围漂移和兼容性问题如何反映到语义判定，不把它们说成鸿蒙 Skill 的总体结果。

### Q19：Skill 具体怎么调，为什么不是手工堆 Prompt？

> 先分轨。固定其他变量后，只追加共享组件证据；如果同类 Case 多次回放都稳定提升，就优先归为工程上下文问题，改组件发现、依赖展开和加载顺序；证据齐全仍错，才调鸿蒙规则。规则侧参考 SkillOpt：冻结目标模型、Harness 和 Judge，把带 Rule ID 的规则当文本参数；训练轨迹分成功/失败 minibatch 反思，每个 optimization step 最多 4 个原子 add/delete/replace，后期衰减到 2 个。Candidate 在独立 selection 上必须严格提升，持平也拒绝；拒绝项进入负反馈，epoch 边界用同一批训练样本比较前后 Skill 并形成 slow update，最终只发布 `best_skill`。

## 10. 口径替换：这些话不要说

| 容易被击穿的说法 | 推荐替换 |
| --- | --- |
| 改完以后交给测试兜底 | 测试设计前置，执行嵌入编码循环，真实环境验收在 QA 前 |
| AI 判断代码改对了 | AI 产出候选，冻结验收合同和独立证据决定交付 |
| Workflow 跑完了 | 状态只有在 guard 对应证据齐全后才推进 |
| Skill 成功率 90% | 报文件范围 P/R/噪音 + Business Consistency + 硬门禁 PASS，分母是历史任务 |
| AI 找全了影响范围 | 确定性范围 + unresolved + 语义判断 + 双向追踪 |
| 全自动端到端 | 自动执行低风险可验证步骤，高风险写入、Push、部署保留授权 |
| 知识都放进 RAG | 规则版本化、状态结构化、历史长尾才检索 |
| QA 不需要再测 | QA 从重复基础检查转向覆盖审查和高风险真实环境验证 |
| Agent 做的是 TDD | 验收测试驱动：可自动化 Case 做 Red/Green，真机计划编码前冻结、编码后执行 |
| 每个 epoch 必须改固定条数 | 每个 optimization step 有最多 `Lₜ` 条的编辑预算；无证据时不为凑数修改 |
| 调优失败就回滚线上 Skill | Candidate 未过 validation gate 就不能晋级，current/best checkpoint 保持不变 |

## 11. 指标口径卡：主评测、诊断指标和业务规模不要混

| 口径 | 计算 / 数据 | 使用场景 |
| --- | --- | --- |
| 文件范围准确率 | `n(A∩E) / n(A)` | 判断 AI 是否多改 |
| 相关文件召回率 | `n(A∩E) / n(E)` | 判断 AI 是否漏改 |
| 改动噪音率 | `n(A−E) / n(A)` + 额外文件数 | 报告无关改动，不能只报准确率 |
| 业务结果一致性 | 关键目标加权完成度、硬约束、额外语义变化，0–100 | 判断实现不同但业务结果是否等价 |
| 最终 PASS | 范围门禁 ∧ 业务硬门禁 ∧ 编译/回归 ∧ 当前产物/真实环境 | 代码改造 Skill 的终局结论 |
| 文档判分示例 | 93 PASS；78 FAIL；81 FAIL | 说明不同失败如何进入语义判定；不是鸿蒙总体结果 |
| Context 诊断 | Evidence Recall@K、Validity、定位率、Retrieval Uplift、Context Cost | 只判断是否该改工程检索，不是 MR 正确率 |
| 真实业务交付 | 日常 117 张 + 高考 50 处；71.2% 高 PV | 开场证明规模与业务价值 |
| 配置存量治理 | 约 2,000 张、24 处、约 3 小时含人工复核 | 证明配置 Skill 的批量价值 |
| 服务端排障 | 5 个链路问题独立定位闭环 | 证明全链路诊断能力 |
| 真实交付试跑 | 12 张约 1 小时、QA 前真机验收 | 证明 Workflow 能闭环真实任务 |

三个防偷换提醒：

- 范围准确率不是整体 Accuracy；它只回答“改过的文件有多少该改”。
- Business Consistency 不能替代硬门禁；关键目标失败时不能靠平均分补回来。
- Evidence Recall 和 Retrieval Uplift 是工程归因指标，不能冒充最终代码正确率。

## 12. 最后的反问与收尾

### 12.1 项目收尾 20 秒

> 这段经历让我形成了一个很稳定的判断：AI Coding 最难的不是让模型写出代码，而是把需求、范围、权限和验收变成模型不能随意改写的工程约束。我的优势是既能做前端和全栈实现，也愿意把 Agent 的状态、评测、失败恢复和真实交付证据补完整。这和岗位希望沉淀 AI Native 研发范式、参与 Agent 大客户交付的方向非常一致。

### 12.2 可以反问面试官

1. 这个岗位的大客户 Agent 项目，目前最主要的难点是模型效果、领域知识接入，还是现有系统/API 的交付集成？
2. 团队如何定义 AI Native 研发的有效性：代码采纳率、交付周期、缺陷率，还是端到端任务成功率？
3. Agent 在客户环境中可以自动执行到什么权限边界，哪些动作必须人工审批？
4. 团队现在的评测更偏离线 Benchmark、历史任务回放，还是生产 Shadow/灰度？
5. 新同学进入后，更可能先负责某个全栈业务模块，还是先参与 Agent 平台或领域 Workflow 建设？

## 13. 证据索引

- 交付规模和四层链路：[20min-defense-speech.md](/Users/bytedance/Downloads/20min-defense-speech.md:9)
- 历史规则扫描、消融和 Bad Case 分层原始证据：[20min-defense-speech.md](/Users/bytedance/Downloads/20min-defense-speech.md:27)
- 代码改造 Skill 的样本净化、范围指标和业务一致性评测：[代码改造场景的 Skill 评测思路](https://bytedance.sg.larkoffice.com/docx/L7dldJbIyou1bYxnkSql8OlVguf)
- 训练式规则优化、textual learning rate、validation gate 和 rejected-edit buffer：[SkillOpt 论文](https://arxiv.org/html/2605.23904v2)｜[Microsoft Research 介绍](https://www.microsoft.com/en-us/research/blog/skillopt-agent-skills-as-trainable-parameters/)｜[官方实现](https://github.com/microsoft/SkillOpt)
- TCC 和服务端联合排障：[20min-defense-speech.md](/Users/bytedance/Downloads/20min-defense-speech.md:59)
- 端到端 SOP 与 12 张真实交付：[20min-defense-speech.md](/Users/bytedance/Downloads/20min-defense-speech.md:79)
- AI 使用方法论：[20min-defense-speech.md](/Users/bytedance/Downloads/20min-defense-speech.md:155)
- TDD 式 Case、追踪和失败归因：[final-review-growth-future-speech.md](/Users/bytedance/Downloads/final-review-growth-future-speech.md:37)
- 主 Workflow 的 planningMode、Profile、deliveryUnit 和验证门禁：[SKILL.md](/Users/bytedance/LinaMono/lina-mono/lina-mono/.agents/skills/lina-search-card-workflow-b-profile-v2/SKILL.md:8)
- Harmony Profile 的规则、配置和终端验收边界：[harmony-profile.md](/Users/bytedance/LinaMono/lina-mono/lina-mono/.agents/skills/lina-search-card-workflow-b-profile-v2/references/harmony-profile.md:1)
- 协议化状态与七道闸口：[workflow-spec-to-code/SKILL.md](/Users/bytedance/LinaMono/experiment_e2e/ai-infra/skills/workflow-spec-to-code/SKILL.md:51)
- 任务文件边界与验证合同：[task-breakdown-rules.md](/Users/bytedance/LinaMono/experiment_e2e/ai-infra/skills/workflow-spec-to-code/references/task-breakdown-rules.md:13)
- Code / Eval 职责隔离：[code-subagent-prompt.md](/Users/bytedance/LinaMono/experiment_e2e/ai-infra/skills/workflow-spec-to-code/templates/code-subagent-prompt.md:16)｜[eval-subagent-prompt.md](/Users/bytedance/LinaMono/experiment_e2e/ai-infra/skills/workflow-spec-to-code/templates/eval-subagent-prompt.md:13)
- 本地产物与真实 query 的不同证据合同：[harmony-card-smoke/SKILL.md](/Users/bytedance/LinaMono/lina-mono/lina-mono/.agents/skills/harmony-card-smoke/SKILL.md:19)
- 组件范围展开的递归、去重与 provenance：[expand_dependencies.js](/Users/bytedance/LinaMono/lina-mono/lina-mono/.agents/skills/lina-search-card-workflow-b-profile-v2/scripts/expand_dependencies.js:214)
- 通用 AI Coding 成长主线：[19-AI时代Coding成长历程.md](/Users/bytedance/Documents/interview_reference/19-AI时代Coding成长历程.md:3)
- 全栈 AI Agent 项目补充：[13-AI-Agent全栈项目面试稿.md](/Users/bytedance/Documents/interview_reference/13-AI-Agent全栈项目面试稿.md:3)
