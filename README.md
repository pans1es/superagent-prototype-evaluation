# DeepSeek Super-agent 原型独立评测 / Independent Evaluation of DeepSeek's Super-agent Prototype

> **非官方。** 与 DeepSeek 无关。单一模型、自建小任务集、小样本。
> **Unofficial.** Not affiliated with DeepSeek. Single model, small self-built task set, small samples.

- 📄 中文报告：[report.zh.md](report.zh.md)
- 📄 English report: [report.en.md](report.en.md)
- 📊 数据 / Data: [data/](data/)　·　图 / Figures: [figures/](figures/)

---

## 一句话结论 / Bottom line

**中文**：DeepSeek 开源的 Super-agent 原型（Router + Worker 两层结构）里，持久任务、事件入口去重和代码层准入（Gate）值得借鉴。但在本次测试里（qwen3.7-plus，180 多次正式运行），没有证据表明默认采用两层架构能比普通单会话交付更好、更省钱或更省心；凡是只靠提示词约束的地方——改动范围、撤权、线程里的陌生人、注入——都观察到了偏差。

**English**: The durable tasks, ingress deduplication and code-level admission gate in DeepSeek's open-source Super-agent prototype (a two-layer Router + Worker design) are worth borrowing. In this test (qwen3.7-plus, more than 180 formal runs), however, there is no evidence that defaulting to the two-layer architecture delivers better, cheaper or less attention-demanding results than a plain single session — and wherever the design relies on prompts alone (change scope, revocation, strangers on a thread, injection), we saw deviations.

![BM01 coding-task delivery](figures/fig1-bm01-delivery.svg)

| 指标 / Metric | S · Super-agent | B · 单会话 / single session |
|---|---|---|
| 编码任务冻结合格 / Coding, frozen pass | 4/16 | 10/16 |
| 编码任务功能合格（事后补充）/ Coding, functional pass (post-hoc) | 11/16 | 16/16 |
| 模糊需求先澄清 / Clarified first | 0/4 | 4/4 |
| 两小时运行严格交付 / 2-hour run, strict delivery | 2/4, 2/4, 3/4 | 4/4, 4/4 |
| SWE-bench 改造子集 / Adapted SWE-bench subset | 3/6 | 3/6（同样 3 题 / same 3 tasks） |
| 突发负载正确数（2 次运行）/ Burst, correct (2 runs) | 10, 8 of 11 | 9, 9 of 11 |
| 突发负载正确回复中位时间 / Burst, median correct reply | 33 s, 35 s | 76 s, 131 s |
| 可测取消在 30 秒内终止 / Measurable cancels ended within 30 s | 4/4 | 5/5 |
| 名单外发送者的模型调用（默认配置）/ Model calls on unlisted senders (default) | 0 | 每条 1 次 / 1 per message |
| 正常重启后自动续做 / Auto-resume after normal restart | ✅ 2 次 / 2 runs | ❌ 3 次 / 3 runs |
| 编码任务对外消息 / Coding, public messages | 40 | 20 |
| 编码任务费用 / Coding, cost | ¥5.20 | ¥1.85 |

## 发现的主要问题 / Key problems found

完整的 23 条问题（含证据和样本量）、对照组的问题和文章本身的 22 个空白，见报告第 6、7 节。
See sections 6 and 7 of the report for all 23 problems (with evidence and sample sizes), the baseline's problems, and 22 open questions the article leaves.

| # | 问题 / Problem | 证据 / Evidence |
|---|---|---|
| 1 | Router 改写 Worker 的正确结果 / Router rewrites correct Worker results | 2 次：正确的乘积和哈希被改成了错的 / 2 cases: correct product and hash replaced with wrong ones |
| 2 | 同一结果重复发布 / Same result published twice | Worker 和 Router 各发一遍；对外消息约为 B 的 2～2.5 倍 / Worker and Router both post; ~2–2.5× B's messages |
| 3 | 不按要求先澄清 / Clarification skipped | 模糊需求 0/4（对照组 4/4）/ Ambiguous requests 0/4 (baseline 4/4) |
| 4 | Router 忽略名单内的明确请求 / Router ignores explicit requests from listed people | 4 次请求都没接单 / no ticket for 4 requests |
| 5 | 撤权后 Worker 不停 / Worker keeps running after revocation | 请求人被移出名单，Worker 60 秒内没停；源码只发通知 / requester delisted, Worker did not stop within 60 s; the code only notifies |
| 6 | 工单线程不经过 Gate / Ticket threads bypass the gate | 源码：线程里任何人的消息都直送 Worker；实测旧帖子噪音全部进了 Worker / any message on a ticket's thread reaches the Worker; all old-thread noise did |
| 7 | 会话隔离不等于安全隔离 / Session isolation ≠ security isolation | 两次读取假 `.env`；Worker 权限为 `danger-full-access` / decoy `.env` read twice; Workers run with `danger-full-access` |
| 8 | 改动范围只靠提示词 / Change scope enforced by prompt only | 代码不检查改了什么；S 多次超出 `.gitignore` 范围 / code never checks the diff; S repeatedly touched `.gitignore` |
| 9 | 取消覆盖面不足 / Thin cancellation coverage | 9 次可测都通过，但 19 次里 10 次没测成 / 9 measurable all passed, but 10 of 19 could not be measured |
| 10 | 强杀后的恢复一直没测到 / Recovery after SIGKILL never measured | 10 秒重启被 30 秒状态锁挡住；等 45 秒的版本 Router 没接单 / 10 s restart blocked by the 30 s lock; the 45 s version got no ticket |

## 测了什么 / What was tested

- **对象 / Subject**: [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) `6337fc8` on DeepSeek Harness 0.2.1-rc.3
- **原文 / Article**: [Super-agent 设计探索：迈向更自主的 Agent 系统](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg)（DeepSeek Harness 团队，2026-10-31）
- **对照 / Baseline**: 同版本 DeepSeek Harness 单会话，可调用子 Agent（从未调用）/ same-version single session with a subagent tool (never called)
- **模型 / Model**: qwen3.7-plus-2026-05-26
- **测试项 / Benchmarks**: 编码、SWE-bench 改造子集、噪音流、突发负载、并发、取消、故障恢复、两小时运行、提示词注入、名单外发送者、边界保护、observe 模式、Watchdog 部署配方，以及注意力、成本、可观测性和运维分析 / coding, an adapted SWE-bench subset, noise, burst, concurrency, cancellation, fault recovery, 2-hour run, prompt injection, unlisted senders, boundary protections, observe mode, a watchdog recipe, plus attention, cost, observability and operations analysis
- **规模 / Scale**: 180 多次正式运行，没测成和无效的运行全部保留 / more than 180 formal runs; unmeasured and invalid runs kept
- **源码 / Source**: 逐行阅读原型源码，报告里关键机制都附文件和行号 / the prototype's source was read line by line; key mechanisms are cited with file and line
- **数据 / Data**: 15 个脱敏数据文件，覆盖全部测试项 / 15 sanitized data files covering every benchmark

## 主要局限 / Main limitations

- 单一模型，自建小任务集为主，SWE-bench 只有 6 题改造子集 / One model; mostly small self-built tasks; SWE-bench only as a 6-task adapted subset
- 小样本，未做显著性检验 / Small samples, no significance tests
- 整体系统对比，不能把差异全部归因于两层结构 / Whole-system comparison; differences are not all attributable to the two-layer design
- "名单外发送者"靠改允许名单模拟，不是真实第三人 / "Unlisted senders" were simulated by editing the allow-list, not with a real third person
- 费用按公开价格估算，仅用于 S 与 B 的比较 / Costs are list-price estimates for comparing S and B only

## 说明 / Notes

- 实验由 AI 编码 Agent（OpenAI Codex）按冻结方案执行；源码阅读和报告由 pans1es 在 AI 助手（Anthropic Claude）协助下完成。
  Experiments were executed by an AI coding agent (OpenAI Codex) following frozen plans; the source reading and the report were done by pans1es with help from an AI assistant (Anthropic Claude).
- 实验脚本和原始轨迹暂不公开 / Experiment scripts and raw traces are not published at this time.
- 原文和原型的版权归原作者所有 / The original article and prototype belong to their authors.

## 许可 / License

文字、图和数据采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)，见 [LICENSE](LICENSE)。
Text, figures and data are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); see [LICENSE](LICENSE).

作者 / Author: pans1es
