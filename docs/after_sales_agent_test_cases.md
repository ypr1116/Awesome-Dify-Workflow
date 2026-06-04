# 电商售后客服 Agent Demo 测试用例

## 测试目标

验证 `after_sales_agent_demo.yml` 是否满足以下规则：

- 支持查订单、申请退款、投诉、闲聊 / 其他咨询 4 类意图。
- 查订单、退款、投诉均需要合法订单号。
- 订单号格式必须匹配 `^OD\d{12}$`。
- 订单号缺失或格式错误时，不调用 `query_order`、`check_refund`、`create_complaint_ticket`。
- 退款金额 `amount <= 500` 时走自动退款处理。
- 退款金额 `amount > 500` 时提示人工审核。

## 测试用例列表

| # | 用户输入 | 预期意图 | 是否需要订单号 | 是否应该调用工具 | 应调用的工具 | 预期回复要点 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 我想查一下订单 OD202605240001 | `query_order` | 是 | 是 | `query_order` | 返回订单 `OD202605240001`；商品为“无线蓝牙耳机”；状态“已发货”；金额 `299` 元；可退款。 |
| 2 | 帮我看看 OD202605240002 到哪了 | `query_order` | 是 | 是 | `query_order` | 返回订单 `OD202605240002`；商品为“手机壳”；状态“已签收”；金额 `129` 元；可退款。 |
| 3 | 我要退款，订单号是 OD202605240001，原因是不想要了 | `refund` | 是 | 是 | `check_refund` | 返回退款申请已受理；退款金额 `299` 元；原因“不想要了”；金额不超过 500 元，不需要人工审核；提示已提交自动退款处理。 |
| 4 | 我要退款，订单号是 OD202605240003 | `refund` | 是 | 是 | `check_refund` | 返回退款申请已受理；退款金额 `899` 元；金额超过 500 元；`need_human_check = true`；包含人工客服审核提示：“该订单退款金额超过 500 元，需要人工客服审核。我已为你提交人工审核，请等待客服确认。” |
| 5 | 我要退款，订单号是 123 | `refund` | 是 | 否 | 无 | 提示订单号格式不正确；说明正确格式为 OD + 12 位数字，例如 `OD202605240001`；不调用退款工具。 |
| 6 | 我要退款，但是我忘了订单号 | `refund` | 是 | 否 | 无 | 追问订单号；提示格式为 OD + 12 位数字，例如 `OD202605240001`；不调用退款工具。 |
| 7 | 订单 OD202605240001 物流太慢了，我要投诉 | `complaint` | 是 | 是 | `create_complaint_ticket` | 创建投诉工单；返回工单号；订单号为 `OD202605240001`；投诉类型为售后 / 物流相关投诉；状态“已创建”。 |
| 8 | 你是谁？你能做什么？ | `chitchat` | 否 | 否 | 无 | 进入兜底回复；说明自己是电商售后客服 Demo；能演示查订单、申请退款和提交投诉；提示订单号格式示例。 |

## 分场景验收说明

### 1. 查订单成功

输入：

```text
我想查一下订单 OD202605240001
```

预期路径：

```text
start → agent_intent → validate_and_route → route_by_intent(query_order) → tool_query_order → answer_query_order
```

预期不会触发：

- `answer_missing_order`
- `answer_invalid_order`
- `tool_check_refund`
- `tool_create_complaint_ticket`

### 2. 小额退款自动处理

输入：

```text
我要退款，订单号是 OD202605240001，原因是不想要了
```

预期路径：

```text
start → agent_intent → validate_and_route → route_by_intent(refund) → tool_check_refund → answer_refund
```

预期回复包含：

- 订单号 `OD202605240001`
- 退款金额 `299` 元
- 不需要人工审核
- 自动退款处理结果

### 3. 大额退款人工审核

输入：

```text
我要退款，订单号是 OD202605240003
```

预期回复必须包含：

```text
该订单退款金额超过 500 元，需要人工客服审核。我已为你提交人工审核，请等待客服确认。
```

### 4. 订单号格式错误拦截

输入：

```text
我要退款，订单号是 123
```

预期路径：

```text
start → agent_intent → validate_and_route → route_by_intent(invalid_order) → answer_invalid_order
```

预期不会调用任何模拟工具。

### 5. 订单号缺失拦截

输入：

```text
我要退款，但是我忘了订单号
```

预期路径：

```text
start → agent_intent → validate_and_route → route_by_intent(missing_order) → answer_missing_order
```

预期不会调用任何模拟工具。


## 导入后人工验证建议

1. 在 Dify 中导入 `after_sales_agent_demo.yml`。
2. 按上表从 1 到 8 逐条输入用户问题。
3. 对查订单、退款、投诉用例，重点确认只有合法订单号才进入对应模拟工具节点。
4. 对用例 5 和用例 6，重点确认不会调用任何模拟工具。
5. 对用例 4，重点确认回复中出现大额退款人工审核提示。
