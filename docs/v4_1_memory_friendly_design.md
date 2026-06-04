# V4.1 Memory Friendly 设计文档

## 1. 文件范围

V4.1 新增独立工作流文件：

- `after_sales_agent_v4_1_memory_friendly.yml`

V4.1 不覆盖、不修改以下文件：

- `after_sales_agent_demo.yml`
- `after_sales_agent_v4_policy_risk_guarded.yml`
- `DSL/Agent工具调用.yml`

本版不接真实 API、不接数据库、不做真实 RAG、不做多 Agent、不做前端、不做 Docker。

## 2. 本版解决的问题

Dify UI 测试中发现两个体验问题：

1. 用户先说“我要退款”，系统追问订单号后，下一轮只输入 `OD202605240001` 时，基础 V4 无法继承上一轮退款意图。
2. 用户只输入 `OD202605240001` 时，基础 V4 会进入 fallback，回复过于机械；更合理的是确认收到订单号，并追问用户想查订单、退款还是投诉。

另外修复一个逻辑隐患：

- 投诉路径遇到格式合法但不存在的订单号时，不再创建投诉工单，而是提示用户核对订单号。

## 3. 会话变量设计

V4.1 在 `workflow.conversation_variables` 中新增 6 个短期会话变量：

| 变量 | 用途 |
| --- | --- |
| `last_order_id` | 最近一次格式合法的订单号，用于后续省略订单号的投诉场景 |
| `pending_intent` | 缺少订单号时暂存用户意图，例如 `refund` / `complaint` |
| `pending_reason` | 缺少订单号时暂存退款原因 |
| `pending_complaint_type` | 缺少订单号时暂存投诉类型 |
| `pending_description` | 缺少订单号时暂存用户描述 |
| `last_route` | 最近一次路由，便于观测调试 |

实现方式：

1. `validate_and_route` 读取上述 conversation variables。
2. `validate_and_route` 输出 `new_last_order_id`、`new_pending_intent`、`new_pending_reason`、`new_pending_complaint_type`、`new_pending_description`、`new_last_route`。
3. 通过 6 个 `assigner` 节点写回 conversation variables。
4. 写回完成后再进入 `route_by_intent` 条件分支。

## 4. 记忆行为规则

### 4.1 缺少订单号时记录 pending

当用户表达 `refund` / `complaint` / `query_order` 但缺少订单号时：

- route 仍为 `missing_order`
- 写入：
  - `pending_intent`
  - `pending_reason`
  - `pending_complaint_type`
  - `pending_description`
  - `last_route = missing_order`

### 4.2 下一轮只输入合法订单号时继承 pending_intent

如果用户只输入 `OD + 12 位数字`，且 `pending_intent` 存在：

- 继承 `pending_intent`
- 继承 pending slot 信息
- 进入对应业务路径，例如 refund 或 complaint
- 处理后清空 pending 信息
- 更新 `last_order_id`

### 4.3 后续投诉可复用 last_order_id

如果用户后续表达“物流太慢了，我要投诉”但没有明确订单号：

- 如果 `last_order_id` 存在且格式合法，则复用该订单号进入 complaint 路径
- 如果没有 `last_order_id`，仍然进入 `missing_order` 追问订单号

### 4.4 单独输入订单号时进入 order_only

如果用户只输入合法订单号，且没有 `pending_intent`：

- route = `order_only`
- 不调用工具
- 回复：

```text
我已收到订单号 ODxxxx。请问你想查询订单状态、申请退款，还是提交投诉？
```

## 5. Workflow 结构变化

V4.1 相比 V4 新增：

- conversation variables
- 6 个 assigner 节点
- `order_only` route
- `answer_order_only` 回复节点

主链路变为：

```text
start
  → agent_intent
  → validate_and_route
  → assign_last_order_id
  → assign_pending_intent
  → assign_pending_reason
  → assign_pending_complaint_type
  → assign_pending_description
  → assign_last_route
  → route_by_intent
```

`refund` / `complaint` 路径继续保留 V4 的：

```text
policy_lookup → calculate_risk_score → draft_v4_reply → answer_verifier → select_verified_answer → answer_v4_final
```

## 6. fallback 文案优化

V4.1 将 fallback 改为更自然的客服风格：

```text
你好呀，我是电商售后客服 Demo，可以帮你查询订单、申请退款或提交投诉。你可以直接告诉我诉求；如果已经有订单号，也可以一起发给我。
```

本版没有继续拆分 `greeting` / `capability_question` / `unknown`，以控制 DSL 复杂度。

## 7. 投诉订单不存在修复

`tool_create_complaint_ticket` 新增 `found` 输出。

如果 `order_id` 不在模拟订单数据中：

```json
{
  "found": "false",
  "ticket_id": "",
  "status": "未创建",
  "message": "没有查询到订单 {order_id}，暂时无法创建投诉工单。请核对订单号后再提交投诉。"
}
```

如果订单存在，则正常创建工单。

`draft_v4_reply` 也同步保护：complaint 路径如发现订单不存在，不生成工单号，不输出“已创建投诉工单”。

## 8. 兼容性说明

Dify conversation variables 和 assigner 节点在本仓库已有示例中使用过。V4.1 采用相同 DSL 结构实现短期会话记忆。

如果目标 Dify 版本导入 assigner 节点或 conversation variables 存在兼容性问题，最小可运行替代方案是：

1. 保留 `validate_and_route` 的记忆逻辑输出字段。
2. 暂时移除 assigner 链路。
3. 在 Dify UI 或外部调用层保存 `pending_intent` / `last_order_id`，并作为输入变量传入。

但首选方案仍是当前 DSL 内置 conversation variables。

## 9. 保持不变的能力

V4.1 继续保留：

- Agent FunctionCalling 参数抽取
- validate_and_route 订单号校验
- route_by_intent 条件分支
- query_order
- check_refund
- create_complaint_ticket
- policy_lookup
- calculate_risk_score
- answer_verifier
- 原 8 条基础测试
- V4 风险 / 政策 / 校验测试
