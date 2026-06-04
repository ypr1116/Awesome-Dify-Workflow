# V4.1 Memory Friendly 测试用例

## 1. 测试范围

V4.1 测试覆盖：

1. 原 8 条基础测试回归。
2. V4 新增 6 条风险 / 政策 / 校验测试回归。
3. V4.1 新增上下文记忆、order_only、自然 fallback、投诉订单不存在拦截测试。

## 2. V4.1 新增测试

| # | 场景 | 用户输入 | 预期 |
| --- | --- | --- | --- |
| 15 | 第一轮退款缺订单号 | 我要退款 | route=`missing_order`，追问订单号，记录 `pending_intent=refund` |
| 16 | 第二轮只给订单号 | OD202605240001 | 继承 `pending_intent=refund`，进入 refund 路径，处理后清空 pending |
| 17 | 第一轮投诉缺订单号 | 我要投诉 | route=`missing_order`，追问订单号，记录 `pending_intent=complaint` |
| 18 | 第二轮只给订单号 | OD202605240001 | 继承 `pending_intent=complaint`，进入 complaint 路径，处理后清空 pending |
| 19 | 单独输入订单号 | OD202605240001 | 无 pending 时 route=`order_only`，询问用户想查订单、退款还是投诉 |
| 20 | 第一轮查询订单 | 查一下订单 OD202605240001 | route=`query_order`，更新 `last_order_id=OD202605240001` |
| 21 | 第二轮省略订单号投诉 | 物流太慢了，我要投诉 | 复用 `last_order_id=OD202605240001` 创建投诉工单 |
| 22 | 不存在订单投诉 | 订单 OD202605249999 物流太慢了，我要投诉 | route=`complaint`，但不创建工单，提示核对订单号 |
| 23 | 自然问候 | 你好 | route=`fallback`，自然问候，不机械要求订单号 |

## 3. 全量回归要求

- 原 8 条基础测试继续通过。
- V4 新增 6 条风险 / 政策测试继续通过。
- V4.1 新增 9 个多轮/单轮步骤通过。
- `missing_order` / `invalid_order` 不调用模拟工具。
- `order_only` 不调用模拟工具。
- 不存在订单投诉不创建工单。
- 高风险 refund / complaint 仍经过 `policy_lookup`、`calculate_risk_score`、`answer_verifier`。

## 4. Dify UI 重点验证

本地等价测试用内存字典模拟 conversation variables。导入 Dify 后需要重点确认：

1. 6 个 conversation variables 是否成功创建。
2. assigner 节点是否能正确写入 conversation variables。
3. 第二轮只输入订单号时，是否能读取上一轮 `pending_intent`。
4. 查询订单后再投诉时，是否能读取 `last_order_id`。
5. answer_verifier 是否稳定输出 JSON。
