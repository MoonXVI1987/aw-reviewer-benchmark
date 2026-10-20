# 大模型代码评审能力评测报告

**测试日期**：2026-10-21
**目的**：为 AW 项目选定比执行方（Hermes + DeepSeek）更强的评审方
**方法**：第三方公开题库，同一批题多方案对比召回率

## 摘要

在 50 个真实 PR、170 条人工验证缺陷标注上测试 7 个方案：

1. LangGraph + deepseek-chat 最优：78.2%，10.5s/PR
2. Hermes 框架不如裸 API：73.5% vs 75.3%，且慢 6.8 倍
3. 多报不等于高召回：裸 API 提 2125 条召回 75.3%，LangGraph 提 445 条召回 78.2%
4. 均低于商业工具：CodeRabbit 100%、mra-max 93.3% 且零误报
5. 本机默认 deepseek-flash 长文档评审技术性失败：32K 预算全烧在 reasoning，正文 0 字符

## 方法

题库：withmartian/code-review-benchmark（MIT，★291）

- 50 个真实 PR（Sentry / Grafana / Cal.com / Discourse / Keycloak）
- 173 条人工验证 golden comments
- 类别：bug 94 / concurrency 14 / api 13 / security 11 / style 10 等
- 严重度：Critical 12 / High 54 / Medium 61 / Low 46
- 附带 49 个商业工具成绩作基线

方案（7 个）：

| 方案 | 框架 | 模型 | 工具 |
|---|---|---|---|
| 裸 API | 无 | deepseek-chat | 无 |
| LangGraph | StateGraph+ToolNode | deepseek-chat | read_file |
| Hermes | hermes -z | deepseek-chat | file 工具集 |
| 裸 API | 无 | qwen3.8-max / qwen3.7-max | 无 |
| 裸 API | 无 | kimi-k2.6 / glm-5 | 无 |

控制变量：前三者同模型，差异归因于框架；后四者同框架，差异归因于模型。

判分（写死）：golden 命中需满足其一

- A. 代码标识符重叠（驼峰 / 下划线 / 全大写）
- B. 文件路径匹配
- C. ≥4 个英文实词重叠

A 是跨语言必需：模型中文作答、golden 英文，仅用 C 会把真实命中判成 miss（实测修正后 0/2 → 2/2）。

指标：recall = 命中 golden / 总数（每条只计一次）；fp_rate = 未匹配意见 / 意见总数。

公平性：所有对比均在跑满方案的 PR 交集上计算（48 个 PR / 170 条 golden）；完成 <45 个 PR 的方案不参与交集。

## 结果（48 PR / 170 golden）

| 方案 | recall | 意见数 | 延迟/PR |
|---|---|---|---|
| LangGraph + deepseek-chat | 133/170 = 78.2% | 445 | 10.5s |
| deepseek-chat 裸 API | 128/170 = 75.3% | 2125 | 16.3s |
| Hermes + deepseek-chat | 125/170 = 73.5% | 271 | 110.1s |
| qwen3.8-max | 97/132 = 73.5% | 172 | 15.5s |
| qwen3.7-max | 116/170 = 68.2% | 161 | 9.2s |
| kimi-k2.6 | 111/170 = 65.3% | 473 | 98.9s |

商业工具基线（前 5）：CodeRabbit 100%/57% 误报、codeant 96.7%/50%、mra-max 93.3%/0%、sourcery 93.3%/37.3%、greptile-v4 86.7%/42.2%。

## 关键发现

1. **多报不能买到高召回**：裸 API 提 2125 条（44/PR）召回 75.3%；LangGraph 提 445 条（9/PR）召回 78.2%。差异在定位质量，不在数量。
2. **Hermes 复杂系统提示是负担**：同模型下 73.5% vs 裸 API 75.3%，且慢 6.8 倍。推测通用系统提示强调谨慎，抑制了报告倾向（未单独验证）。
3. **deepseek-flash 长文档失败**：4000/16000/32000 三档 max_tokens 全是 reasoning_tokens、正文 0 字符；换 deepseek-chat 24.2s 得 19708 字符、golden 2/2。
4. **免费额度不够**：qwen3.8-max 14 个 PR 后 HTTP 403 Free quota exhausted，单模型完整评测约需 170 万输入 tokens。

## 局限

- 判分规则：碎片化使 fp_rate 偏低；同义表达漏判使 recall 偏低（recall 是下限）
- 只给 diff，无仓库上下文
- 单次评审，无迭代
- 48 PR/170 golden 足以给方向性结论，不足以支撑 1-2 个点差异（未做显著性检验）
- glm-5 仅 2 个 PR，不参与结论
- 未测 Claude/GPT（api.anthropic.com 对香港 IP 返 403）

## 结论

- 推荐 LangGraph + deepseek-chat 作为骨架（最高召回、最低延迟）
- 不推荐 Hermes 当评审方（更低更慢）
- 不推荐 qwen / kimi 系列（均低于 deepseek-chat）
- 当前方案不足以作为强于执行方的第二意见：最高 78.2%，执行方同模型裸跑 75.3%，差距在统计波动内
- 需 Claude Sonnet 5.5 级别模型（Terminal-Bench 70.6%、GDPval-AA 1844）；本机需日本/新加坡出口

## 复现

完整报告：`G:/HermesKnowledge/AW评审方客观评测-完整报告-20261021.md`

脚本与原始数据：`G:/HermesHome/cache/research-aw/`

- `eval_bench.py`（裸 API）、`langgraph_bench.py`（LangGraph）、`hermes_batch.py`（Hermes）、`summarize.py`（汇总）
- `eval_results/*.jsonl`（逐 PR 原始输出）、`report_data.json`（本报告数字来源）
- `benchmark_data.json`（题库）、`diffs/*.diff`（50 个 PR diff）
