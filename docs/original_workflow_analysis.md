# `Agent工具调用.yml` 原始工作流结构分析

> 分析对象：`DSL/Agent工具调用.yml`  
> 分析目的：为后续改造成“电商售后客服 Agent Demo”提供只读结构基线。  
> 本文档仅新增分析文档；未修改原始 YML 文件。

## 1. 当前目录结构概览

仓库顶层目录主要包含：

```text
.
├── DSL/                 # Dify DSL / workflow yml 文件集中目录
│   └── 图文知识库/
│       └── 知识库内容/
├── assets/
│   └── partners/
│       └── logos/
├── images/              # 示例图片资源
├── snapshots/           # 示例截图资源
├── LICENSE
├── README.md
├── README_EN.md
└── chat_history.md
```

与本次目标最相关的是 `DSL/` 目录，其中包含多个 `.yml` 工作流文件；目标文件已定位为：

```text
DSL/Agent工具调用.yml
```

## 2. 目标工作流文件定位

当前仓库中存在大量 Dify workflow / app DSL 文件，例如：

- `DSL/Agent工具调用.yml`
- `DSL/AgentFlow.yml`
- `DSL/Demo-tod_agent.yml`
- `DSL/MCP.yml`
- `DSL/MCP-amap.yml`
- `DSL/搜索大师.yml`
- `DSL/旅行Demo.yml`

其中 `DSL/Agent工具调用.yml` 的应用名为 `Agent工具调用`，模式为 `advanced-chat`，版本为 `0.1.5`。

## 3. 应用与 Workflow 基本信息

`Agent工具调用.yml` 顶层结构如下：

| 字段 | 当前值 | 说明 |
| --- | --- | --- |
| `kind` | `app` | Dify 应用 DSL |
| `version` | `0.1.5` | DSL 版本 |
| `app.name` | `Agent工具调用` | 应用名称 |
| `app.mode` | `advanced-chat` | 高级聊天应用 |
| `workflow.conversation_variables` | `[]` | 未定义会话变量 |
| `workflow.environment_variables` | `[]` | 未定义环境变量 |

工作流功能配置要点：

- 文件上传能力存在配置项，但 `enabled: false`，当前未启用。
- `retriever_resource.enabled: true`，但图中没有知识检索节点。
- 敏感词规避、语音转文字、文本转语音、建议问题等均未启用或为空。

## 4. Graph 节点结构

当前 graph 共有 **3 个节点**：

| 顺序 | 节点 ID | 标题 | 类型 | 作用 |
| --- | --- | --- | --- | --- |
| 1 | `1739781961838` | `开始` | `start` | 接收用户输入，当前未配置额外变量 |
| 2 | `1739781971571` | `Agent` | `agent` | 使用 Function Calling 策略，根据用户问题调用工具并生成回复 |
| 3 | `answer` | `直接回复` | `answer` | 将 Agent 输出文本直接返回给用户 |

### 4.1 开始节点

开始节点配置非常简单：

- 标题：`开始`
- 类型：`start`
- `variables: []`

这表示工作流没有在开始节点声明额外表单输入变量；用户消息通过系统变量 `sys.query` 进入后续 Agent 节点。

### 4.2 Agent 节点

Agent 节点是当前工作流的核心节点。

基础配置：

| 配置项 | 当前值 |
| --- | --- |
| 节点 ID | `1739781971571` |
| 标题 | `Agent` |
| 类型 | `agent` |
| Agent 策略 | `FunctionCalling` / `function_calling` |
| 策略提供方 | `langgenius/agent/agent` |
| 插件标识 | `langgenius/agent:0.0.4@5eb03c08764cc37249f9ef18b89903a99493f6d02c4d5b8ffb40b9f7ef4e865c` |

Agent 参数：

| 参数 | 当前值 | 说明 |
| --- | --- | --- |
| `instruction` | `根据用户的需求，调用不同工具，回复用的内容。` | 给 Agent 的系统指令，非常通用 |
| `query` | `{{#sys.query#}}` | 直接使用用户当前问题作为 Agent 输入 |
| `model.provider` | `langgenius/openai/openai` | OpenAI 模型提供方 |
| `model.model` | `gpt-4o-mini` | 使用的聊天模型 |
| `model.mode` | `chat` | 聊天模型模式 |
| `completion_params` | `{}` | 未显式设置温度、最大 token 等参数 |

### 4.3 工具配置

当前 Agent 节点的工具列表中配置了 **3 个内置工具**，其中 **2 个启用**、**1 个禁用**。

#### 工具 1：获取当前时间

| 配置项 | 当前值 |
| --- | --- |
| `enabled` | `true` |
| `provider_name` | `time` |
| `tool_name` | `current_time` |
| `tool_label` | `获取当前时间` |
| `type` | `builtin` |

参数与设置：

- `parameters: {}`：没有 LLM 自动填写参数。
- `settings.format.value: %Y-%m-%d %H:%M:%S`
- `settings.timezone.value: UTC`
- schema 中 `format` 与 `timezone` 均为 `form` 参数，说明它们由工具配置固定，而不是由模型在运行时根据用户问题填写。

用途：当用户询问当前日期、时间或需要基于当前时间回答时，Agent 可以调用该工具获取 UTC 当前时间。

#### 工具 2：DuckDuckGo 搜索

| 配置项 | 当前值 |
| --- | --- |
| `enabled` | `true` |
| `provider_name` | `langgenius/duckduckgo/duckduckgo` |
| `tool_name` | `ddgo_search` |
| `tool_label` | `DuckDuckGo 搜索` |
| `type` | `builtin` |

参数与设置：

- `parameters.query.auto: 1`：搜索 query 由 LLM 自动从用户问题中生成。
- `settings.max_results.value: 5`：最多返回 5 条搜索结果。
- `settings.require_summary.value: 0`：工具本身不要求对搜索结果进行总结；总结/整合主要由 Agent 模型完成。
- schema 中 `query` 的 `form` 为 `llm` 且 `required: true`，说明该参数必须由 Agent 在工具调用时提供。

用途：当用户问题需要外部实时信息或网页检索时，Agent 可以生成搜索关键词并调用 DuckDuckGo 搜索。

#### 工具 3：天气查询

| 配置项 | 当前值 |
| --- | --- |
| `enabled` | `false` |
| `provider_name` | `langgenius/openweather/openweather` |
| `tool_name` | `weather` |
| `tool_label` | `天气查询` |
| `type` | `builtin` |

参数与设置：

- `parameters.city.auto: 1`：城市由 LLM 从用户问题中抽取。
- `settings.lang.value: zh_cn`：中文返回。
- `settings.units.value: metric`：摄氏度单位。
- 但由于 `enabled: false`，该工具当前不会被 Agent 实际调用。

用途：该工具配置保留在 DSL 中，但在当前工作流运行时不可用。若启用，可支持天气查询。

### 4.4 输出节点

输出节点标题为 `直接回复`，类型为 `answer`。

关键配置：

```yaml
answer: '{{#1739781971571.text#}}'
```

这表示最终回复直接取自 Agent 节点的 `text` 输出，不再经过额外格式化、条件判断、模板渲染或二次 LLM 处理。

## 5. 连接关系

当前 graph 共有 **2 条边**，形成一条线性链路：

```text
开始(start)
  → Agent(agent)
  → 直接回复(answer)
```

详细连接关系：

| 边 ID | Source | Source 类型 | Target | Target 类型 |
| --- | --- | --- | --- | --- |
| `1739781961838-source-1739781971571-target` | `1739781961838` | `start` | `1739781971571` | `agent` |
| `1739781971571-source-answer-target` | `1739781971571` | `agent` | `answer` | `answer` |

该工作流没有条件分支、循环、迭代、变量赋值、代码节点、HTTP 请求节点、知识库检索节点或专门的独立工具节点；工具能力全部挂载在 Agent 节点内部。

## 6. 当前 Agent 工具调用流程总结

当前工作流完成 Agent 工具调用的方式如下：

1. 用户在高级聊天应用中输入问题。
2. 开始节点不做额外变量收集，用户问题通过系统变量 `{{#sys.query#}}` 传给 Agent 节点。
3. Agent 节点使用 `gpt-4o-mini`，并采用 Dify Agent 的 `FunctionCalling` 策略。
4. Agent 根据通用 instruction 判断是否需要调用工具：
   - 如果问题涉及当前时间，可调用 `current_time`。
   - 如果问题需要联网检索，可调用 `ddgo_search`，并由模型自动生成 `query` 参数。
   - 天气工具虽然已配置，但未启用，因此不会参与实际调用。
5. 工具调用结果回到 Agent，由 Agent 组织自然语言回答。
6. `直接回复` 节点读取 `{{#1739781971571.text#}}` 并返回给用户。

## 7. 对后续改造成“电商售后客服 Agent Demo”的结构启示

从当前结构看，它是一个最小可用的 Agent 工具调用 Demo，适合作为改造基础，但要变成电商售后场景，需要重点调整 Agent 节点中的指令、工具列表和可能的输入/输出结构：

- **开始节点**：当前没有额外变量。后续可考虑增加订单号、手机号后四位、问题类型等输入变量；也可以继续使用自然语言输入，由 Agent 自行抽取。
- **Agent 指令**：当前 instruction 过于通用。后续应替换为电商售后客服角色设定、服务边界、话术规范、核验规则、退款/退货/物流查询流程等。
- **工具配置**：当前工具是时间、搜索、禁用天气。后续应替换或新增为订单查询、物流查询、退款政策查询、退货申请、工单创建等售后工具。
- **输出节点**：当前直接输出 Agent 文本。若需要统一客服格式，可保留直接回复；若需要结构化结果，可增加模板或变量处理节点。
- **连接关系**：当前线性结构简单清晰。初版售后 Agent Demo 可以继续保持 `开始 → Agent → 直接回复`，先把业务能力集中到 Agent 与工具定义中。

## 8. 只读检查结论

- 原始目标文件已定位：`DSL/Agent工具调用.yml`。
- 原始工作流未在本次分析中修改。
- 当前工作流是一个三节点线性 Agent Demo。
- Agent 通过 Function Calling 策略在节点内部调用工具。
- 当前启用工具为“获取当前时间”和“DuckDuckGo 搜索”，天气工具配置存在但禁用。
- 最终输出直接来自 Agent 节点文本结果。
