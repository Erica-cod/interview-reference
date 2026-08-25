# Text2SQL量化评测与SQL安全校验

## 面试口述总稿（优先复习）

下面几段集中放置所有可直接用于面试的说法。面试时不需要一次全部讲完，先回答当前问题，再根据面试官追问展开。

### 1. 量化评测、校验与Bad Case闭环

> 这个项目的量化评测主要基于约600条人工标注Query，覆盖约80到100张业务表和1000到2000个字段，其中30%到40%是复杂查询。Schema检索采用字段级标注，Recall等于正确召回字段数除以Gold字段总数，Precision等于正确召回字段数除以返回候选字段总数。整套BM25、Embedding、RRF和Rerank两阶段检索，将字段Recall从约72%提升到93%，Precision从约18%提升到43%。
>
> SQL侧不比较字符串，因为两条写法不同的SQL可能得到相同的正确结果。我们在固定数据库快照上分别执行人工Gold SQL和模型SQL，通过规范化后的结果集是否一致计算Execution Accuracy。单库场景约为81%，加入校验回溯后提升到88.5%，相当于600条Query中额外修正约45条；多库场景从67%提升到70%，约额外修正18条。
>
> 每个LangGraph节点还会保存结构化日志，用于定位Bad Case。例如路由错误就补充意图样本，Schema漏召回就补同义词和字段描述，噪声过多就调整Rerank阈值，Join错误就完善Schema关系，业务口径不明确就触发人工澄清，SQL错误则结合SQLGlot检查结果和数据库报错进行修正。这样优化不是凭感觉改Prompt，而是根据错误发生在哪一个节点进行数据驱动迭代。

口径边界：上述指标是V1内部项目的近似实测结果；当前使用LangGraph、Skill、Human-in-the-loop和Mock数据的V2 Demo主要验证工程链路，不应声称V2已经重新跑出完全相同的指标。

### 2. SQLGlot、DuckDB与结果语义校验

> SQL进入数据库前会先经过SQLGlot。SQLGlot是SQL Parser，我们用它把SQL解析成AST，从语法树层面检查是否只有一条语句、是否为只读查询、是否包含写操作、是否使用`SELECT *`，同时提取CTE、子查询和嵌套语句最终访问的真实表和字段，再与白名单及当前用户权限比较。它负责判断“这条SQL允不允许执行”。
>
> 通过静态检查后，再交给DuckDB或目标数据库实际执行，由执行引擎检查字段是否存在、类型是否匹配、Join和聚合是否合法，并返回真实结果。它负责判断“这条SQL能不能执行，以及执行结果是什么”。
>
> 但是能安全执行不代表业务口径一定正确，所以最后还会由Validation节点结合原始Query、局部Schema、SQL和结果集做语义验收。可以简单概括为：SQLGlot是安检，DuckDB是执行引擎，Validation是业务验收。

### 3. 当前多库范围、日志统计及追问引导

> 从现有业务场景和脱敏评测集来看，绝大部分Query是单库查询，多库Case主要涉及两个业务库。我们在日志中保存每个SQL步骤实际访问的`database`，按照一次Query对`sql_steps.database`去重，就能统计平均数据库数、最大数据库数、多库Query占比、跨库成功率和P95耗时。

如果确实分析过真实生产日志，可以说：

> 我们按照一次Query中`sql_steps.database`去重统计，现有日志里一条Query最多涉及两个数据库。

用于自然引导规模化追问：

> 当前字段级检索在一到两个业务库范围内效果比较好，但它有一个比较明确的规模边界。如果数据库增长到上千甚至上万，平铺字段检索的候选噪声、权限过滤成本和检索延迟都会明显上升。

### 4. 如果扩展到一万个数据库

> 如果扩展到一万个数据库，我不会把全部Schema放进模型上下文，也不会向模型注册一万个数据库工具，而会把当前字段级检索升级为分层Schema Router。离线按照“业务域—数据库—表—字段—关联关系”建立多级元数据索引；在线先根据租户和用户权限过滤，再通过数据库摘要索引和全局字段索引两路召回候选数据库，把字段命中证据聚合到数据库级，经过融合和Rerank缩小到几个数据库。之后只在候选库内部进行表级和字段级检索，最后使用SchemaGraph补全Join字段和中间桥接表。
>
> SQL生成阶段，单库继续走现有单库Agent；多库查询则拆成各数据库内部可以独立执行的子查询，把过滤和聚合尽量下推到数据源，只返回小规模中间结果，最后通过Trino这类联邦查询引擎或应用层合并。如果一条Query真的需要实时访问成百上千个数据库，我会把它判断为数据架构问题，提前汇总到数仓、湖仓或物化视图，而不是让Agent实时扇出。
>
> 评测除了最终Execution Accuracy，还要增加Database Recall@K、Table Recall@K、Column Recall@K、平均数据库扇出数和P95路由耗时。其中第一层优先保证Database Recall，因为正确数据库一旦被Router漏掉，后续模型能力再强也无法恢复。

### 5. 为什么把权限统一收束到MCP Server

> 第一版只有普通用户和管理员两个角色，所以权限逻辑直接写在后端。后续PM提出需要支持普通用户、业务负责人和管理员三类角色，而且角色还可能继续增加。如果继续通过`if/else`分别修改Schema检索、工具暴露和SQL执行，很容易造成各层权限不一致。因此我们把MCP Server升级为统一的策略执行点。上游只传递经过认证的用户身份，MCP统一处理工具可见性、字段级Schema过滤、SQL二次鉴权、限流和审计。
>
> 当前数据库数量不多，所以仍保留一库一工具，但每个工具只是数据库适配器，固定绑定数据源、连接池和只读执行器，不再重复编写鉴权逻辑。完成一次Token透传、Schema经MCP过滤、数据库查询统一走MCP的接入改造后，后续增加角色主要修改MCP侧权限配置，不需要调整LangGraph主流程。

这里的核心是把角色与资源权限配置化，而不是把新的角色继续写成分散的条件判断：

```text
可信用户身份
→ MCP读取角色策略
→ 过滤可见数据库工具
→ 过滤可检索的表和字段Schema
→ SQLGlot提取真实表、字段并二次鉴权
→ 限流、并发控制、只读执行和审计
```

字段权限也不能只靠“隐藏工具”实现。一库一工具只能粗粒度控制数据库可见性；同一张表中不同角色可见不同字段，需要Schema返回前过滤一次，并在SQL执行前对AST中引用的真实字段再次鉴权。

### 6. MCP如何鉴权，伪造`role=admin`是否有效

> MCP鉴权不信任模型或者调用方传入的角色。HTTP请求必须携带统一身份系统签发的Access Token，MCP Server首先验证Token签名、签发方Issuer、Audience、有效期和Scope，然后根据验签后Token中的用户标识`sub`查询权限中心，得到可信的角色以及数据库、表、字段权限。
>
> 如果有人直接访问MCP，在请求参数里写`role=admin`是无效的，因为角色不从工具参数、请求Body或模型输出中读取。没有Token或者Token验签失败返回401；普通用户即使跳过`tools/list`，直接伪造一个隐藏工具调用，也会在工具执行阶段重新鉴权并被拒绝；即使访问的是有权使用的数据库工具，只要SQL引用了无权查看的字段，也会被SQLGlot解析后的字段权限检查拒绝。用户可以伪造角色字符串，但无法伪造由可信身份系统签名的身份，MCP只相信验签后的Token和服务端权限中心。
>
> 真正需要防范的是合法管理员Token被窃取，因此生产环境还需要HTTPS、较短的Token有效期、Audience绑定、Token撤销、密钥轮换和审计告警。更稳妥的做法是Token主要携带稳定的用户标识，MCP根据`sub`实时查询权限中心，这样角色降级和权限撤销可以及时生效。

直接攻击时的判断可以快速回答为：

```text
无Token直接访问                 → 401
随便编一个Token                 → 验签失败，401
自己签JWT并写role=admin          → 签名或Issuer不可信，401
普通Token加参数role=admin        → 参数被忽略或拒绝
普通用户直接调用隐藏数据库工具   → 执行前重新鉴权，403
普通用户SQL引用同表敏感字段       → 字段级AST鉴权拒绝，403
窃取到合法管理员Token            → 在失效前可能成功，需要专门防护
```

一句话总结：

> 模型负责提出工具调用，Access Token证明调用者是谁，权限中心决定其能访问什么，MCP Server负责强制执行；模型输出永远不能成为授权依据。

## 一、项目量化评测

这个项目的量化评测主要基于约600条人工标注Query，覆盖约80到100张业务表和1000到2000个字段，其中30%到40%是复杂查询。

Schema检索采用字段级标注：

```text
Recall
= 正确召回字段数 / Gold字段总数

Precision
= 正确召回字段数 / 返回候选字段总数
```

整套BM25、Embedding、RRF和Rerank两阶段检索，将字段Recall从约72%提升到93%，Precision从约18%提升到43%。

SQL侧不比较生成SQL的字符串是否与Gold SQL完全一致，而是在固定数据库快照上分别执行人工Gold SQL和模型生成SQL，通过规范化后的结果集是否一致计算Execution Accuracy：

```text
Execution Accuracy
= 结果集正确的Query数量 / 全部Query数量
```

单库场景执行准确率约为81%，对应600条Query中约486条正确；加入校验回溯后提升到88.5%，相当于额外修正约45条Query。

多库场景从67%提升到70%，对应正确Query数量约从402条增加到420条，相当于额外修正约18条Query。

这里的7.5%和3%更准确地说是分别提升了7.5个和3个百分点：

```text
单库：88.5% - 81% = 7.5个百分点
多库：70% - 67% = 3个百分点
```

每个节点还会保存结构化日志，用于定位和沉淀Bad Case：

- 路由错误：补充意图识别样本和Few-shot。
- Schema漏召回：补充字段同义词、业务别名和字段描述。
- Schema噪声过多：调整Rerank阈值和候选数量。
- Join错误：完善Schema中的表关系和关联键。
- 业务口径不明确：触发Human-in-the-loop人工澄清。
- SQL错误：结合SQLGlot检查结果和数据库报错进行修正。

> 口径说明：上述数据来自V1内部项目的近似实测结果；当前使用LangGraph、Skill、Human-in-the-loop和Mock数据的V2 Demo主要用于验证完整工程链路，不能直接声称V2已经重新跑出了相同指标。

## 二、SQLGlot静态安全检查

SQLGlot是一个SQL Parser。项目使用它把模型生成的SQL解析成AST，也就是抽象语法树，再从语法结构层面进行静态检查，主要包括：

- 是否只有一条SQL语句。
- 是否属于`SELECT`或`WITH`只读查询。
- 是否包含`INSERT`、`UPDATE`、`DELETE`、`DROP`、`ALTER`等写入或管理操作。
- 是否使用`SELECT *`。
- SQL引用的数据表是否在白名单中。
- 当前用户是否具有对应数据库和数据表的访问权限。

项目不会只通过正则表达式搜索`DELETE`或`DROP`等字符串，因为这种方式难以可靠处理CTE、子查询、嵌套结构、大小写和注释。SQLGlot先理解SQL的语法结构，再检查对应AST节点，因此比简单字符串匹配更可靠。

需要注意，权限控制不是SQLGlot自身提供的功能。SQLGlot负责从AST中提取真实表名，项目代码再将这些表与Schema白名单及当前用户的权限范围进行比较。

SQLGlot的职责可以概括为：

```text
判断这条SQL是否属于系统允许执行的安全查询
```

## 三、DuckDB实际执行检查

DuckDB是真正执行SQL的嵌入式分析数据库。当前Demo会将CSV文件注册成DuckDB只读视图，因此不需要额外部署独立数据库服务。

例如：

```text
orders_current.csv → orders_current表
customers.csv      → customers表
```

SQL通过SQLGlot检查后，再交给DuckDB执行。DuckDB在实际解析、绑定和执行过程中进一步检查：

- 字段是否真实存在。
- 字段引用是否有歧义。
- 字段类型是否支持当前运算。
- Join、聚合和分组是否合法。
- 日期函数和类型转换是否正确。
- SQL最终能够返回哪些真实结果。

例如模型生成：

```sql
SELECT SUM(sales_amount)
FROM orders_current;
```

`orders_current`是合法表，所以SQLGlot的表白名单检查可能通过；但真实字段是`paid_amount`，表中不存在`sales_amount`，因此DuckDB会在执行时返回字段不存在错误。

DuckDB的职责可以概括为：

```text
判断这条SQL能否基于真实数据结构执行，并返回实际结果
```

## 四、为什么还需要结果语义校验

SQLGlot和DuckDB都无法完全判断业务口径是否正确。

例如用户问：

```text
统计实际销售额
```

模型生成：

```sql
SELECT SUM(order_amount)
FROM orders_current;
```

其中：

```text
order_amount：优惠和退款处理前的订单原始金额
paid_amount：客户实际支付金额，可用于统计实际销售额
```

这条SQL：

- SQLGlot检查可以通过，因为它是合法的只读查询。
- DuckDB执行也可以成功，因为表和字段都真实存在。
- 但它选择了错误的业务指标，最终业务语义不正确。

因此，SQL执行成功后还需要最终校验模型结合以下内容进行语义检查：

```text
用户原始Query
+ Schema字段含义
+ 生成的SQL
+ SQL执行状态
+ 查询结果集
```

三个层次的职责分别是：

```text
SQLGlot
→ 安全性和静态结构是否合法

DuckDB
→ SQL能否在真实数据结构上执行

Validation
→ SQL的业务语义是否满足用户问题
```

一句话记忆：

> SQLGlot是安检，DuckDB是执行引擎，Validation是业务验收。

## 五、执行链路日志具体记录什么

项目保存的不是一条简单的文本日志，而是一次Query从路由到校验的结构化执行链路，主要包括以下几部分。

### 1. 路由日志

```json
{
  "route": "database_query",
  "confidence": 0.96,
  "reason": "用户请求查询新的业务数据",
  "signals": ["fresh_data"],
  "needs_database": true
}
```

它用于记录当前请求被分到数据库查询还是普通数据问答，以及模型的置信度、判断原因和辅助信号。

### 2. Schema检索日志

Schema检索阶段记录：

- 原始Query和检索关键词；
- 召回字段的物理名称、类型、描述和语义角色；
- 字段最终得分、BM25排名、向量排名和Rerank分数；
- 命中的数据库、表、主键和关联关系；
- 最终交给规划模型和Coder的局部Schema。

示例：

```json
{
  "query": "统计本月各客户等级销售额",
  "keywords": ["本月", "客户等级", "销售额"],
  "fields": [
    {
      "full_name": "sales_db.orders.paid_amount",
      "data_type": "DECIMAL",
      "description": "销售实付金额",
      "semantic_role": "metric",
      "score": 0.82,
      "keyword_rank": 3,
      "vector_rank": 1,
      "rerank_score": 0.94
    }
  ],
  "tables": [],
  "relations": []
}
```

### 3. CoT规划日志

规划阶段既保存模型的原始输出，也保存解析后的显式四元组：

```json
{
  "step_no": 1,
  "database": "sales_db",
  "processing_objects": "orders、customers",
  "operation_instruction": "按客户等级分组汇总本月销售额",
  "output_target": "各客户等级及其销售额"
}
```

### 4. SQL执行日志

每个CoT步骤对应一条执行日志：

```json
{
  "database": "sales_db",
  "cot_step": {},
  "local_schema": "本步骤使用的局部Schema",
  "sql": "SELECT ...",
  "execution_request": {
    "database": "sales_db",
    "sql": "SELECT ..."
  },
  "execution_result": {
    "success": true,
    "columns": [],
    "rows": []
  },
  "execution_log": {
    "started_at": "2026-08-25T12:00:00Z",
    "duration_ms": 86.4,
    "database": "sales_db",
    "success": true,
    "row_count": 12,
    "error": null
  }
}
```

### 5. 校验与回溯日志

校验阶段记录：

- 当前是第几次尝试以及已经重试多少次；
- 本轮采用的修正建议；
- Schema召回结果、CoT、SQL及执行结果；
- 错误所在阶段和具体原因；
- 是否回调Schema检索、CoT规划、SQL生成或SQL执行节点；
- 上一次校验结果、各模块执行次数以及是否达到重试上限。

因此可以利用结构化日志定位Bad Case发生在哪一层，而不是只根据最终SQL报错进行猜测。

## 六、“最多涉及两个数据库”是怎么统计的

当前日志没有单独保存`database_count`字段，而是需要对一次Query的所有SQL步骤按照`database`去重：

```python
database_count = len({
    step["database"]
    for step in sql_steps
})
```

例如：

```json
{
  "sql_steps": [
    {"database": "sales_db", "sql": "SQL1"},
    {"database": "target_db", "sql": "SQL2"},
    {"database": "sales_db", "sql": "SQL3"}
  ]
}
```

虽然生成了三条SQL，但去重后实际涉及两个数据库。按照Query维度聚合日志，可以统计：

```text
数据库数量分布
平均涉及数据库数
最大涉及数据库数
多库Query占比
不同数据库的访问频率
跨库执行成功率
跨库查询P95耗时
```

需要注意口径真实性：当前代码仓库本身没有生产日志样本可以直接证明“最大值为2”。没有分析真实生产日志时，只能依据现有业务场景和脱敏评测集说明多库Case主要涉及两个业务库；确实完成日志聚合后，才能声称日志中的最大值为2。对应的面试说法已经集中在文档最前面。

## 七、如何引导面试官追问“一万个数据库怎么办”

这里不要故意说一个明显错误，而是主动说明当前方案的适用范围和规模边界。

回答当前设计时，先交代实际规模、字段级索引为什么适用，再主动指出数据库数量继续增长时会出现候选噪声、权限过滤成本和检索延迟问题。讲到规模边界后可以停顿，让面试官自然追问：

```text
如果有一万个数据库怎么办？
为什么当前没有数据库级Router？
一万个数据库如何选库和选表？
难道要向模型注册一万个MCP工具吗？
```

这样既没有否定当前设计，也为后续讨论大规模Schema路由留下了入口。可直接使用的话术已经集中在文档最前面。

## 八、一万个数据库时的扩展方案

一万库场景需要在现有字段检索前增加数据库级路由，通过数据库摘要召回和全局字段召回相互补充。字段命中的证据需要向上聚合到数据库级，然后再进行候选库重排；进入少量候选数据库后，才继续执行表级、字段级检索与SchemaGraph补全。SQL生成与跨源执行同样需要拆成数据源内子查询、计算下推、小结果集回传和最终合并。完整口述答案集中在文档最前面。

整个分层流程可以记成：

```text
用户Query
→ 租户、权限和业务域过滤
→ 数据库级召回与精排
→ 候选库内表级检索
→ 字段级检索
→ SchemaGraph补全Join路径
→ 单库或多库执行计划
→ SQL生成与安全校验
→ 数据源执行及结果合并
```

设计上的关键变化是：当前规模主要解决“在有限业务Schema中找到正确字段”，一万库规模首先要解决“在海量数据源中找到正确数据库”。

---

## 九、用户鉴权与MCP工具权限

### 1. 身份认证与权限来源

用户先通过统一身份系统登录，调用MCP时由可信后端在HTTP Header中携带Access Token：

```http
Authorization: Bearer <access_token>
```

角色不能放在工具参数中作为授权依据。MCP先校验Token签名、Issuer、Audience、有效期和Scope，验签成功后读取稳定的用户标识`sub`，再向服务端权限中心查询该用户当前拥有的角色以及数据库、表和字段权限。

整体链路可以概括为：

```text
统一登录系统确认用户身份
→ 可信后端透传Access Token
→ MCP校验签名、Issuer、Audience、有效期和Scope
→ 根据Token中的sub查询权限中心
→ 权限中心计算数据库、表和字段权限
→ Schema检索前过滤无权限元数据
→ MCP只暴露有权使用的数据库工具
→ SQLGlot提取真实表和字段并再次鉴权
→ 使用只读数据库连接执行
```

需要区分两个概念：

```text
身份认证 Authentication：确认“你是谁”
数据授权 Authorization：确认“你可以访问什么”
```

直接访问MCP也不会绕过权限，因为服务端不读取调用参数中的`role`。无Token或无效Token属于认证失败，通常返回401；Token有效但访问无权限工具、表或字段属于授权失败，通常返回403。`tools/list`隐藏无权限工具是减少误调用和上下文噪声，执行前鉴权才是最终安全边界。

如果Token内直接携带角色，还需要处理角色变更后的缓存和失效问题。更稳妥的设计是Token以`sub`为核心，MCP根据`sub`查询权限中心，并配合短时缓存，在性能与权限及时撤销之间做平衡。

### 2. 角色策略配置与字段级权限

第一版角色少时，可以在后端维护简单的普通用户与管理员分支；角色和资源维度增加后，应将权限改成集中策略配置，例如：

```json
{
  "business_owner": {
    "databases": {
      "project_db": {
        "project": [
          "project_id",
          "project_name",
          "budget",
          "actual_cost"
        ]
      }
    },
    "rate_limit_per_minute": 30
  }
}
```

一次请求需要在多个阶段使用同一份策略决策：

```text
Schema阶段
→ 只返回当前用户有权查看的数据库、表和字段文档

工具发现阶段
→ tools/list只返回当前用户有权调用的数据库工具

工具执行阶段
→ 不信任模型传来的工具名和SQL，重新计算权限

SQL检查阶段
→ SQLGlot解析AST，提取实际引用的基表与字段并逐项校验

运行阶段
→ 执行只读、超时、行数、频率和并发限制，记录审计日志
```

如果用户对表有权限但对其中的敏感字段无权限，Schema检索不能向模型暴露该字段；即使调用方手写SQL猜中了字段名，执行前的字段级AST鉴权也必须拒绝。对于`SELECT *`通常直接禁止，避免星号展开造成敏感字段泄露。

### 3. MCP工具不可见与一库一工具

现有小规模设计采用“一库一工具”：

```text
query_project_db
query_trade_db
query_finance_db
```

每个工具内部主要包含四部分：

```text
工具名称和功能描述
→ 告诉Coder这个工具查询哪个数据库

参数Schema
→ 规定模型需要传入SQL等参数

数据库执行器
→ 绑定固定数据库的数据源配置、连接池和SQL执行方式

安全限制
→ 校验用户权限、只读SQL、表白名单、超时和返回行数
```

一个简化的工具定义可以表示为：

```json
{
  "name": "query_project_db",
  "description": "在项目管理数据库中执行只读查询",
  "inputSchema": {
    "type": "object",
    "properties": {
      "sql": {
        "type": "string",
        "description": "需要执行的单条只读SQL"
      }
    },
    "required": ["sql"]
  }
}
```

工具在服务端固定绑定：

```text
query_project_db
→ project_db数据库连接
→ project_db表白名单
→ project_db只读执行器
```

用户只有`project_db`权限时，MCP Client调用`tools/list`只能得到：

```text
query_project_db
```

`query_trade_db`和`query_finance_db`不会出现在模型的工具列表中，因此Coder正常情况下无法选择无权限工具。

但是工具不可见不能作为唯一安全措施。如果调用方绕过`tools/list`，直接伪造工具名称或调用参数，MCP Server仍然会在执行前根据服务端注入的用户身份重新检查数据库权限和SQL中的表权限。

### 4. 一万个数据库时如何调整工具设计

如果扩大到一万个数据库，就不适合注册一万个工具，否则工具列表本身会导致上下文膨胀、模型选错工具和服务注册管理成本上升。

这时可以将“一库一工具”改为统一数据库查询工具：

```json
{
  "name": "query_database",
  "description": "在指定的已授权数据库中执行只读SQL",
  "arguments": {
    "database": "project_db",
    "sql": "SELECT ..."
  }
}
```

调用链路变为：

```text
Schema Router选出候选数据库
→ Coder生成database和sql参数
→ 后端读取服务端用户身份
→ 校验database是否在用户权限范围内
→ SQLGlot提取SQL引用的真实表
→ 检查所有表是否属于目标数据库和用户表白名单
→ 校验通过后路由到对应数据库连接并执行
```

数据库名称虽然由Coder填写，但它只是一项待校验参数，不代表用户自动获得该数据库的访问权限。真正的授权判断始终由MCP Server完成。

小规模时通过“一库一工具”降低模型选库难度，大规模时通过“统一工具加数据库参数”控制工具数量。无论采用哪种形式，服务端都必须在执行前根据可信用户身份重新鉴权，不能信任模型输出。
