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
| 编码任务已知费用 / Coding, known cost | ¥5.20 | ¥1.85 |
| 编码任务对外消息 / Coding, public messages | 40 | 20 |
| 突发负载正确数（2 轮）/ Burst, correct (2 rounds) | 10, 8 of 11 | 9, 9 of 11 |
| 突发负载正确回复中位时间 / Burst, median correct reply | 33 s, 35 s | 76 s, 131 s |
| 正常重启后自动续做 / Auto-resume after normal restart | ✅ | ❌ |

## 测了什么 / What was tested

- **对象 / Subject**: [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) `6337fc8` on DeepSeek Harness 0.2.1-rc.3
- **原文 / Article**: [Super-agent 设计探索：迈向更自主的 Agent 系统](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg)（DeepSeek Harness 团队，2026-10-31）
- **对照 / Baseline**: 同版本 DeepSeek Harness 单会话，可调用子 Agent / same-version single session with a subagent tool
- **模型 / Model**: qwen3.7-plus-2026-05-26
- **测试项 / Benchmarks**: 编码任务、噪音流、突发负载、并发、故障恢复、两小时运行、提示词注入 / coding, noise, burst, concurrency, fault recovery, 2-hour run, prompt injection

## 主要局限 / Main limitations

- 单一模型，自建小任务集，不是公开基准 / One model; small self-built tasks, not a public benchmark
- 小样本，未做显著性检验 / Small samples, no significance tests
- 整体系统对比，不能把差异全部归因于两层结构 / Whole-system comparison; differences are not all attributable to the two-layer design
- 费用为按公开价格的代理估算，不是账单 / Costs are list-price proxy estimates, not invoices

## 说明 / Notes

- 实验由 AI 编码 Agent（OpenAI Codex）按冻结方案执行；报告由 pans1es 在 AI 助手（Anthropic Claude）协助下整理。
  Experiments were executed by an AI coding agent (OpenAI Codex) following a frozen plan; the report was prepared by pans1es with help from an AI assistant (Anthropic Claude).
- 实验脚本和原始轨迹暂不公开 / Experiment scripts and raw traces are not published at this time.
- 原文和原型的版权归原作者所有 / The original article and prototype belong to their authors.

## 许可 / License

文字、图和数据采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)，见 [LICENSE](LICENSE)。
Text, figures and data are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); see [LICENSE](LICENSE).

作者 / Author: pans1es
