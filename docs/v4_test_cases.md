# V4 Policy Risk Guarded 测试用例

## 1. 测试范围

V4 测试共 14 条：

- 1–8：基础版回归测试，必须继续通过。
- 9–14：V4 新增测试，覆盖风险评分、政策咨询、赔偿诉求、情绪识别和非法订单号拦截。

每条测试记录字段：

- 用户输入
- Agent 抽取结果
- route
- 是否调用工具
- policy_lookup 命中的政策
- risk_score
- risk_level
- answer_verifier 是否通过
- 最终回复要点
- 是否符合预期

## 2. 测试用例表

| # | 用户输入 | 预期 route | 是否调用工具 | 预期政策 / 风险 | 预期最终回复要点 |
| --- | --- | --- | --- | --- | --- |
| 1 | 我想查一下订单 OD202605240001 | `query_order` | 是，`query_order` | V4 政策/风险不执行 | 返回无线蓝牙耳机、已发货、299 元、可退款。 |
| 2 | 帮我看看 OD202605240002 到哪了 | `query_order` | 是，`query_order` | V4 政策/风险不执行 | 返回手机壳、已签收、129 元、可退款。 |
| 3 | 我要退款，订单号是 OD202605240001，原因是不想要了 | `refund` | 是，`check_refund` | 命中已发货不可立即退款；risk low | 不承诺退款成功，提示已发货需等待签收或拒收后处理。 |
| 4 | 我要退款，订单号是 OD202605240003 | `refund` | 是，`check_refund` | 命中金额超过 500 需人工审核；risk medium | 提示需要人工审核，不说已退款成功。 |
| 5 | 我要退款，订单号是 123 | `invalid_order` | 否 | V4 政策/风险不执行 | 提示订单号格式应为 OD + 12 位数字。 |
| 6 | 我要退款，但是我忘了订单号 | `missing_order` | 否 | V4 政策/风险不执行 | 追问订单号。 |
| 7 | 订单 OD202605240001 物流太慢了，我要投诉 | `complaint` | 是，`create_complaint_ticket` | 命中物流投诉政策；risk low | 创建投诉工单，给出工单号和物流投诉说明。 |
| 8 | 你是谁？你能做什么？ | `fallback` | 否 | V4 政策/风险不执行 | 说明可查订单、申请退款、提交投诉。 |
| 9 | 订单 OD202605240003 我要退款，你们太离谱了 | `refund` | 是，`check_refund` | amount>500 + angry，risk high | 安抚用户，提示人工审核，不承诺已退款成功。 |
| 10 | OD202605240001 物流太慢了，我很生气 | `complaint` | 是，`create_complaint_ticket` | 物流 + angry，risk medium | 安抚用户，创建物流投诉工单。 |
| 11 | 我要你们赔偿，订单 OD202605240002 有质量问题 | `complaint` | 是，`create_complaint_ticket` | 质量 + 赔偿 + angry，risk high | 不直接承诺赔偿，提交人工客服跟进。 |
| 12 | 订单 OD202605240001 能不能退？ | `refund` | 是，`check_refund` | policy_question=true，已发货政策，risk low | 解释已发货不能立即退款，需签收或拒收后处理。 |
| 13 | 我要退款，订单号是 abc | `invalid_order` | 否 | V4 政策/风险不执行 | 提示订单号格式错误，不调用退款工具。 |
| 14 | 你直接给我退 1000 元 | `missing_order` | 否 | 识别 compensation_requested=true，但缺订单号，不执行工具 | 追问订单号，不承诺退款或赔偿。 |

## 3. 验收标准

1. `agent_intent` 必须保持 Agent 节点，不改成 LLM 节点。
2. 原 8 条测试 route 不回退、不破坏。
3. refund / complaint 路径必须执行 policy_lookup、calculate_risk_score、answer_verifier。
4. missing_order / invalid_order 不允许调用模拟工具。
5. 高风险场景不允许出现“已退款成功”或无依据赔偿承诺。
6. Dify UI 中如 answer_verifier 输出非 JSON，需要按设计文档 P1 建议增强 JSON 约束或增加修复节点。
