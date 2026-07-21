---
version: 0720
module: agent-runtime
feature_type: functional
feature_id: FEAT-002
status: draft
merged_from:
  - agent-runtime-core-interface
---

# Versatile 意图识别工作流适配兼容特性文档

## 1. 特性定位

FEAT-002 已定义 `agent-runtime` 通过 `VersatileAgentRuntimeHandler` 代理远端 REST/SSE Agent 服务的通用能力，包括请求转换、URL/header/metadata 映射、结果提取、统一执行结果、会话连续性和基础异常断流检测。本特性在该通用能力上增加 Versatile 意图识别工作流的标准接入要求，使独立部署的一层、二层意图识别工作流能够通过同一套 Adapter 实现接入 Runtime，并向调用方返回可供后续处理的结构化结果。

本特性解决的问题是：客户低码平台中的 Versatile 意图识别工作流具有约定的字符串输入、结构化结果以及用户交互中断与恢复要求；通用 Versatile REST/SSE 代理尚未明确这些输入输出契约，也未明确如何把工作流显式产生的用户交互中断转换为 FEAT-008 的标准中断并恢复原工作流。`agent-runtime` 必须把这些工作流接口差异约束在既有 Versatile adapter 内部，向标准智能体服务入口保持统一的 Task、结果、失败、取消和用户交互语义。

同一个 `VersatileAgentRuntimeHandler` 实现必须能够通过不同 Runtime 实例的配置分别适配一层和二层意图识别工作流，每个 Runtime 实例只调用当前实例配置的一个工作流。工作流独立部署、标准 Agent 服务入口和 Agent Card 配置属于方案接入前提，由客户集成与部署流程完成。

本特性面向以下角色：

- Versatile 意图工作流开发者：按照约定提供工作流输入、正常结果和用户交互中断能力。
- Runtime 集成方：为每个独立工作流配置现有 `VersatileAgentRuntimeHandler` 和 `versatile.*`，并通过标准 Agent 服务入口发布能力。
- Adapter 开发者：在既有 Versatile adapter 内实现意图工作流请求映射、结果提取、中断转换和恢复请求适配。
- 测试与验收团队：验证一层、二层独立接入、输入输出映射、用户交互恢复、失败、取消和可观测行为。

本特性归属 `agent-runtime`，定义 Versatile 意图识别工作流的技术适配与标准结果转换。标准 northbound A2A 服务入口由 FEAT-001 约束；通用 Versatile 代理和 Runtime Adapter SPI 由原 FEAT-002 约束；Task 用户交互中断与恢复由 FEAT-008 约束；输出 `agent_id` 对应的逻辑 Agent Card 身份语义由 FEAT-015 约束。

## 2. 当前版本能力要求

| 能力 | 要求级别 | 事实要求 |
|---|---|---|
| 既有 Versatile 入口复用 | MUST | 一层和二层意图识别工作流必须复用 FEAT-002 已有的 `VersatileAgentRuntimeHandler`、`AgentRuntimeHandler.execute(AgentExecutionContext)` 和 `versatile.*` 配置入口接入 Runtime，不得另建职责重复的公开 Handler 或执行函数。 |
| 单工作流实例适配 | MUST | 同一 Adapter 实现必须能够通过不同 Runtime 实例的 `versatile.*` 配置适配不同意图工作流；每个 Runtime 实例只调用当前实例配置的一个工作流。 |
| 统一五字段输入 | MUST | Adapter 必须使用 `query`、`intents_id`、`intent_name`、`messages_role` 和 `messages_content` 调用当前意图识别工作流，五个字段均按当前低码工作流契约使用字符串类型。 |
| 当前主输入映射 | MUST | Adapter 必须将 `AgentExecutionContext` 中当前调用的主输入映射为 `query`。当前主输入由调用方提供，Adapter 统一执行输入转换。 |
| 意图配置传递 | MUST | `intents_id` 和 `intent_name` 必须来自当前独立工作流的开发者或部署配置；Adapter 负责读取、校验字符串类型并传递，不从 Agent Card 或用户输入推导。 |
| 消息信息可配置映射 | MUST | Adapter 必须支持从现有执行上下文或请求 metadata 映射 `messages_role` 和 `messages_content`，并由部署配置明确数据来源、是否必填和缺省字符串；Adapter 不推断角色、不生成或总结消息内容。 |
| 三字段正常结果 | MUST | 工作流正常完成时，Adapter 必须提取 `response_content`、`intent_id` 和 `agent_id`；三个字段必须存在且为字符串，`intent_id` 和 `agent_id` 必须为非空值。 |
| 结构化结果保留 | MUST | Adapter 必须在现有标准执行结果中以机器可读方式完整保留三个结果字段，使后续 Runtime 能力能够稳定读取；不得要求后续能力从自然语言或文本 JSON 中再次推导目标。 |
| 唯一 Agent 目标 | MUST | 当前版本一次正常结果必须只包含一个非空 `agent_id`；返回数组、多个目标或无法确定唯一值时必须形成结构化失败。 |
| 逻辑 Agent 身份 | MUST | `agent_id` 必须表达当前 tenant 下可与 FEAT-015 Agent Card 逻辑 `agentId` 对应的目标身份。Adapter 只执行字段结构校验，目标是否存在、可访问和可调用由后续目标发现与调用能力处理。 |
| 正常业务结果一致处理 | MUST | 匹配成功、未匹配和需要澄清都必须作为工作流正常结果处理，并返回约定的三字段结果；Adapter 不解释三类结果的业务含义，也不自行选择澄清或兜底目标。 |
| 标准结果状态映射 | MUST | 工作流正常结束、显式用户交互中断、远端调用或结果解析失败必须分别映射为标准 `COMPLETED`、`INTERRUPTED` 和 `FAILED` 语义；异常断流不得误报为完成。 |
| 结构化用户交互中断 | MUST | Versatile 意图识别工作流显式请求用户输入时，Adapter 必须取得足以形成 FEAT-008 标准用户交互中断的信息，并通过 `AgentExecutionResult.interrupted(UserInputInterrupt)` 返回给 Runtime。 |
| 原工作流恢复 | MUST | Runtime 以 `USER_INTERACTION_RESUME` 恢复执行时，Adapter 必须从现有 `AgentExecutionContext` 取得已校验的用户响应，并结合原会话和工作流续接关联恢复产生中断的原工作流。 |
| 再次中断适配 | MUST | Adapter 恢复原工作流后，必须能够继续转换其完成、失败或再次产生的标准用户交互中断；Task 交互轮次和状态管理遵守 FEAT-008。 |
| 会话与续接关联 | MUST | Adapter 必须使用稳定的 Runtime state/session 语义和工作流提供的续接信息维持会话连续性，保证用户响应恢复到产生当前中断的工作流执行。 |
| 协作式取消 | MUST | Runtime 发起取消时，Adapter 必须复用现有 `cancel` 能力停止继续消费本次结果，并尽力通知远端 Versatile 工作流取消或终止对应执行。 |
| 结构化技术失败 | MUST | 远端调用失败、超时、输入配置缺失、响应字段缺失或类型错误、目标不唯一、结果解析失败以及恢复失败必须映射为可诊断的标准失败结果。 |
| 可观测与审计 | SHOULD | 工作流调用、输入映射、结果提取、中断、恢复、再次中断、取消和失败应形成可关联的观察记录，并关联 tenant、agent、Task/context、conversation、request、trace、结果和耗时。 |
| 敏感信息保护 | SHOULD | 观测与错误信息应按照平台策略对用户输入、消息内容、工作流响应及其他敏感字段执行掩码、截断或禁止落盘。 |

## 3. 外部接口与入口要求

本特性不新增公开类、执行函数、结果类型、中断类型或独立配置入口，完整复用 FEAT-002 已有的 Runtime SDK / SPI、`VersatileAgentRuntimeHandler` 和 `versatile.*` 配置入口，并复用 FEAT-008 已有的用户交互中断与恢复入口。

既有接口的名称、函数签名和基础职责以 FEAT-002、FEAT-008 为准；本特性对输入映射、结果转换、中断恢复、状态、失败和取消的使用语义统一在第五章定义。实现确需增加公开 API 时，必须先通过 version-scope 特性评审明确新增接口事实。

## 4. 场景与用户旅程

| 场景 | 前置条件 | 用户/系统动作 | 期望行为 |
|---|---|---|---|
| 调用一层意图识别工作流 | 一层 Versatile 工作流已独立部署，并由一个 Runtime 实例使用 `VersatileAgentRuntimeHandler` 接入 | 调用方通过标准 Agent 服务入口提交当前用户业务输入 | Adapter 将当前主输入映射为一层 `query`，结合一层自身配置调用工作流；工作流正常完成后返回机器可读的三字段结果。 |
| 分类错误后重新调用一层工作流 | 调用方已经按照业务旅程形成新的调用主输入，并重新调用一层 Agent | Runtime 将本次调用主输入交给当前一层 Adapter | Adapter 按普通调用将当前主输入映射为 `query`，结合当前实例配置执行工作流并返回标准结果；输入来源和重新分类原因不改变 Adapter 行为。 |
| 调用二层意图识别工作流 | 二层 Versatile 工作流已独立部署，并由另一个 Runtime 实例使用相同 Adapter 实现接入 | 直接调用方把一层 `response_content` 作为二层当前主输入，并调用二层 Agent | 二层 Adapter 将当前主输入映射为 `query`，结合该二层工作流自己的配置完成调用并返回三字段结果；Adapter 不重复执行一层工作流。 |
| 匹配成功 | 当前一层或二层工作流正常完成，并返回匹配目标的完整三字段结果 | Adapter 提取工作流结果 | Adapter 将三字段以机器可读形式保留在标准执行结果中，并映射为 `COMPLETED`。 |
| 未匹配或需要澄清 | 客户工作流已为未匹配和需要澄清配置对应的处理工作流，并返回完整三字段结果 | Adapter 提取工作流结果 | Adapter 按与匹配成功相同的技术流程返回 `COMPLETED`，并保留完整三字段供调用方处理；只有工作流显式产生用户交互中断时才进入中断转换。 |
| 意图工作流请求用户交互 | 当前一层或二层工作流执行过程中显式请求用户补充、确认或选择 | 工作流产生可转换为标准交互要求的原生中断 | Adapter 将原生中断转换为 FEAT-008 标准中断；Task 等待、用户响应接收和原执行入口调度由 FEAT-008 处理，Adapter 在恢复调用中续接原工作流。 |
| 多轮用户交互 | 原工作流已通过一次有效用户响应恢复执行 | 恢复后的工作流再次请求用户输入 | Adapter 再次产生标准用户交互中断；每轮 Task 状态和交互关联由 FEAT-008 管理，Adapter 持续负责当前工作流的原生中断与恢复请求转换。 |
| 原生用户交互信息不完整 | 工作流表示需要用户输入，但未提供足以构造标准交互要求或恢复原执行的信息 | Adapter 解析原生中断 | Adapter 返回可诊断的标准失败或 FEAT-002 规定的通用技术中断，不生成缺少有效问题、输入要求或续接关联的 FEAT-008 用户交互请求。 |
| 用户交互恢复失败 | Runtime 已按 FEAT-008 接受并校验用户响应，但原工作流续接信息已失效、缺失或远端恢复调用失败 | Runtime 使用 `USER_INTERACTION_RESUME` 再次调用原 Adapter | Adapter 返回可区分恢复阶段的标准失败，不把恢复失败映射为正常完成，也不创建一次新的工作流执行代替原执行。 |
| 输入或部署配置不完整 | 当前工作流缺少必填的意图配置、输入来源或映射规则 | Runtime 调用当前 Adapter | Adapter 在发起远端调用前返回可诊断的结构化失败，并指出配置读取或输入组装阶段。 |
| 正常结果不满足三字段契约 | 工作流已经结束，但结果字段缺失、类型错误或 `agent_id` 不是唯一值 | Adapter 提取并校验工作流结果 | Adapter 返回可诊断的结构化失败，不输出部分正常结果。 |
| 远端调用、解析或异常断流失败 | 远端工作流不可用、调用超时、响应无法解析，或连接关闭时没有明确 terminal event | Runtime 调用工作流或 Adapter 消费响应 | Adapter 返回可诊断的标准失败或 FEAT-002 规定的通用技术中断，不把异常断流映射为正常完成，也不虚构用户交互问题。 |
| 取消工作流执行 | 当前工作流正在执行或 Task 正处于用户交互等待状态 | 调用方通过标准 Task 取消入口发起取消 | Runtime 调用现有 Handler 的取消入口；Adapter 停止继续消费本次结果，并尽力协作取消远端工作流执行。 |

## 5. 行为语义与边界

### 5.1 核心行为语义

#### 5.1.1 部署与执行语义

- 同一个 `VersatileAgentRuntimeHandler` 实现通过不同 Runtime 实例的配置适配一层和二层意图识别工作流；层级差异不形成两套专用 Adapter 类或执行函数。
- 一个 Runtime 实例只适配一个意图识别工作流，继续遵守 FEAT-002 的单 Agent Runtime 边界。
- 工作流独立部署、标准服务入口和 Agent Card 配置由客户集成与部署流程完成，并遵守 FEAT-001 及相关特性的接入要求。
- 每次 `execute(context)` 只调用当前 Runtime 实例配置的一个工作流。当前工作流结束后，Adapter 返回标准结果供调用方后续处理。

#### 5.1.2 输入契约

所有 Versatile 意图识别工作流调用使用相同的五字段输入契约，字段类型均为 `string`：

| 字段 | 要求 | 输入来源 | 事实语义 |
|---|---|---|---|
| `query` | MUST | `AgentExecutionContext` 中当前调用的主输入 | 当前工作流调用需要处理的主要内容；Adapter 原样映射，不解释其业务来源。 |
| `intents_id` | MUST | 当前 Runtime 实例的开发者或部署配置 | 当前工作流使用的低码意图标识配置信息；Adapter 按字符串传递，不从 Agent Card 推导。 |
| `intent_name` | MUST | 当前 Runtime 实例的开发者或部署配置 | 与当前 `intents_id` 配套的意图名称配置信息；Adapter 按字符串传递，不解释业务含义。 |
| `messages_role` | conditional | 执行上下文或请求 metadata，按当前部署配置映射 | 传递给当前工作流的消息角色信息；来源、必填性和缺省字符串由部署配置声明。 |
| `messages_content` | conditional | 执行上下文或请求 metadata，按当前部署配置映射 | 传递给当前工作流的其他消息内容；来源、必填性和缺省字符串由部署配置声明。 |

输入处理遵守以下语义：

- `query` 是当前调用的主输入。调用方按照具体业务旅程组装该输入，Adapter 对所有意图工作流调用执行相同的映射。
- 部署配置声明为必填的输入无法取得时，Adapter 必须返回结构化输入错误；配置允许缺省时，Adapter 使用配置提供的缺省字符串。
- Adapter 只执行字符串类型、必填性和配置完整性等技术校验，不判断 `intents_id`、`intent_name` 或消息内容在业务上是否正确。
- Adapter 不调用模型生成字段，不推断消息角色，也不拼接、总结或改写会话内容。

#### 5.1.3 正常结果契约

Versatile 意图识别工作流每次正常完成时必须返回以下三个字段：

| 字段 | 类型 | 要求 | 事实语义 |
|---|---|---|---|
| `response_content` | `string` | MUST | 当前工作流返回给调用方后续处理的主要业务内容；字段必须存在，业务是否允许空字符串由客户工作流契约决定。 |
| `intent_id` | `string` | MUST | 客户低码平台生成的下一跳工作流标识，用于业务关联、透传和审计；必须为非空字符串，不作为 Agent Card 查询条件。 |
| `agent_id` | `string` | MUST | 当前 tenant 下与 FEAT-015 逻辑 `agentId` 对应的唯一下一跳目标标识；必须为单个非空字符串。 |

结果处理遵守以下语义：

- Adapter 必须使用 FEAT-002 的结果提取机制取得三个字段，并在现有标准执行结果中以机器可读形式完整保留。
- 当前版本不新增专用结果类；三个字段在现有结果模型中的具体承载位置由下游设计确定，但不得降级为需要解析自然语言才能取得的结果。
- 一个 `intent_id` 可以在客户工作流内部对应多个候选 `agent_id`，但客户工作流必须完成本次业务选择并只返回一个 `agent_id`。Adapter 不执行第二次业务选择。
- Adapter 只校验 `agent_id` 的存在性、字符串类型和唯一性，不查询注册中心验证目标是否存在或可路由。
- 匹配成功、未匹配和需要澄清均属于正常结果。三类结果通过客户预先定义的 `intent_id`、唯一 `agent_id` 和 `response_content` 表达，Adapter 对其执行相同的技术转换并映射为 `COMPLETED`。

#### 5.1.4 用户交互中断与恢复语义

- Versatile 意图识别工作流只有在显式返回用户交互中断，并提供足以构造 FEAT-008 标准用户交互中断和恢复原执行的信息时，Adapter 才能将其转换为 `AgentExecutionResult.interrupted(UserInputInterrupt)`。
- 交互提示、用户输入要求和恢复关联必须能够完整映射到 FEAT-008 契约；Versatile 原生响应字段名称、JSON/SSE 路径和恢复请求格式由下游设计和部署配置确定，不在本文中另行定义。
- Runtime 接受标准中断后的 Task 等待状态、交互轮次、响应校验、幂等和原执行入口调度统一遵守 FEAT-008。
- Runtime 接受有效用户响应后，以 `inputType = USER_INTERACTION_RESUME` 再次调用原 `AgentRuntimeHandler.execute(AgentExecutionContext)`。Adapter 从现有执行上下文取得标准用户响应，并结合稳定的 conversation、state/session 和工作流续接关联恢复原工作流。
- 用户响应的业务有效性由恢复后的 Versatile 意图识别工作流判断。工作流可以完成、失败或再次请求用户交互，Adapter 必须将其重新映射为相应标准结果。
- 恢复后的工作流再次请求用户交互时，Adapter 必须继续转换原生中断；多轮 Task 和交互关联由 FEAT-008 管理。
- 仅发生 HTTP/SSE 连接关闭且没有完整的显式用户交互信息时，Adapter 继续遵守 FEAT-002 的通用异常断流检测语义，但不得虚构交互提示、用户输入要求或 FEAT-008 用户交互请求。

#### 5.1.5 状态、失败、取消与可观测语义

| 场景 | 标准结果 | 事实要求 |
|---|---|---|
| 工作流返回完整三字段正常结果 | `COMPLETED` | 匹配成功、未匹配和需要澄清执行相同的技术完成语义。 |
| 工作流显式请求用户交互 | `INTERRUPTED` | 必须携带可被 FEAT-008 接受的标准用户交互中断；后续 Task 行为遵守 FEAT-008。 |
| 远端 HTTP/SSE 调用失败或超时 | `FAILED` | 必须保留可诊断的远端错误、状态或超时分类。 |
| 输入配置缺失或输入映射失败 | `FAILED` | 必须指出失败发生在配置读取或请求组装阶段。 |
| 正常结果字段缺失、类型错误或 `agent_id` 不唯一 | `FAILED` | 不得输出可被调用方误认为完整正常结果的部分数据。 |
| 用户交互恢复上下文不可用或远端恢复失败 | `FAILED` | 必须遵守 FEAT-008 的恢复失败语义，并保留可区分的失败阶段。 |
| HTTP/SSE 无明确 terminal event 即关闭 | 通用技术中断或 `FAILED` | 按 FEAT-002 判断，不得映射为 `COMPLETED`，也不得在缺少有效交互描述时生成用户问题。 |
| Runtime 发起取消 | 取消语义 | Adapter 必须停止继续消费本次结果并尽力通知远端工作流；最终 Task 状态由 Runtime 统一推进。 |

- Adapter 不把远端技术失败转换为正常 `intent_id`，也不生成匹配、澄清或未匹配业务兜底结果。
- 工作流调用、结果、中断和恢复必须使用 Runtime 提供的 tenant、Task/context、conversation、request、trace 和 correlation 语义建立观测关联。
- 日志、轨迹和错误表面不得无控制地记录完整用户输入、`messages_content`、工作流响应或用户交互响应。

### 5.2 显式边界与不承诺项

| 边界 | 事实归属 |
|---|---|
| 工作流部署与发布 | 客户集成与部署流程负责一层、二层工作流的独立部署、Runtime 实例配置、标准服务入口和 Agent Card；本 Adapter 复用同一实现，单个 Runtime 实例只调用当前配置的一个工作流。 |
| 结构化结果消费 | Adapter 返回机器可读的 `response_content`、`intent_id` 和唯一 `agent_id`；调用方如何使用结果，以及 Agent Card 查询、实例路由和 A2A 调用，分别由对应 Runtime、注册发现和远程调用特性定义。 |
| `agent_id` 有效性 | Adapter 负责基本结构校验；目标是否已注册、是否满足版本和安全约束、是否有可用运行实例由目标发现与调用链路判断。 |
| 业务匹配与兜底 | 匹配、未匹配、需要澄清、候选 `agent_id` 选择和业务失败兜底由客户意图识别工作流或其调用方负责；Adapter 只转换工作流已经给出的正常结果或技术失败。 |
| 用户交互 Task 管理 | Adapter 负责原生中断与恢复请求适配；Task 状态、交互轮次、用户响应结构校验、幂等和原执行入口调度由 FEAT-008 Runtime 能力负责。 |
| Agent Bus 用户消息路由 | 下游 Task 面向用户的消息投影和用户响应直达真实 Task owner 由对应 Agent Bus 与 Gateway 特性定义，不改变本 Adapter 的中断转换和恢复职责。 |
| Versatile 原生协议细节 | 原生中断事件、字段路径、恢复请求格式和具体 `versatile.*` 配置项由 L2 设计与开发实现确定，并必须满足本文行为契约。 |
| 外部类型与入口 | 本特性复用 FEAT-002、FEAT-008 已有类型和入口，不新增专用 Handler、执行函数、结果类或中断类。若实现确需新增公开 API，必须先更新 version-scope 事实要求。 |

## 6. 对下游设计与实现的约束

- L2 设计必须把本文作为 Versatile 意图识别工作流适配能力的事实来源，并保持单 Agent Runtime 和单实例只调用当前配置工作流的边界。
- 实现必须复用 `VersatileAgentRuntimeHandler`、`AgentRuntimeHandler.execute(AgentExecutionContext)`、`AgentExecutionResult` 和 FEAT-008 `UserInputInterrupt`，不得未经特性评审另建职责重复的公开类、执行函数、结果类型或中断类型。
- L2 设计必须在现有 `versatile.*` 配置体系内明确工作流 URL、输入映射、结果提取、中断识别和恢复请求映射；本文不指定 Versatile 原生字段路径和具体配置项名称。
- L2 设计必须使 `messages_role` 和 `messages_content` 的来源、必填性和缺省字符串可由部署配置明确，并在开发者指南中说明权限、长度、编码和敏感信息处理规则。
- 实现必须保持三个正常结果字段的机器可读性，不得把调用方需要消费的 `intent_id` 或 `agent_id` 仅写入自然语言文本。
- 输入输出测试必须覆盖不同调用场景下当前主输入到 `query` 的统一映射、独立工作流配置、消息字段必填与缺省策略、三字段完整结果、匹配成功、未匹配、需要澄清、空 `response_content`、字段缺失、类型错误和多目标 `agent_id`。
- 中断恢复测试必须覆盖原生中断到标准 `UserInputInterrupt` 的转换、有效恢复输入映射、恢复后再次产生中断、恢复上下文不可用、远端恢复失败，以及无 terminal event 的异常断流不得伪造用户交互请求；Task 投影、交互轮次和等待期间取消按照 FEAT-008 验证。
- 失败与可观测测试必须覆盖远端 HTTP/SSE 错误、超时、结果解析失败、取消、敏感字段掩码以及调用、中断和恢复的 trace/correlation 关联。
- 若未来需要在单个 Adapter 实例内编排一层和二层工作流、访问注册中心、选择候选目标或承担 Agent Bus 用户消息路由，必须先由相应 version-scope 特性明确职责与外部契约，再进入 L2 和实现。

## 7. 关联文档

- `../spring-ai-ascend/architecture/L0-Top-Level-Design/boundaries.md`
- `../spring-ai-ascend/architecture/L0-Top-Level-Design/glossary.md`
- `../spring-ai-ascend/architecture/L1-High-Level-Design/agent-runtime/README.md`
- `../spring-ai-ascend/architecture/L1-High-Level-Design/agent-runtime/logical.md`
- `../spring-ai-ascend/architecture/L1-High-Level-Design/agent-runtime/process.md`
- `../spring-ai-ascend/architecture/L1-High-Level-Design/agent-runtime/scenarios.md`
- `../spring-ai-ascend/architecture/L1-High-Level-Design/agent-runtime/spi-appendix.md`
- `../spring-ai-ascend/version-scope/FEAT-001-standardized-agent-service-entrypoint.md`
- `../spring-ai-ascend/version-scope/FEAT-002-heterogeneous-agent-framework-compatibility.md`
- `../spring-ai-ascend/version-scope/Feat-008-user-interaction-interrupt-response.md`
- `../spring-ai-ascend/version-scope/Feat-015-agent-card-registration-and-discovery.md`
- `../spring-ai-ascend/version-scope/DFX-001-trajectory-observability.md`
