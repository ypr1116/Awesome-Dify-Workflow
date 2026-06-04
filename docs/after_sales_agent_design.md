# 电商售后客服 Agent Demo 设计说明

## 1. 文件与改造范围

本次新增改造版工作流文件：

- `after_sales_agent_demo.yml`

本次不覆盖、不删除、不修改原始工作流文件：

- `DSL/Agent工具调用.yml`

该 Demo 只用于学习 Dify 中的 Agent、模拟 Tool Calling、Workflow 条件分支和兜底规则，不连接真实 API、不添加真实数据库、不引入 RAG、不做多 Agent、不做前端和 Docker。


## 2. 与原始工作流的对应关系

本 Demo 基于 `DSL/Agent工具调用.yml` 的最小链路思路改造，但输出为独立文件 `after_sales_agent_demo.yml`：

| 原始工作流能力 | 改造后设计 |
| --- | --- |
| `开始` 节点接收 `sys.query` | 保留自然语言输入方式，不新增表单依赖 |
| 单个 `Agent` 节点负责理解用户问题 | 保留一个 Agent 节点，专注售后意图识别与参数抽取 |
| Agent 内挂载通用工具 | 改为 Workflow 代码节点模拟 3 个售后工具，避免真实 API / 数据库依赖 |
| `直接回复` 输出 Agent 文本 | 改为条件分支后的多路回复，体现缺失订单号、格式错误、工具结果和兜底规则 |

说明：这里的“模拟工具”使用 Dify Code 节点实现，目的是在不接入真实 API 的前提下演示 Tool Calling 的输入、输出和调用边界；生产环境可替换为真实 HTTP 工具、工作流工具或插件工具。

## 3. 业务目标

将原始“Agent 工具调用”示例改造成“电商售后客服 Agent Demo”。用户可以向售后客服咨询：

1. 查订单
2. 申请退款
3. 投诉
4. 闲聊 / 其他咨询

工作流重点演示：

- Agent 负责意图识别和参数抽取。
- Workflow 负责确定性规则校验与条件分支。
- 代码节点模拟 3 个售后工具。
- 订单号缺失或格式错误时不进入工具分支。
- 退款金额超过 500 元时触发人工审核提示。
- 非售后问题进入兜底回复。

## 4. 总体流程

```text
用户输入
  → Agent：识别售后意图与参数
  → Code：校验订单号并生成路由
  → If/Else：按路由分支
      ├─ missing_order  → 追问订单号
      ├─ invalid_order  → 提示订单号格式错误
      ├─ query_order    → 工具：query_order → 回复订单查询结果
      ├─ refund         → 工具：check_refund → 回复退款处理结果
      ├─ complaint      → 工具：create_complaint_ticket → 回复投诉工单结果
      └─ fallback       → 兜底回复
```

## 5. 节点设计

### 5.1 开始节点

节点 ID：`start`

作用：

- 接收用户自然语言输入。
- 不额外配置表单变量。
- 用户消息通过 `{{#sys.query#}}` 传给 Agent 节点。

### 5.2 Agent 节点：识别售后意图与参数

节点 ID：`agent_intent`

作用：

- 基于原始 `Agent工具调用.yml` 的 Agent 思路改造。
- 使用 `FunctionCalling` Agent 策略。
- 模型配置为 `gpt-4o-mini`。
- 只负责识别意图和提取参数，不直接处理业务结果。

Agent 输出 JSON：

```json
{
  "intent": "query_order|refund|complaint|chitchat",
  "order_id": "",
  "reason": "",
  "complaint_type": "",
  "description": ""
}
```

支持意图：

| intent | 含义 |
| --- | --- |
| `query_order` | 查订单、查物流、查订单状态 |
| `refund` | 申请退款、退钱、退货退款 |
| `complaint` | 投诉物流、投诉服务或商品问题 |
| `chitchat` | 闲聊或其他咨询 |

### 5.3 规则校验节点：校验订单号并生成路由

节点 ID：`validate_and_route`

作用：

- 解析 Agent 输出。
- 必要时用关键词做兜底意图识别，避免 Agent 输出不是合法 JSON 时流程中断。
- 判断当前意图是否需要订单号。
- 判断订单号是否缺失。
- 判断订单号格式是否合法。
- 生成后续条件分支使用的 `route`。

订单号规则：

```regex
^OD\d{12}$
```

合法示例：

```text
OD202605240001
```

关键规则：

- 查订单、退款、投诉都需要订单号。
- 如果需要订单号但用户没有提供，则进入 `missing_order`。
- 如果订单号格式不合法，则进入 `invalid_order`。
- 订单号缺失或格式错误时，不允许调用订单、退款或投诉工具。

路由值：

| route | 含义 |
| --- | --- |
| `missing_order` | 需要订单号但未提供 |
| `invalid_order` | 订单号格式不符合 `^OD\d{12}$` |
| `query_order` | 调用订单查询模拟工具 |
| `refund` | 调用退款检查模拟工具 |
| `complaint` | 调用投诉建单模拟工具 |
| `fallback` | 闲聊 / 其他咨询兜底 |

### 5.4 条件分支节点

节点 ID：`route_by_intent`

作用：根据 `validate_and_route.route` 分流：

- `missing_order`：直接追问订单号。
- `invalid_order`：直接提示正确订单号格式。
- `query_order`：调用 `query_order` 模拟工具。
- `refund`：调用 `check_refund` 模拟工具。
- `complaint`：调用 `create_complaint_ticket` 模拟工具。
- 其他情况：兜底回复。

## 6. 模拟工具设计

本 Demo 不连接真实 API 或数据库，模拟工具由 Dify Code 节点实现，数据硬编码在工作流内部。

### 6.1 模拟订单数据

| order_id | status | amount | product | can_refund |
| --- | --- | ---: | --- | --- |
| `OD202605240001` | 已发货 | 299 | 无线蓝牙耳机 | true |
| `OD202605240002` | 已签收 | 129 | 手机壳 | true |
| `OD202605240003` | 已签收 | 899 | 机械键盘 | true |

### 6.2 query_order

节点 ID：`tool_query_order`

输入：

- `order_id`

输出：

- `status`
- `amount`
- `product`
- `can_refund`
- `reply`

功能：根据订单号返回订单状态、金额、商品名和是否可退款。

### 6.3 check_refund

节点 ID：`tool_check_refund`

输入：

- `order_id`
- `reason`

输出：

- `amount`
- `can_refund`
- `need_human_check`
- `message`
- `reply`

退款规则：

- 当 `amount <= 500` 时，返回自动退款处理结果。
- 当 `amount > 500` 时，`need_human_check = true`，并返回人工审核提示：

```text
该订单退款金额超过 500 元，需要人工客服审核。我已为你提交人工审核，请等待客服确认。
```

### 6.4 create_complaint_ticket

节点 ID：`tool_create_complaint_ticket`

输入：

- `order_id`
- `complaint_type`
- `description`

输出：

- `ticket_id`
- `status`
- `message`
- `reply`

功能：为合法订单创建模拟投诉工单。工单号按订单号后四位生成，例如 `OD202605240001` 对应 `CP0001001`。

## 7. 兜底规则

### 7.1 缺少订单号

当用户表达查订单、退款或投诉，但没有提供订单号时，工作流直接回复：

```text
需要先确认订单号才能继续处理。请提供订单号，格式为 OD + 12 位数字，例如 OD202605240001。
```

不会调用任何模拟工具。

### 7.2 订单号格式错误

当用户提供了候选订单号但不符合 `^OD\d{12}$` 时，工作流直接回复正确格式提示。

不会调用任何模拟工具。

### 7.3 闲聊 / 其他咨询

当意图为 `chitchat` 或无法归类时，进入兜底回复，说明当前 Demo 能力范围：查订单、申请退款、提交投诉。

## 8. 设计取舍

- 使用一个 Agent 节点做自然语言理解，避免多 Agent。
- 使用 Code 节点模拟工具，避免真实 API、数据库和外部依赖。
- 使用 Workflow 条件分支实现强规则，保证订单号缺失或格式错误时不会误调用工具。
- 工具节点直接生成 `reply`，让 Demo 更容易导入和学习；生产环境可进一步拆分为“工具返回结构化结果 + LLM 统一客服话术生成”。
