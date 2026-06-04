# V4 Policy Risk Guarded 设计文档

## 1. 文件范围

V4 新增独立工作流文件：

- `after_sales_agent_v4_policy_risk_guarded.yml`

V4 不覆盖、不修改以下基础版或原始文件：

- `after_sales_agent_demo.yml`
- `DSL/Agent工具调用.yml`

本版仍然不接真实 API、不接数据库、不绑定真实 Dify Knowledge Retrieval、不做多 Agent、不做前端、不做 Docker。

## 2. 核心设计决策

V4 **保留 Agent 节点**，不把 `agent_intent` 改成普通 LLM 节点。

原因：当前 Agent 使用 FunctionCalling 策略，更适合作为学习阶段的 Parameter Extractor / Query Understanding 节点。V4 只增强 Agent 的参数抽取字段，并把确定性规则继续放在 Workflow Code 与条件分支中。

新增能力：

1. 售后政策轻量检索：`policy_lookup`
2. 风险评分：`calculate_risk_score`
3. 输出校验：`answer_verifier`

接入范围：

- 已接入：`refund` 路径、`complaint` 路径
- 保持基础版：`query_order`、`missing_order`、`invalid_order`、`fallback`

## 3. V4 主流程

```text
start
  → agent_intent
  → validate_and_route
  → route_by_intent
      ├─ query_order   → tool_query_order → answer_query_order
      ├─ missing_order → answer_missing_order
      ├─ invalid_order → answer_invalid_order
      ├─ fallback      → answer_fallback
      ├─ refund        → tool_check_refund
      │                   → policy_lookup
      │                   → calculate_risk_score
      │                   → draft_v4_reply
      │                   → answer_verifier
      │                   → select_verified_answer
      │                   → answer_v4_final
      └─ complaint     → tool_create_complaint_ticket
                          → policy_lookup
                          → calculate_risk_score
                          → draft_v4_reply
                          → answer_verifier
                          → select_verified_answer
                          → answer_v4_final
```

## 4. Agent 参数抽取增强

`agent_intent` 仍是 Agent 节点，仍使用 FunctionCalling 策略。V4 要求它输出以下 JSON：

```json
{
  "intent": "query_order|refund|complaint|chitchat",
  "order_id": "",
  "reason": "",
  "complaint_type": "",
  "description": "",
  "user_emotion": "normal|angry|urgent",
  "requested_action": "query|refund|complaint|consult|compensation|other",
  "compensation_requested": "true|false",
  "policy_question": "true|false"
}
```

字段用途：

| 字段 | 用途 |
| --- | --- |
| `user_emotion` | 风险评分：愤怒 / 紧急用户加分 |
| `requested_action` | 辅助理解真实诉求，例如咨询、退款、赔偿 |
| `compensation_requested` | 识别赔偿、补偿、退 1000 元等高风险诉求 |
| `policy_question` | 识别“能不能退、政策是什么、七天无理由”等政策咨询 |

## 5. validate_and_route 兼容策略

V4 的 `validate_and_route` 兼容基础版字段和新增字段：

- 订单号规则仍是 `^OD\d{12}$`
- 查订单、退款、投诉仍需要合法订单号
- 缺少订单号或订单号格式错误时仍不允许调用模拟工具
- 原 8 条基础测试必须继续通过

该节点优先解析 Agent 输出；如果 Agent 输出为空或非 JSON，会用原始 `sys.query` 做有限兜底：

- 兜底识别 `intent`
- 兜底识别 `user_emotion`
- 兜底识别 `requested_action`
- 兜底识别 `compensation_requested`
- 兜底识别 `policy_question`
- 从文本兜底提取合法订单号

## 6. policy_lookup：售后政策轻量检索

`policy_lookup` 是 Code 节点，不接真实知识库。它硬编码 6 条售后政策，并按关键词、intent、complaint_type、订单状态和 reason 轻量匹配。

硬编码政策：

1. 已发货订单不能立即退款，需要等待签收或拒收。
2. 已签收订单 7 天内可申请无理由退货。
3. 退款金额超过 500 元需要人工审核。
4. 食品、定制商品、虚拟商品不支持无理由退款。
5. 物流超过 72 小时未更新，可以创建物流投诉工单。
6. 用户情绪强烈时，需要优先安抚并升级人工客服。

输出：

```json
{
  "matched_policies": "...",
  "policy_source": "static_after_sales_policy"
}
```

后续可将该节点替换为 Dify Knowledge Retrieval 节点，但 V4 第一版使用静态政策，优先保证可导入、可运行、可稳定测试。

## 7. calculate_risk_score：风险评分

`calculate_risk_score` 是 Code 节点，基于用户原问题、Agent/route 抽取结果、订单上下文和政策命中结果计算风险分。

评分规则：

| 条件 | 分值 |
| --- | ---: |
| `amount > 500` | +40 |
| `user_emotion = angry` | +20 |
| `user_emotion = urgent` | +15 |
| `complaint_type` 包含“物流” | +10 |
| `complaint_type` 或用户问题包含“质量” | +20 |
| `compensation_requested = true` | +20 |
| 用户表达“赔偿、补偿、投诉威胁、不退就投诉” | +20 |
| 订单状态为“已发货”且用户申请退款 | +10 |

风险等级：

| 分数 | 等级 |
| --- | --- |
| 0–29 | `low` |
| 30–59 | `medium` |
| 60+ | `high` |

输出：

```json
{
  "risk_score": "0",
  "risk_level": "low|medium|high",
  "risk_reasons": "..."
}
```

## 8. answer_verifier：输出校验

`answer_verifier` 是新增 LLM 节点，不替代 Agent 节点。它只在 refund / complaint 路径执行。

输入包含：

- 用户原问题
- Agent / route 抽取字段
- policy_lookup 命中的政策
- calculate_risk_score 输出
- 初版客服回复 `draft_reply`

输出 JSON：

```json
{
  "pass": true,
  "issues": [],
  "revised_answer": ""
}
```

校验规则：

1. 不能编造订单信息。
2. 不能和工具 / 政策 / 风险结果矛盾。
3. 不能承诺工具结果没有返回的退款成功。
4. 高风险场景不能直接说“已退款成功”。
5. 订单号缺失或格式错误时不能调用工具。
6. 如涉及退款 / 投诉政策，应结合 `matched_policies`。
7. 回复语气要有客服安抚感。

`select_verified_answer` 会解析 verifier 输出：

- `pass=true`：最终回复使用 `draft_reply`
- `pass=false`：最终回复使用 `revised_answer`

## 9. 取舍说明

为控制复杂度，V4 第一版只把 `policy_lookup`、`calculate_risk_score`、`answer_verifier` 接入 refund 和 complaint 两条路径。

原因：

- 缺失订单号、错误订单号、闲聊兜底不应该调用工具或 verifier。
- query_order 路径本身风险较低，保持基础版更利于回归测试。
- refund / complaint 是最容易出现政策争议、风险承诺和用户情绪问题的路径，优先接入 V4 能力。

## 10. 导入 Dify 后重点验证

1. `agent_intent` 是否仍是 Agent 节点，并能输出 V4 JSON 字段。
2. `validate_and_route` 是否能接收 `agent_intent.text`。
3. 原 8 条测试是否保持通过。
4. 新 6 条 V4 测试是否命中预期政策和风险等级。
5. `answer_verifier` 是否输出合法 JSON。
6. 高风险场景是否不会承诺“已退款成功”。
