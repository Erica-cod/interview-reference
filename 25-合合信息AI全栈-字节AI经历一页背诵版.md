# 合合信息 AI 全栈：字节 AI 经历一页背诵版

完整版：[24-合合信息AI全栈-字节AI-Native经历高压追问.md](/Users/bytedance/Documents/interview_reference/24-合合信息AI全栈-字节AI-Native经历高压追问.md:1)

## 一句话定位

> 我做的不是一个会写代码的 Agent，而是一套面向搜索卡片交付的 AI Native 研发控制面：人定义目标和验收合同，确定性工具控制范围与状态，AI 做语义判断和候选实现，独立测试、配置证据和真实终端结果决定能不能交付。

## 90 秒项目介绍

> 我在字节独立负责抖音搜索卡片的鸿蒙适配，支撑日常 117 张卡和高考专项 50 处交付，覆盖 71.2% 的高 PV 场景。这个任务不是只改前端：服务端要有数据，TCC 要满足鸿蒙出卡条件，代码要兼容，最后真机真实 query 下要正常出卡。
>
> 第一阶段，我把高频适配经验沉淀成规则 Skill，并用真实历史 MR 构造信息隔离、避免显式答案泄漏的回放：净化掉顺手重构和无关修复，反向生成只描述业务目标的需求，再固定改造前 Repo 执行。最终不逐行比 Diff，而是同时看文件范围准确率、召回率、噪音率，以及关键业务目标的一致性分和硬门禁。失败后再区分工程上下文和鸿蒙规则，不把所有问题都归因于 Prompt。
>
> 第二阶段，我把完成标准从“代码改完”扩展到“真机出卡”。进测前主动检查 TCC；不出卡时用同一 LogID 联合定位配置和服务端最早断点。这个能力扫描约 2,000 张存量卡，定位并处理 24 处遗漏，约 3 小时包含人工复核。
>
> 第三阶段，我把代码扫描、Spec/Coding、配置诊断和真机验收编排成公共主干：明确需求走 explicit，高层适配走 rule-driven；按 Business 拆 Demand、分支、MR 和 Pipeline；最后分别用 local_dev 证明当前产物、用 query_search 证明真实配置和出卡。核心不是让 AI 做得更多，而是让每一步都有可追踪的外部证据。

## 最关键追问：怎么证明 Agent 改对了

> 我不让生成代码的 Agent 同时担任裁判。需求先变成验收合同：每条 requirement 映射 Case、计划项、变更文件和证据；编码前确认覆盖并冻结，Coding Agent 只能在文件和权限边界内把同一批 Case 从 Red 推到 Green，不能修改验收标准。跑绿后还要做历史回归、Diff 范围审计、本地产物验证；涉及 TCC 或真实出卡时再做 query_search 和真机验收。Agent 产出候选，独立 oracle 决定交付。

## 回应“你只是交给测试兜底”

> 不是。测试设计在编码前，测试执行在编码循环里，真实环境验收在 QA 前。我只是把昂贵、不可重复的模型主观判断，替换成外部化、可执行、可回归的工程证据。

## 必问：Skill 的具体评测指标

> 设净化后的预期文件集为 `E`、AI 实际修改集为 `A`。文件侧算范围准确率 `n(A∩E)/n(A)`、相关文件召回率 `n(A∩E)/n(E)`、噪音率 `n(A−E)/n(A)` 和漏改率；业务侧由独立 Judge 把需求拆成关键目标与硬约束，算 0–100 的 Business Consistency。关键目标、编译回归、当前产物或真实环境任一门禁失败就判 FAIL，不能靠平均分掩盖。评测方案文档中的判分示例是 93 PASS、78 FAIL、81 FAIL，用来解释范围偏移和兼容性失败如何反映到语义判定，不是鸿蒙 Skill 的总体结果。

## 必问：Skill 怎么调优

> 先分两轨。工程侧固定模型、Skill 和 Case，只把共享组件及调用关系额外喂给 AI；如果同类 Case 多次回放都稳定提升，这是 Context Defect 的强信号，就优先改组件发现、依赖展开和上下文装配。证据齐全仍漏判或误判，才调鸿蒙规则。
>
> 规则侧是在原有“小步修改 + 固定回放”上，用 Microsoft 2026 年 SkillOpt 正式标准化：把带 Rule ID 的原子规则当文本参数，冻结目标模型、Harness 和 Judge；成功/失败轨迹分 minibatch 反思，每个 optimization step 最多做 4 个 `add/delete/replace`，后期衰减到 2 个。Candidate 必须在独立 selection 集上严格提升，持平也拒绝；拒绝 edit 进入负反馈，epoch 边界抽取同一批训练样本分别运行前后 Skill，形成 slow update，最终只发布 `best_skill`。准确说是“每 step 的修改上限”，不是每个 epoch 为凑数必须改 N 条。

## 正确性的六层合同

```text
L0 需求覆盖：requirement → case
L1 范围覆盖：searchScope / chain / unresolved
L2 代码行为：单测 / 契约 / Lint / 历史回归
L3 集成链路：TCC / Trace / 下游响应
L4 当前产物：local_dev
L5 用户结果：query_search / 真机 / 截图 / 组件树
```

## 十二个追问钉子

1. **知识怎么组织？** 公共 Workflow、Domain Profile、原子 Skill/规则库、请求级证据四层。
2. **Skill 大了怎么选对？** `explicit/rule_driven` + platform Profile + Business，未命中就 blocker，不让模型猜。
3. **怎么知道资料真被消费？** 结论绑定 `rule_id/evidence_id/file#line/chain`；共享组件单独输入做反事实 A/B。
4. **第一个需求成功，后面呢？** 按需求族/组件族/时间切 train、selection、frozen test；test 只做最终报告。
5. **流程完成不等于正确。** 状态由 guard 的外部证据推进，不由工具调用成功推进。
6. **影响范围怎么找？** 确定性组件图先找候选，模型只做语义判断；`unresolved` 不能静默丢。
7. **全绿但需求漏了？** 这是 Case/Spec 缺口，回测试设计，不回 Coding 盲目打补丁。
8. **Coding 会不会改测试？** 测试冻结并独立 Owner；Coding 只写实现；关键任务有隐藏回归。
9. **为什么不用 RAG 存规则？** 规则要版本化、评审和精确命中；RAG 只用于历史事故和长尾经验。
10. **长流程怎么防漂移？** 一个 Business 一个 deliveryUnit，固定 Demand/Branch/MR/Pipeline，结构化 artifact 传递状态。
11. **具体评测什么？** 文件范围 P/R/噪音 + Business Consistency + 编译、回归、产物和真实环境硬门禁。
12. **规则怎么调？** `Lₜ` 限制每 step 原子 edit，selection 严格晋级，rejected buffer 记失败，部署只取 best checkpoint。

## 指标卡

- 真实交付：117 张日常卡 + 高考 50 处，覆盖 71.2% 高 PV。
- 文件范围：准确率 `n(A∩E)/n(A)`、召回率 `n(A∩E)/n(E)`、噪音率、漏改率。
- 业务结果：关键目标加权完成度 + 硬约束 + 额外语义变化，输出 0–100 和 PASS/FAIL。
- 文档判分示例：93 PASS；78 FAIL；81 FAIL；不是鸿蒙总体成功率。
- 工程诊断：Evidence Recall@K、Validity、首次定位率、Retrieval Uplift、Context Cost。
- 配置治理：约 2,000 张，24 处遗漏，约 3 小时含人工复核。
- 服务端排障：5 个链路问题独立定位闭环。
- 真实 Workflow：12 张约 1 小时，QA 前完成真机验收。

不要把 Evidence Recall 说成 MR 正确率，不要用 Business 平均分掩盖关键失败，也不要把一次 12 张交付说成普遍四倍提效。

## 岗位匹配收尾

> 这段经历和岗位最匹配的地方，是我已经把 AI 从个人编程工具推进到真实研发交付：能做需求和架构拆分，也能处理代码、配置、服务链路、测试、权限和最终用户结果。进入大客户 Agent 场景后，我会保留公共主干，把客户差异放进领域 Profile 和工具层，用同一套状态、评测和证据合同控制交付质量。

## 口径替换

```text
“测试兜底”       → “验收合同前置，测试嵌入编码循环”
“AI 判断改对了”  → “AI 产出候选，独立 oracle 裁决”
“Workflow 跑通”  → “guard 对应证据齐全”
“AI 找全范围”    → “确定性范围 + unresolved + 回归”
“成功率 90%”     → “范围 P/R/噪音 + 业务一致性 + 硬门禁 PASS”
“每 epoch 改 N 条”→ “每 optimization step 最多 Lₜ 条；不为凑数修改”
```

依据：[代码改造场景的 Skill 评测思路](https://bytedance.sg.larkoffice.com/docx/L7dldJbIyou1bYxnkSql8OlVguf)｜[SkillOpt 论文](https://arxiv.org/html/2605.23904v2)｜[Microsoft Research 介绍](https://www.microsoft.com/en-us/research/blog/skillopt-agent-skills-as-trainable-parameters/)
