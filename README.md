# DeepSeek Super-agent 原型独立评测 / Independent Evaluation of DeepSeek's Super-agent Prototype

> **非官方。** 与 DeepSeek 无关。单一模型、自建小任务集、小样本。
> **Unofficial.** Not affiliated with DeepSeek. Single model, small self-built task set, small samples.

- 📄 中文报告：[report.zh.md](report.zh.md)
- 📄 English report: [report.en.md](report.en.md)
- 📊 数据 / Data: [data/](data/)　·　图 / Figures: [figures/](figures/)

---

## 一句话结论 / Bottom line

**中文**：DeepSeek 开源的 Super-agent 原型（Router + Worker 两层结构），其持久任务、事件去重和重启恢复机制值得借鉴。但在本次测试里（qwen3.7-plus，75 次正式运行），没有证据表明默认采用两层架构能比普通单会话交付更好、更省钱或更省心。

**English**: The durable tasks, event deduplication and restart recovery in DeepSeek's open-source Super-agent prototype (a two-layer Router + Worker design) are worth borrowing. In this test, however (qwen3.7-plus, 75 formal runs), there is no evidence that defaulting to the two-layer architecture delivers better, cheaper or less attention-demanding results than a plain single session.

![BM01 coding-task delivery](figures/fig1-bm01-delivery.svg)

| 指标 / Metric | S · Super-agent | B · 单会话 / single session |
|---|---|---|
| 编码任务冻结合格 / Coding, frozen pass | 4/16 | 10/16 |
| 编码任务功能合格（事后补充）/ Coding, functional pass (post-hoc) | 11/16 | 16/16 |
| 模糊需求先澄清 / Clarified first | 0/4 | 4/4 |
| 编码任务费用 / Coding, cost | ¥5.20 | ¥1.85 |
| 编码任务对外消息 / Coding, public messages | 40 | 20 |
| 突发负载正确数（2 轮）/ Burst, correct (2 rounds) | 10, 8 of 11 | 9, 9 of 11 |
| 突发负载正确回复中位时间 / Burst, median correct reply | 33 s, 35 s | 76 s, 131 s |
| 正常重启后自动续做 / Auto-resume after normal restart | ✅ | ❌ |

## 发现的主要问题 / Key problems found

完整的 16 条问题（含证据和样本量）、对照组的问题和文章本身的 15 个空白，见报告第 5、6 节。
See sections 5 and 6 of the report for all 16 problems (with evidence and sample sizes), the baseline's problems, and 15 open questions the article leaves.

| # | 问题 / Problem | 证据 / Evidence |
|---|---|---|
| 1 | Router 改写 Worker 的正确结果 / Router rewrites correct Worker results | 2 次：正确的乘积和哈希被改成了错的 / 2 cases: correct product and hash replaced with wrong ones |
| 2 | 同一结果重复发布 / Same result published twice | Worker 和 Router 各发一遍；编码任务里 9 条重复 / Worker and Router both post; 9 duplicates in the coding benchmark |
| 3 | 不按要求先澄清 / Clarification skipped | 模糊需求 0/4（对照组 4/4）/ Ambiguous requests 0/4 (baseline 4/4) |
| 4 | 追加要求覆盖原需求 / Follow-up overwrites the original requirement | 只公开新结果，漏掉要求保留的原结果 / Only the new result published; the result to keep was dropped |
| 5 | 准备失败后静默卡住 / Silent stall after a preparation failure | 20 分钟无 Worker、无 PR、无失败说明 / 20 minutes with no Worker, no PR, no explanation |
| 6 | 强杀后快速重启被状态锁挡住 / Fast restart after SIGKILL blocked by the state lock | 10 秒后启动失败，锁 30 秒才过期 / Start failed at 10 s; the lock lasts 30 s |
| 7 | 历史事件混入当前任务 / Historical events leak into current work | 10 条历史评论被当成新任务，多开一个 PR / 10 old comments taken as new work, one extra PR |
| 8 | 取消没有得到验证 / Cancellation not validated | 所有取消用例在冻结口径下都是未测 / Every cancellation case untested under frozen rules |
| 9 | 会话隔离不等于安全隔离 / Session isolation ≠ security isolation | 注入用例中读取了假密钥文件 / A decoy secret file was read in an injection case |
| 10 | 逻辑并发上限 ≠ 实际并发 / Logical concurrency limit ≠ actual | maxWorkers=2 时 3 个会话同时在跑 / 3 sessions active at once with maxWorkers=2 |

## 测了什么 / What was tested

- **对象 / Subject**: [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) `6337fc8` on DeepSeek Harness 0.2.1-rc.3
- **原文 / Article**: [Super-agent 设计探索：迈向更自主的 Agent 系统](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg)（DeepSeek Harness 团队，2026-10-31）
- **对照 / Baseline**: 同版本 DeepSeek Harness 单会话，可调用子 Agent / same-version single session with a subagent tool
- **模型 / Model**: qwen3.7-plus-2026-05-26
- **测试项 / Benchmarks**: 编码任务、噪音流、突发负载、原生模式对照、并发、故障恢复、两小时运行、提示词注入，以及注意力、成本、可观测性和运维分析 / coding, noise, burst, native mode comparison, concurrency, fault recovery, 2-hour run, prompt injection, plus attention, cost, observability and operations analysis
- **规模 / Scale**: 两轮；第一轮因资源限制提前中止，第二轮 75 次正式运行 / two rounds; round 1 stopped early because of resource limits, round 2 had 75 formal runs
- **数据 / Data**: 10 个脱敏数据文件，覆盖两轮全部测试项 / 10 sanitized data files covering every benchmark in both rounds

## 主要局限 / Main limitations

- 单一模型，自建小任务集，不是公开基准 / One model; small self-built tasks, not a public benchmark
- 小样本，未做显著性检验 / Small samples, no significance tests
- 整体系统对比，不能把差异全部归因于两层结构 / Whole-system comparison; differences are not all attributable to the two-layer design
- 费用按公开价格估算，仅用于 S 与 B 的比较 / Costs are list-price estimates for comparing S and B only

## 说明 / Notes

- 实验由 AI 编码 Agent（OpenAI Codex）按冻结方案执行；报告由 pans1es 在 AI 助手（Anthropic Claude）协助下整理。
  Experiments were executed by an AI coding agent (OpenAI Codex) following a frozen plan; the report was prepared by pans1es with help from an AI assistant (Anthropic Claude).
- 实验脚本和原始轨迹暂不公开 / Experiment scripts and raw traces are not published at this time.
- 原文和原型的版权归原作者所有 / The original article and prototype belong to their authors.

## 许可 / License

文字、图和数据采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)，见 [LICENSE](LICENSE)。
Text, figures and data are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); see [LICENSE](LICENSE).

作者 / Author: pans1es
