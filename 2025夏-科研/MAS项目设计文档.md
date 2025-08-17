# MAS项目设计文档

## 7.11会议纪要
目前是功能导向居多  为了复用和规模  来提供agent分布式服务

目前认为agent都是A+MEM+ACT+环境 只是agent三模块的复制

新架构的分离 弱化mem 因为每个模块都用mem

因此 每个agent借助act独立和env模块互动 
agent不需要物理限制
system控制序列  因果  事件可以回滚
inductive bias控制参数 设定 随机

TL  八月开始  先定义API

## 基类的关系
```mermaid
    A[Agent] <--> B[Action]  --> C[Environment]
      ^                                   |
      |------------------------------------
```


## Agent
| 类         | 存储类型 | 存储方式           | 内存 | 磁盘 | 存储优先级 |
|------------|----------|--------------------|------|------|------------|
| Profile    | kv数据   | redis              | ✅   |      | 活跃度     |
| Perception | 无       |                    |    |   ✅   | 活跃度     |
| Planning 计划  | kv数据   | redis/pg           | ✅ （当前）  | ✅（历史）   | 时序       |
| Reflection 思考| kv数据   | 向量数据库/pg      | ✅ （当前）  | ✅（历史）   | 时序       |
| State      | kv数据   | 向量数据库/redis   |      |      |            |

- Profile：存储智能体的身份信息
- Perception：用于感知环境
- Planning：制定计划
- Reflection：对最近的一些记忆和对话的总结，对问题的反思
- State：智能体的状态

## Environment
| 类                   | 存储类型 | 存储方式 | 内存 | 磁盘 | 存储优先级 |
|----------------------|----------|----------|------|------|------------|
| Communication Network| 图数据   |          |      |      | 时序       |
| Events               |      |          |      |      |      时序      |
| Space Time Simulation| 时空数据 |          |      |      | 时序       |
| script               |          |          |      |      |            |

- Communication Network：智能体的关系网络，可以是社交关系，也可以是工作流程图（有向无环图）
- Events：用户投放的数据（包括发生的新闻等）
- Space Time Simulation：智能体以及模拟环境中地点的位置信息
- script：模拟剧本（如：定义吃饭需要多少个step等）

## Action
| 类           | 存储类型 | 内存 | 磁盘 | 存储优先级 |
|--------------|----------|------|------|------------|
| Communication|      |      |      |     时序       |
| Tools        |          |      |      | 使用频率   |
| Play         |          |      |      |            |

- Communication：与其余智能体进行交流
- Tools：可以使用的一些工具
- Play：模拟一些虚拟动作（例如：吃饭，睡觉等）

## System
| 类           | 存储类型 | 内存 | 磁盘 | 存储优先级 |
|--------------|----------|------|------|------------|
| Timer        |          |      |      |            |
| Inductive Bias|         |      |      |            |

- Timer：时间管理器，其中包括最长等候时间，超时处理方法，step与物理时间的映射等
- Inductive Bias：全局的一些规则，其中需要包含所有的更新方法









## 个人思考

如果env存了关系网 那么agent是否再存关系还是直接查表？直接查询！

同理 message 存哪：本身act有了 是否要env放一份记录 用于回滚？ 如果有反思和communication表 就回滚【认为回滚先不做 只提供接口】
交流可以涉及多播 


message怎么存：格式 ？
message可见性：怎么控制？

同理 play在act也有 是否env放一份？和env的script什么关系？
act的other可以自定义行为

env的script怎么兼容复杂协作的task chain？
【这部分直接全局统一】

是否可以把script移动到sys 让用户定义单个动作的顺序和多个协作的过程，且方便回滚和并发
【两个不一样】

system的某种全量更新方法？考虑component？
【SYS直接改很多agent表状态】
【先不管sys】


是否采用component模式 天然具备并行控制 包含动 静属性和状态控制 可见性控制 observer模式  解耦agent agent专注于模拟大脑


分布式的回滚问题

1 全局时钟的实现？ 
2 状态的快照？ 
3 事件影响范围怎么控制？【哪些类 同一个类哪些查询实体】
4 回滚的一致性问题-内存毫秒级 持久化分钟级


最终是逻辑架构确定后的物理架构实现 关注主从 状态同步 冷热回滚等（集群不仅简单扩大规模的高可用 要考虑强弱一致性问题）

0714
TL 周二先看逻辑
TL 周三开始写 听师兄分配 写到这周末就行 可以自建分支

0721
开会讨论结果
核心必要问题：agent的调度 结论是通过一个中心（认为控制中心管理分发）
控制agent的激活+发送限制+接受act
并且流程要支持agent自发和用户自定义
中心为agent提供图 agent专注子图的输入输出
perception应提供agent自发和用户自定义优先级

小问题的结论
1 act全局独立 agent的profile中定义可用的act即可
2 component的父子结构问题  承认不完备 因此尽量扁平 但交给用户可自定义插件和子插件 修改原插件
如state一定要子结构
3 agent要支持交给用户自定义执行组件间的顺序【component依赖检查先不做】
4 perception是要优先队列【但打断功能先不做 要打断也是整个流程结束后打断 不支持step级别打断】
5 plan时走一步看一步怎么读取目前自己的执行状态是实现细节 不关心
6 支持env全局组件化

0723
开始编写和合并代码
个人负责agent模块

0724
TODO 个人的plugin粒度要到component 不要到大类【可以 但是保留最简化的统一继承
+加上组件识别！！！只能插上本component的插件  怎么设计 】  OKKKKKK  经过单元测试

TODO 代码对齐  有一些manager和数据库操作的命名要统一

TODO system最复杂 建议由简如繁 但要求先单机run 后面用ray 分布式的实现可以抄别的多agent框架
    阶段性目标 10 agent 根据random 社交网络和event交互=跑通单机

0725

Events应该有哪些插件，可以支持从外部数据源获取(例如 MCP 或数据库) OKK但是还没写

env新增一个“公共媒介”组件 用于社交产物  统一存储到中心  去CRUD  perception去获取  例如如果是发帖子的社交模拟
然后消息也是此公共媒介下的一个 对话是 Agent 的交互历史，应支持检索、溯源和上下文记忆
两两消息在ENV中操作一个列表  放内存  等消息结束就放外存  ENV共享一个    不关心agent的隐私  

TODO 定一下目录结构 如agent分base和component两个file  然后component下分各种file  每个file下是具体的component和plugins文件夹 OKK

TODO 【plan的插件还没写！！！！参考一下其它仓库！！很复杂】
【至于step  plan就plan里的memory  plan也有线性和树形  交给plugin】
定一下plan下component的公共属性（如维护当前做到哪一步  线性的step列表）和公共方法  公共内容越少越好
其它的component同理

TODO perception方面 可以先个人定好component  plugin确实要联调 所以perception的plugin先不写


0728

TODO   大类命名【module】+component+plugin

TODO    组件下 对DB的查询 通过【MCP】等方案  做中间层 接入具体plugin  ，   Component include 【中间层】
调研MCP
Component                                              <- plugin
   |-- 【connectivity 中间层 检索和使用plugin及其公共方法 生成统一query 可拓展】     <- plugin
调研MCP适配
    中间层只负责规范调用框架原生的plugin方法  并提供plugin方法的拦截写法
    但用户要自定义方法时 需要自行在中间层“插座”实现诸如user defined接口 【用户若自行提供插件的】

TODO  每个组件的plugin提取固定写法的约束规范【】 新plugin不允许约束外的方法
因此每个component要确定个性化的 方法、参数形式的规范


模块内文档->code [模块内--模块间的联调]
                            |
模块内文档->code [模块内--模块间的联调]
                            |
模块内文档->code [模块内--模块间的联调]

0729

宏观问题
设计统一数据访问层 (Data Access Layer, DAL) 架构
类似后端统一数据访问框架（如 Java 的 JDBC/JPA，或 Python 的 SQLAlchemy）更灵活，以支持非关系型数据库。
面向接口和策略模式   上层业务逻辑不关心具体实现，调统一接口


统一数据库问题
接口层：
定义标准、通用的数据操作方法，例如 create(), read(), update(), delete(), query()
方法参数要足够通用，比如使用字典或数据类 (dataclass) 来传递数据，
并用一个 model_name 或 schema_name 字符串来标识操作的是哪种数据模型（例如 'agent_state', 'communication_log'）。

Data Manager【昨天的中间层】 
解析来自接口层的请求。
核心职责：根据 model_name 和配置文件，决定将这个请求路由到哪个具体的数据库适配器 (Adapter)。
例如，'agent_state' 应该由 PostgresAdapter 处理，而 'agent_memory_embedding' 应该由 VectorDBAdapter 处理。
而且管理着所有数据库连接的生命周期，包括连接池的初始化和关闭。

adapter层：
基类接口（如 BaseAdapter）定义了必须被实现的 CRUD 等方法。
示例:
PostgresAdapter: 内部使用 SQLAlchemy 或 psycopg2，将通用的 create 请求转换为 SQL INSERT 语句。
VectorDBAdapter: 内部使用 pymilvus 或 pinecone-client，将通用的 search 请求转换为向量相似度搜索。
GraphDBAdapter: 内部使用 neo4j-driver，将请求转换为 Cypher 查询。

最后 配置管理 (Configuration Management):
一个中心化的配置文件（例如 database.yml



插件边界问题  功能型和数据型
【老师认为静态的数据可以作为plugin  我们认为除了profile等可以作为数据插件 state依然是方法  动态更改的network等共享数据不应该作为数据插件导入 而是DB】
state还是功能插件 统一外存 先不考虑开销


开会结论：environment的plugin就是数据  不是功能  功能做到component里

社交产出是公共的 累加的参数  主要是分发等功能  功能插件 不是数据插件
而社交网络是开始初始化的数据插件




Agent下的每个组件下的插件共性：【因为今天达成共识 有公共的数据接口 不含CRUD 专注业务逻辑】
1. Profile (智能体身份信息管理)
管理和提供智能体的身份、角色、目标等静态或动态信息。
核心业务逻辑：无论身份信息是静态的、动态的还是基于角色的，所有Profile插件最根本的功能是在需要时能够生成并提供一个当前时刻的、完整的身份视图。这个过程可能很简单（直接读取），也可能很复杂（需要计算和整合），但其最终产出是相同的。
最小共性定义：
方法: get_profile() -> ProfileData
返回值: 一个结构化的数据对象（例如，一个Pydantic模型 ProfileData），包含了如ID、名称、目标、价值观、技能列表等所有身份相关字段。这个方法封装了所有内部逻辑，对外提供一个统一的身份快照。

【插件本身 可以有公共的身份 隐私的身份等等】



2. Perception (环境感知模块)
核心职责：接收来自环境的原始数据，并将其转换为智能体可以理解的结构化信息。
插件设想：
TextPerceptionPlugin: 处理文本信息，如聊天记录、日志文件。
VisualPerceptionPlugin: 处理图像或视频信息，识别物体、场景。
SensorPerceptionPlugin: 处理来自模拟环境的传感器数据，如温度、位置。
核心业务逻辑：本质都是一个数据。一种特定原始输入（raw_data），然后通过处理（解析、识别、分析），输出为统一的、对智能体有意义的内部表示（PerceptionData）。
最小共性定义：
方法: perceive(raw_data: Any) -> PerceptionData
【区分主动 被动  然后每类下面有filter和优先级prioritize】

参数: raw_data 的类型是 Any
返回值: 一个标准的 PerceptionData 对象，其中可能包含：感知到的内容、来源、时间戳、重要性评估等。


3. **Planning** (计划制定与执行)
核心职责：根据当前状态和目标，创建一系列待执行的动作或任务，即“计划”。
插件设想：
【planning最简化   plan 根据内外信息  通用的调用action方法 不涉及具体行动】
【planing概念上应该高于perception等组件】

step里可以调的是【 action下的动作(communication or toolMCP)  /  agent里的其它组件能力 】


【plan下的plugin共性？】
plugin1.plan(){} 一次性生成多步
plugin2.plan(){}
【按照是否固定流程定义不同插件】



根据每步或之前的结果 “合心意” reach goal交给LLM自己判断

component里 固定写死 去不同数据源获取数据 再plan
    COMMUNICATION = "communication"  # 交流计划
    TOOL_USAGE = "tool_usage"       # 工具使用计划
    OTHER = "other"                 # 其他计划


SimpleGoalPlannerPlugin: 针对单一、明确的目标，生成一个线性的任务列表。
HTNPlannerPlugin (Hierarchical Task Network): 将高层级的复杂目标分解为层级化的子任务网络。
ReactivePlannerPlugin: 不制定长期计划，而是根据当前的感知信息，快速生成立即要执行的动作。
PolicyBasedPlannerPlugin: 基于一个预先训练好的策略模型（如强化学习模型），在给定状态下选择最优动作。
核心都是一个决策生成过程。输入是“目标”（Goal）和“世界状态”（WorldState），输出是一个“计划”（Plan）。
最小共性定义：
方法: plan(goal: Goal, current_state: AgentState) -> PlanTree
get_next_action(plan_tree: PlanTree, current_state: AgentState) -> Action
replan(current_plan_tree: PlanTree, event: Event, current_state: AgentState) -> PlanTree:

作用: 根据指定的目标和智能体当前的状态，生成一个行动计划。
参数: 需要明确的 goal 和 current_state 作为决策依据。
返回值: 一个结构化的 Plan 对象，其中包含了一系列 Task 或 Action。


4. Reflection (记忆管理与反思)

核心职责：分析过去的记忆，形成更高层次的见解、摘要或新知识，用于指导未来行为。
插件设想：
SummarizationReflectionPlugin: 定期回顾最近的记忆，并生成一段摘要。
InsightGenerationPlugin: 从大量经验中提炼出新的、可泛化的规则或“人生经验”。
核心不是CRUD 是知识合成（Knowledge Synthesis）。 创造出新的、更高层次的信息（Insight 或 Reflection）。
最小共性定义：
方法: reflect(memories: List[Memory]) -> List[Insight]
作用: 分析一组记忆，并从中生成新的见解。
返回值: 输出是一个或多个 Insight 对象。一个 Insight 是一个结构化的数据，代表了新形成的知识、摘要或发现。

【额外：：：属性: trigger: ReflectionTrigger
每个反思插件都应声明其激活条件。例如，“每日总结”插件是周期性的，而“失败复盘”插件则由action_failed事件触发。
generate_questions(insights: List[Insight]) -> List[Question]
优秀的反思不仅是总结过去，更是为了启发未来。此方法基于新生成的“见解”，提出智能体可以进一步探索的问题，这些问题可以转化为新的规划目标，形成一个学习闭环。】

5. State (智能体状态管理)
核心职责：管理和更新智能体的内部状态，如情绪、资源、认知状态等。
插件设想：
EmotionalStatePlugin: 根据发生的事件（如“任务成功”、“受到攻击”）来更新智能体的情绪状态（如“开心”、“愤怒”）。
ResourceStatePlugin: 跟踪智能体的资源消耗和补充（如“能量”、“金钱”）。
CognitiveStatePlugin: 管理智能体的认知资源，如“注意力”、“疲劳度”。
根本作用是定义“在什么事件（Event）发生时，状态应该如何改变（StateChange）”的规则。【数据本身放在外存】

最小共性定义：
方法: update_state(event: Event, current_state: State) -> StateChange
作用: 根据发生的事件和当前状态，计算出状态应该如何变化。

decay(current_state: State, time_delta: float) -> StateChange: 处理状态随时间的自然变化。例如，情绪会随时间平复，饥饿感会随时间加剧。


结论：除了planning下的plugin要干什么 其它agent下都确定好了


0730
会议
设计架构的角度:
用户开发  系统开发  可实现

开会内容：
数据操作 就每个DB一个mcp协议
我们框架写好这个协议即可
plugin自己调mcp 自己传表名等参数即可
plugin只有功能业务函数

且component直接用mcp调插件
插件是唯一功能 不需要插件的公共方法
且可以写死plugin的执行顺序也可以模型决定调用mcp

！确定每次做一个插件！
所以单个插件就应该包括一个组件下的所有功能


TODO 本人用MCP试试profile 数据插件？和state  
KV数据联系到数据库 
【component不能互相调用！】 planning这个组件只能访问其它组件下的数据库 而不是组件功能
现在提出了每个模块下的要一个DB  然后plan或者reflect可以直接调用感知等的DB
模块写完后可以写demo的插件


0731：已经完成的重构工作：
保留了最佳实践：相对引用关系

profile和state两大组件  依然通过基类继承保留插件识别功能。但是只作为插件基座，不关心插件业务逻辑
【是否可以经过plugin提取直接的get方法 存疑】
将 ProfileComponent 和 StateComponent 的职责简化为仅管理其对应的插件（ProfilePlugin 和 StatePlugin）。
所有具体的业务逻辑（如Profile属性管理、State状态管理）均已下沉到插件层。

Profile下：
ProfileStoragePlugin 定义了管理Profile数据的接口（get, set, delete, has, import, export），并通过 register_mcp_operations 方法将具体实现委托给用户通过MCP提供的工具。

State下：
StateStoragePlugin: 与 ProfileStoragePlugin 类似，负责定义和委托状态数据的CRUD及导入/导出操作。
StateTriggerPlugin: 负责响应外部事件（如 process_event）和逻辑时间步（如 process_tick），并调用 StateStoragePlugin 来修改状态。



0801
又要求插件随便写  组件里硬编码 写死插件的具体方法 不需要插件共性
组件里有一个统一的execute方法  外面统一调用execute

以agent为例
各种组件都用公共的CRUD接口  和execute   具体执行里用户自己调具体插件的方法
【execute里具体的插件可以调用其他组件的公共CRUD】

以perception为例 CRUD都是交给其他模块调的 被动感知  每个组件的execute里都是主动的方法感知

但每个component的CRUD就意味着每个component都要统一一种数据结构  只是plan和perception的数据结构比较复杂
例如感知的渠道不同 用不同的插件实现

profile和state 嵌套的KV
reflect：文档型数据
plan：嵌套KV  goal step123都是一块KV
perception  每个【插件下都要实现一个其他数据结构转KV的方法】 

然后具体到每个组件的KV具体有什么字段 就

晚上开会：
基本达成共识 就是每个组件的CRUD
qmj有自己的分支
dr有改network
环境不需要插件 只需要写在action 然后动作去调用具体的环境里的CRUD





0805会议 
profile的插件直接使用组件方法 不用自己写方法 在init中保留：导入到数据库
然后一个插件一份profile 依然作为数据

state的状态变化问题 在action中做 甚至trigger action
因此同理profile

环境里各种复杂的图查询都可以放到perception交给用户自己写插件
目前的环境 只有mcp server 或者mcp component 作为中层 可以让client调用 或者sys调用
adapter放别的 client放agent

0806会议 

对于环境 认为空component作为基座   把所有图数据库操作变成一个plugin 即mcp server  自己实现 甚至是adapter也就放在plugin用户自己写

但是为了系统层统一监管调用  需要sys或者哪里 写一个基本数据库基本CRUD   这种sys里必须硬编码   普通实体都是mcp

老师认为多个component都要用同一个plugin没关系 直接复用代码 重复代码没关系 每个plugin里都用重复的读写DB没事

实在要统一adapter可以放在MAS之外的一个toolkit

【agent模块】限制用户只在plugin层更改 坚持增删改查就是抽象方法 然后放组件
组件的CRUD要通过具体的adapter实现  adapter下放插件或者外部toolkit来做 
如何选取具体的plugin的adapter：交给用户硬编码 或者随机 或者怎么样

sys是否需要存储：agent调工具的记录 用户查看的   可以再写日志


【结论 敏捷开发 先统一用KV  就避免用多样的adapter】
adapter先留好都行  可以先不写adapter

学生个人讨论：
plugin不涉及底层数据库交互  DB交互都由component抽象方法和更外面的adapter实现
大agent一个exe() 里面顺序调每个组件的exe()  每个组件的exe调用具体plugin

sys:timer 消息调度（发布话题）

action要联调

adapter要先写   基于redis  get set update delete
一期分工：我和明杰合并一下agent即可  agent里写一下LLM的API config    其它两人environment
二期：4人 每个一个大模块   做联调


0807：
开发完毕：实现重构 目前
职责明确: ProfileComponent 和 StateComponent 作为数据操作的中心枢纽，其抽象的 CRUD 方法（set, get, update, delete）保持不变，负责与未来的数据适配器（Adapter）交互。
plugin专注于数据存储和导入逻辑 (profile_data, state_data)。
并且编写单元测试 插拔不同profile和不同state的单元测试通过

问题：这种属性类型的组件 有数据流向问题 相当于数据插件的值只在初始化单方面存一次 后续只与DB交互
如果plugin主动插一次 一次性流向component -> adapter -> DB即可
【但当前设计下，后续component的修改查询只交给adapter去DB里完成 插件内部的字典要不要同步？这是一个设计逻辑问题】



0808：
学生会议
基于假设：每个组件的数据库应该统一
那么现在组件里依然有CRUD 了   老师觉得的插件个性化独立adapter没必要

个人：全权负责agent  等toolkit的adapter写好直接用


0810开发工作：
获取adapter和环境的更新，直接开始重构agent下剩余三个组件的结构和实例插件的编写。

设计问题：
perception的优先队列具体到插件粒度 用户自己写优先逻辑 perception的组件只负责感知数据存取


具体考虑：
一 agent_base中使用统一的LLM配置项（要连tool）。
二 粒度细到每个样例插件的单元测试写好
三 接入具体的adapter再测试好 即可
四 最终agent_base要支持编排执行内部component，并且agent模块要联调 与其他模块通信和调试


目前结果：数值插件inti时：单向数据流：插件 -> 组件 -> 放入数据库。
数值插件的更新和查询逻辑：更新插件值 (内存操作)，第二步(持久化)用户通过业务逻辑显式调用组件方法来保存变更。查询时也显示指定查插件还是DB  插件永远更快更新


伴随问题的 Q & A：
agnet剩下三个组件的插件都是功能插件，不是数据，那插件是否还要留一份数据？是否这些组件还需要统一的导入导出方法？ans:形式上这些功能插件的数据是临时缓存 各有用途
伴随的问题：功能组件的导入导出也是必要的 这个功能面向上帝用户或者说sys

对于plan:LinearPlannerPlugin: 不持有所有计划的列表。它只在被调用 create_plan 时，去各个组件拉取数据，然后将生成的一个计划存回去。

对于反思下的InsightCreationPlugin: 核心是 generate_insight 方法，从 PerceptionComponent 拿数据，然后将结果存入 ReflectionComponent 只是中间处理过程，拿到的数据被视为一个临时缓存

对于感知组件，确实比较特殊，例如MessagePerceptionPlugin: self.message_queue 不是数据库的完整拷贝，而是一个用于实现“优先级处理”功能所必需的操作性数据结构。因此确实要再存一份进行操作用




0811学生会议
由于图太复杂 需要具体到业务逻辑的中间层 中间层方法要调整后得到最原子的方法 统一一套模板参数  作为adapter  具体让不同的adapter（数据库）按照相同功能写不同的中间过程
例如add_relation等  即这些功能要拓展到数据结构操作   
但是非业务逻辑  业务一定要在组件写

共识：Env里 采用横向插件 拓展的是组件函数 供其它调用
此时agent直接调用环境组件的方法即可 没有方法就报错即可

然后adapter共识：顶层有公共的方法和参数  底层实现redis的KV一套
目前组件统一先用KVStoreAdapter

TODO 感知比较复杂  需要加relation和message  然后共识是感知存消息记录  环境也存消息记录
目前只有relation的感知要与环境交互 而且是硬编码去查有什么函数什么方法 不用MCP
Plan里要先感知 然后判断要不要回复等action  再去调用action即可

TODO 反思可以强制在agent大类执行时做 保证每轮以反思结尾
plan目前设计一次性plan 然后直接按步骤执行OKK    目前不涉及sys的timer


0813五个agent下的开发内容：
OKK 感知层重构 (Perception)：RelationPerceptionPlugin 需要与环境中的 communication_network 直接交互，通过硬编码方式调用其方法，并处理可能的方法缺失错误。

OKK【这个改动在agent_base的test中有写样例 即插入component的时候就可以指定好adapter 不用在具体component一一赋值】数据持久化适配 (Persistence)：所有Agent的核心组件（Profile, State, Perception, Planning, Reflection）都需要统一使用 packages/tools 下的 RedisKVAdapter 实例进行数据存取，且不能修改 tools 包。

OKK 规划层保留 (Planning)：LinearPlannerPlugin 的具体实现暂时搁置，保留其结构，
添加虚拟的step逻辑 为未来由LLM驱动和系统时钟控制的逻辑步执行做准备。

OKK Agent执行流定义 (Execution Flow)：在 agent_base 中硬编码定义组件的执行顺序为：
Perception -> Planning -> Reflection，确保每轮交互以反思结束。

OKK 测试验证 (Testing)：完成上述功能后，修改现有的单元测试来验证新架构的正确性。


【问题 目前的KV adapter只有create 而不是set 等统一命名 会在调用时出问题
因此目前的在agent_base的test还未通过  其它无殊】
【问题2  main分支和远程的wxj分支的合并冲突问题  最好用merge 然后解决一下冲突 再议】


0813学生会议
plan的格式  是否指定大类action  然后交给action分发到具体的tool等 根据意图提取参数
【先统一所有plan插件生成的step格式】
两次意图分发 减少context【可以参考agent society仓库的plan_block.py的step格式】
【且现在的plan可以仿照上述的  通过plan中的一个prompt去规定可以干什么 希望干什么】

【目前成本不考虑  因此可以直接把agent所有profile等 传到plan的一个很长的prompt中即可】
【plan按照step分步执行 然后每步去action找分发路由 到具体的方法进行执行】

现在action需要调用perception的对话历史 通过agent_id来查  perception要提供

【TODO 现在perception要调的环境里的内容只有relationship 和messagehub】
然后调用方法是environment实例.xxComponent.xxx方法来调  但是目前env还没跑通

【TODO  现在system有了 里面有消息的派发 要调用perception的add方法 要看着是配一下】

【TODO 参照action  让agent的具体组件只能插一个插件】


0814开发内容
一、OKK 限制agent下的组件也为单插件的形式。【通过重写具体component的add_plugin实现】
使其在尝试添加第二个插件时引发一个 Exception，从而保证单一插件的约束。相应地更新 remove_plugin 和 get_plugin 方法。移除不再需要的 list_plugins 方法。【但是perception可以插多个插件 因此直接用父类add_plugin】
二、OKK 规范LinearPlannerPlugin的Plan输出格式  type 的值必须是 communication、tool 或 other 交给action进行路由分发。并且此插件其它方法都针对新格式再适配
三、OKK 启动环境注入。修改agent三级的base，以支持环境注入。agent对环境实例的接受和操作：使用单例模式，系统启动时创建唯一的 Environment 对象。并且把【引用】传递给每个 Agent，再由 Agent 传递给它的组件（Component）和插件（Plugin），保证共享同步env的数据，避免内存开销。
四、感知适配 Perception对System和Environment要暴露一些接口 且还要暴露给Action获取对话历史等（get_chat_history）（add）  【目前写在具体的感知plugin中  这个要一直更新拓展】
五、最终审查与测试 编写或者修改单元测试用例


0815会议
action要不要独立 
老师认为：统一的action模块利于同步更新 解耦动作 
共识：agent要有一个tool box
agent可以加act组件  但是真正调用的还是独立的action模块
老师认为大脑智能应该完全限制在agent中 即plan要完全细粒度规划完整 
不能把路由等智慧问题放到action中

老师认为plan用户自己实现中间过程 要留出可定制的入口   
只需要【细粒度的可执行的带参数的具体action】作为plan的结果即可。 
怎么生成计划 计划的格式 中间过程 都交给用户自己实现！

老师坚持plugin自己拥有自己的adapter 不要公共 没法公共
【老师认为用户会自己处理调哪个插件adapter的问题】
老师认为sys的adapter和业务的插件adapter分开就行
【sys有自己的存储和自己的adapter就行！！功能的adapter本来就没法复用  因此直接插件调其他组件下的插件即可！！】
sys有自己的存储 包括config 日志 等等 

【模块内的插件互调依赖问题  有缺失更改 直接报错就行  示例只要保证一套自洽的插件即可】

最后发布时候注释一定要英文
【ans:现在不可控  agent只能自由plan  真正执行和调用插件方法都是硬编码】
【ans:系统adapter可以直接拿数据 用户自己写 不用经过插件的adapter】

TODO
重构agent  下方到插件adapter 组件无公共方法   
然后plan只到细粒度 中间过程不管

问题：环境是否要关心业务逻辑，要 就可以做多插件 不要业务逻辑，那么环境的插件只需要CRUD和复杂查询的单个插件。如果这个插件必要，那就不应该是插件了，而是直接写到组件下，环境就没有插件了，不符合统一的三级架构。怎么办。
【共识：简单复杂查询都放到插件  组件初始化的时候统一加载一下所有插件  后续直接组件.方法即可】

sys的config只有对agnent的插件list初始化


0817开发内容
OKK 第一，目前五大component都有一些公共adapter供其下的plugin调用。现在adapter全部下放到插件! 现在要求组件无公共方法。
即目前每个组件的每个plugin自己拥有自己的adapter 不要公共，并且插件方法调用自己的插件adapter进行数据操作，但是函数方法不变。
注意，每个插件的每个adapter使用的是toolkit下的adapter实例的引用，不需要真的为每个插件的adapter反复创建实例。

OKK 第二，plan的具体插件（例如现在的线性plan）要返回可执行的一系列的最细粒度的action模块下的具体方法。不能只传递模糊意图给action模块去选择具体动作。
plan插件中，用户怎么生成计划 计划的格式 中间过程 都交给用户自己实现！要自己实现中间生成具体动作的过程 要留出可定制的入口，接受一个goal，最终生成可执行的具体方法（但是不执行）。

OKK 第三，对于agent模块内的跨插件互调依赖问题  直接硬编码 如a插件调用b插件的c方法，如果方法或参数不正确 直接报错就行

OKK 第四，agent_base这个文件下是将来需要实例化的一系列agent，那么思考是否需要id等属性，自行添加或修改。

OKK 第五，在agent的test中进行修改或添加函数，实现上述重构的单元自测。重点在测试时实现一下对一个新agent实例，读一系列插件列表，自动完成插件的组装。

OKK 第六，单测显示，perception对最新environment的relation和messagehub的结构较大变化应该进行适配 
perception做了很多协同改动














目前已有的架构类图 在下述网站打开图即可
https://mermaid.live/
```mermaid
classDiagram
%% ========== agent ==========
class agent.Profile {
  - configs: Dict[str, ProfileAttribute]
  + add_attribute(name, value, description)
  + remove_attribute(name)
  + get_attribute(name, default)
  + has_attribute(name)
  + update_attribute(name, value)
  + add_attribute_value(name, value)
  + to_dict()
  + from_dict(data)
}
class agent.ProfileAttribute {
  - name: str
  - value: Any
  - type: Any
  - description: str
  + update_value(new_value)
  + to_dict()
}

class agent.StateAttribute {
  - name: str
  - value: Any
  - min_value: Any
  - max_value: Any
  - optional_values: List[Any]
  + update(new_value)
  + update_range(min_value, max_value)
  + update_optional_value(values)
  + to_dict()
}
class agent.BaseState {
  - _kv_attributes: Dict[str, StateAttribute]
  + add_attribute(key, value, min_value, max_value, optional_values)
  + update_attribute_value(key, value)
  + get_attribute_value(key)
  + remove_attribute(key)
  + update_attribute_optional_values(key, optional_values)
  + update_range(key, min_value, max_value)
  + has_attribute(key)
  + get_attribute_object(key)
  + get_all_attributes()
  + get_attribute_keys()
  + clear_attributes()
  + to_dict()
  + from_dict(data)
}
class agent.EmotionalState {
  + add_optional_emotion(emotion)
  + remove_optional_emotion(emotion)
  + clear_optional_emotions()
  + update_emotion(emotion)
}
class agent.CognitiveState {
  + set_focus(focus)
  + get_focus()
  + add_working_memory(item, timestamp)
  + clear_working_memory()
  + fetch_working_memory(count)
}
class agent.PhysicalState {
  + update_hunger(delta)
  + update_fatigue(delta)
  + reset()
  + hunger_level
  + get_fatigue_level
}

agent.BaseState <|-- agent.EmotionalState
agent.BaseState <|-- agent.CognitiveState
agent.BaseState <|-- agent.PhysicalState
agent.Profile o-- agent.ProfileAttribute
agent.BaseState o-- agent.StateAttribute

class agent.ActionPerception {
  - agent_id: str
  - message_queue: Queue
}
class agent.EnvironmentPerception {
  - agent_id: str
}
class agent.RelationshipPerception {
  + get_relationships()
  + get_agents_by_relationship(relationship_type)
  + get_agents_from_prompt(prompt)
}
class agent.LocationPerception {
  + get_agent_location(target_agent_id)
  + get_my_location()
  + get_nearest_location(location_type)
  + get_nearby_locations(radius, location_type)
  + get_nearby_agents(radius)
}
class agent.MessagePerception {
  + get_messages()
  + add_message(message)
  + clear_messages()
  + has_messages()
  + get_messages_from_others()
  + get_messages_from_self()
}
agent.ActionPerception <|-- agent.MessagePerception
agent.EnvironmentPerception <|-- agent.RelationshipPerception
agent.EnvironmentPerception <|-- agent.LocationPerception

class agent.PlanStatus
class agent.PlanStep {
  - step_id: str
  - name: str
  - description: str
  - type: PlanType
  - status: PlanStatus
  - started_at: datetime
  - completed_at: datetime
  - metadata: Dict
  + start()
  + complete()
  + fail()
  + to_dict()
}
class agent.BasePlanning {
  - agent_id: int
  - plan_id: str
  - goal: str
  - steps: List[PlanStep]
  - status: PlanStatus
  - current_step_index: int
  - created_at: datetime
  - completed_at: datetime
  - metadata: Dict
  - model_service
  - database_service
  + generate_plan_with_model(context)
  + get_goal()
  + get_steps()
  + get_current_step_index()
  + get_current_step()
  + get_remaining_steps()
}
agent.BasePlanning o-- agent.PlanStep
agent.BasePlanning ..> agent.PlanStatus

class agent.ReflectionType
class agent.BaseReflection {
  - agent_id: str
  - vector_db: Any
  + reflect(...)
  + _search_vector_db(query)
  + save_reflection(reflection_content, reflection_type, metadata)
  + _call_llm_for_reflection(...)
}
class agent.MemoryReflection {
  + reflect()
  + _get_recent_workflow_memories()
  + _call_llm_for_reflection(recent_memories)
}
class agent.ConversationReflection {
  + reflect(other_agent_id)
  + _get_conversation_history(other_agent_id)
  + _call_llm_for_reflection(conversation_history)
}
class agent.QuestionReflection {
  + reflect(question)
  + _call_llm_for_reflection(question, relevant_info)
}
agent.BaseReflection <|-- agent.MemoryReflection
agent.BaseReflection <|-- agent.ConversationReflection
agent.BaseReflection <|-- agent.QuestionReflection

%% ========== action ==========
class action.BaseAction {
  - name: str
  - description: str
  - required_params: Dict[str, str]
  + execute(**kwargs)
  + get_name()
  + get_description()
  + get_required_params()
  + to_prompt()
}
class action.EatAction {
  + execute(**kwargs)
}
class action.SleepAction {
  + execute(**kwargs)
}
class action.MoveAction {
  + execute(**kwargs)
}
class action.OtherAction {
  - actions: Dict[str, BaseAction]
  + add_action(action)
  + remove_action(action_name)
  + get_available_actions()
  + get_actions_prompt()
  + select_action(context, **kwargs)
  + forward(step_content, **kwargs)
}
action.BaseAction <|-- action.EatAction
action.BaseAction <|-- action.SleepAction
action.BaseAction <|-- action.MoveAction
action.OtherAction o-- action.BaseAction

class action.Tool {
  - name: str
  - description: str
  + execute(*args, **kwargs)
}
class action.FunctionTool {
  - func: Callable
  + execute(*args, **kwargs)
}
class action.MCPTool {
  - mcp_client
  - method_name: str
  + execute_async(*args, **kwargs)
  + execute(*args, **kwargs)
}
class action.ToolManager {
  - tools: Dict[str, Tool]
  + add_tool(name, tool, description)
  + add_mcp_tool(name, mcp_client, method_name, description)
  + remove_tool(name)
  + get_available_tools()
  + select_tool(name)
  + call_tool(name, *args, **kwargs)
  + call_tool_async(name, *args, **kwargs)
  + forward(tool_name, *args, **kwargs)
  + forward_async(tool_name, *args, **kwargs)
  + get_tool_info(name)
  + list_tools()
}
action.Tool <|-- action.FunctionTool
action.Tool <|-- action.MCPTool
action.ToolManager o-- action.Tool

class action.Communicate {
  - database_manager
  + analyze_intent(step_content)
  + get_target_agent_id(intent_analysis)
  + get_chat_history(agent_id, target_agent_id)
  + generate_message(agent_id, target_agent_id, step_content, chat_history)
  + save_message_to_database(agent_id, message, target_agent_id)
  + send_to_message_queue(message, target_agent_id)
  + forward(agent_id, step_content)
}

%% ========== environment ==========
class environment.Property {
  - _name: str
  - _value: Any
  - _description: str
  + name
  + value
  + description
  + get_value()
  + set_value(new_value)
}
class environment.ScriptBase {
  - _properties: Dict[str, Property]
  + _initialize_properties()
  + add_property(name, value, description)
  + get_property(name)
  + get_property_value(name)
  + set_property_value(name, value)
  + list_properties()
  + remove_property(name)
}
environment.ScriptBase o-- environment.Property

class environment.SpaceTimeEntity {
  - entity_id: str
  - position: Tuple[float, float]
  - entity_type: str
  - name: str
  - timestamp: datetime
}
class environment.SpaceTimeSimulation {
  - space_time_db
  + add_entity(entity)
  + update_entity_position(entity_id, position, timestamp)
  + get_entity_position_by_name(entity_type, name)
  + get_entity_position_by_type(entity_type)
  + get_historical_positions(entity_id, start_time, end_time)
  + remove_entity(entity_id)
  + add_agent(agent_id, position, name, timestamp)
  + add_aoi(aoi_id, position, name, timestamp)
}
environment.SpaceTimeSimulation o-- environment.SpaceTimeEntity

class environment.RelationshipType
class environment.RelationshipData {
  - source_agent_id: str
  - target_agent_id: str
  - relationship_type: RelationshipType
  - strength: float
  - created_at: float
  - updated_at: float
  - metadata: Dict[str, Any]
}
class environment.BaseRelationshipManager {
  - graph_db
  + connect_to_graph_db(connection_config)
  + add_agent(agent_id, agent_properties)
  + remove_agent(agent_id)
  + add_relationship(relationship)
  + update_relationship(relationship)
  + remove_relationship(source_agent_id, target_agent_id)
  + get_relationship(source_agent_id, target_agent_id)
  + get_direct_relationships(agent_id)
  + find_mutual_relationships(agent_id, relationship_type)
  + get_network_statistics()
  + close_connection()
}
environment.BaseRelationshipManager ..> environment.RelationshipData
environment.RelationshipData ..> environment.RelationshipType

class environment.BaseEvent {
  - id: str
  - event_type: str
  - data: str
  - timestamp: datetime
  + to_dict()
}
class environment.EventManager {
  - events: List[BaseEvent]
  + add_event(event_type, data)
  + get_event(event_id)
  + get_events_by_type(event_type)
  + remove_event(event_id)
  + clear_events()
  + save_to_database(event)
  + load_from_database(event_id)
  + update_in_database(event)
  + delete_from_database(event_id)
}
environment.EventManager o-- environment.BaseEvent

%% ========== 归属模块标注 ==========
note for agent.Profile "agent"
note for agent.BaseState "agent"
note for agent.ActionPerception "agent"
note for agent.BasePlanning "agent"
note for agent.BaseReflection "agent"
note for action.BaseAction "action"
note for action.Tool "action"
note for action.Communicate "action"
note for environment.Property "environment"
note for environment.ScriptBase "environment"
note for environment.SpaceTimeEntity "environment"
note for environment.BaseRelationshipManager "environment"
note for environment.BaseEvent "environment"
```


