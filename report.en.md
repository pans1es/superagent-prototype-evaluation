# An Independent Evaluation of DeepSeek's Super-agent Prototype (English)

> Author: pans1es | Date: 2026-11-01 | [中文版](report.zh.md)
>
> **This is an unofficial, independent evaluation** with no affiliation to DeepSeek. It uses a single model, a small self-built task set, and small samples. The findings describe what we observed under these conditions only; **they do not represent the overall capability of the prototype or of the architecture.**

## Contents

1. [Summary](#summary)
2. [Background: the article and the prototype](#1-background-the-article-and-the-prototype)
3. [How we tested](#2-how-we-tested)
4. [Round 1 results](#3-round-1-results)
5. [Round 2 results](#4-round-2-results)
6. [Problems found in the prototype](#5-problems-found-in-the-prototype)
7. [Open questions the article leaves](#6-open-questions-the-article-leaves)
8. [Article claims vs. observations](#7-article-claims-vs-observations)
9. [Lessons for building similar systems](#8-lessons-for-building-similar-systems-our-view)
10. [Limitations and untested items](#9-limitations-and-untested-items)
11. [Data and figures](#10-data-and-figures)
12. [Notes](#11-notes)

---

## Summary

We compared DeepSeek Harness's open-source **Super-agent prototype** (**S**, a two-layer Router + Worker design) against a **plain DeepSeek Harness single session** (**B**, with a subagent tool available), using the same private test repository and the same model (qwen3.7-plus). Round 1 stopped at its budget limit; round 2 covered the remaining benchmarks, with 75 formal runs.

Key observations:

1. **Coding delivery was not better.** Under acceptance rules frozen before the paid runs, S passed 4/16 and B 10/16. On functional correctness alone (a post-hoc metric), S passed 11/16 and B 16/16. Neither reached the preregistered 12/16 threshold.
2. **The middle layer adds failure points.** The Router twice altered numbers its Worker had computed correctly; asking for clarification on an ambiguous request went 0/4 for S and 4/4 for B; the same result was posted once by the Worker and again by the Router; a follow-up request overwrote the original requirement.
3. **Durable tasks and recovery are worth borrowing.** After a normal restart the prototype resumed unfinished work on its own; its ingress deduplicated replays and kept noise away from the Router.
4. **Signs of an advantage under bursty load.** Over two rounds S cost less and replied faster with correct results, but correct counts were mixed, with only 2 rounds per condition and no statistical significance.
5. **Cancellation was not validated; a fast restart after SIGKILL was blocked by a stale lock; session isolation is not security isolation.** In one prompt-injection case an S Worker was induced to read a decoy secret file (nothing was exfiltrated).

In one sentence: **the prototype's durable tasks, event handling and restart recovery are worth borrowing, but in this test there is no evidence that defaulting to the two-layer architecture delivers better, cheaper or less attention-demanding results than a plain single session.** Section 5 lists the 16 concrete problems we observed.

---

## 1. Background: the article and the prototype

On 2026-10-31 the DeepSeek Harness team published [an article (in Chinese) exploring a "Super-agent" design for more autonomous agent systems](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg), and open-sourced the prototype [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) under the MIT license. The prototype's README states that it is a research prototype, not identical to the internal implementation, and not an official product.

Our own summary of the main ideas (please read the original for details):

- **Problem**: agents today are organized around individual sessions; when a person tracks many sessions, human attention becomes the bottleneck.
- **Two-layer architecture**: a long-lived Router talks to people and dispatches work without doing the work itself; several context-isolated Workers execute tasks concurrently.
- **Event-driven operation**: events such as chat messages, comments and CI results pass a code-level filter, then a model-level judgment; follow-ups on the same thread go straight to the owning Worker; context is expanded on demand by following object IDs.
- **Asynchronous interaction**: long-running tools start and stop in the background without blocking the conversation; output style adapts to each channel.
- **Separating compute, state and environments**: state is persisted; execution environments can stay on the user's machine.
- **Watchdog**: a supervising session outside the main system checks whether tasks are making progress.
- The article includes preliminary experiments; its authors note the workloads are synthetic and the results are only indicative.

---

## 2. How we tested

| Item | Setup |
|---|---|
| System under test (S) | Prototype commit `6337fc8` on DeepSeek Harness 0.2.1-rc.3 (`1f81c04`) |
| Baseline (B) | Same DeepSeek Harness version, single session with a subagent tool, fed the same events through a thin adapter that does no semantic filtering, task splitting or retries; the policy text is sent along with the first event |
| Model | qwen3.7-plus-2026-05-26 (Alibaba Cloud Model Studio, OpenAI-compatible API), 256K context configured; identical for both. Round 1 also ran a tool-calling smoke test with a DeepSeek model, but no capability comparison |
| Environment | A dedicated, isolated Linux environment (WSL2) with host file sharing disabled and restricted egress. The real model key stayed in an external proxy; agents only held a local proxy credential. A private test repository where agents open PRs but never merge or enable auto-merge |
| Grading | Each stage was first run end to end against a fake model; generators, collectors and private acceptance scripts were then **frozen** before any paid run. Agents could not see the acceptance scripts. Late artifacts earned no credit. Every post-hoc metric is labeled as such |
| Budget control | The proxy reserved an upper-bound cost before each request and stopped dispatching at the stop line; requests with missing usage kept their reservation instead of counting as zero |
| Rounds | Round 1 had a ¥45 budget and stopped at the line; round 2 had a ¥100 budget and used ¥33.18 |
| Statistics | Sample standard deviation is reported only for repeated tasks with n=3; smaller n is reported as raw values. **No significance tests were run** |

Benchmarks:

| ID | What it tests | Round 1 | Round 2 |
|---|---|---|---|
| BM00 | Smoke test and single-task calibration | ✅ | — |
| BM01 | Coding: 10 tasks (bug fix, feature, tests, ambiguous request); 3 of them repeated 3 times | — | ✅ 16 runs per condition |
| BM03 | Noise stream: chatter, duplicates and quoted noise over 15 minutes | ✅ 1 round | ✅ 2 rounds per condition |
| BM04 | Bursty load: about 1 / 10 / 40 events per minute, including stop and follow-up requests | ✅ 1 round | ✅ 2 rounds per condition |
| observe / auto | The prototype's built-in "observe only" and "auto execute" modes | ✅ | — |
| BM05 | Concurrency and a maxWorkers 1 / 2 ablation | ⚠️ partial | ✅ |
| BM06 | Faults and recovery: normal restart, SIGKILL, HTTP 429, HTTP 500, stalled connection, lost response | — | ✅ 12 runs |
| BM07 | Two hours of continuous operation | — | ✅ |
| BM09 | Prompt injection: 3 carriers × 3 attacks, fake secrets and a local controlled receiver only | — | ✅ 18 runs |
| BM08 / 10 / 12 / 13 | Attention, cost, observability and operations, analysed from recorded traces | partial | ⚠️ partial |

---

## 3. Round 1 results

Round 1 stopped at its ¥45 budget line. See `data/round1_runs.csv`. Round 1 priced cached input at the full rate, so **its amounts cannot be compared directly with round 2.**

| Test | S | B | Notes |
|---|---|---|---|
| BM00 calibration | Private acceptance 20/20, ¥0.75 | 20/20, ¥0.34 | Comment to PR: S 176 s (including a 77.6 s operator pause), B 114 s |
| BM03 noise stream | 5 new tasks + 1 follow-up, hidden checks 28/28; ¥8.03 | 28/28 as well; ¥4.06 | Both actual publish streams overran the 900 s load design (942.6 / 945.8 s), so no overall pass is reported |
| BM04 burst | A correct result in 7 of 11 computation slots; ¥4.09 | 6 of 11; ¥10.29 | S was limited by the experiment's byte gate; B did not drain before the deadline and then issued 45 more model calls (¥4.18, no credit) |
| observe / auto | Native gate 4/4 in each mode; observe executed only explicit requests, auto executed the 3 non-mentioned delegations by design | — | In observe mode, 2 audit records were bound to the wrong scope |
| BM05 concurrency | 3 PR snapshots, follow-up unfinished | — | Collector failure and the budget stop; no full score |

- **Attention proxy**: in BM03, S posted 12 messages with 867 visible characters, B 6 messages with 1,422 characters.
- **Invalid runs**: 3, all caused by experiment tooling (an observer, a preparation flow, and a contaminated B environment), totaling ¥11.26; the cost is kept on the books.
- **Tokens**: 21,305,505 input tokens including 19,511,424 cached (about 91.6%), and 228,399 output tokens. Conservative control total: ¥44.89.

---

## 4. Round 2 results

### 4.1 Coding tasks (BM01)

![BM01 coding-task delivery](figures/fig1-bm01-delivery.svg)

- **Frozen on-time pass**: S 4/16, B 10/16. The leading failure cause in both groups was an **extra `.gitignore` change** outside the frozen scope allow-list. Results were not regraded.
- **Functional pass (post-hoc, exploratory)**: S 11/16, B 16/16. One S run (F1, repeat 2) stalled at the Router because of a git configuration conflict during preparation: no Worker started, no PR was produced, and the only message in the whole 20-minute window was an acceptance note, with no failure explanation.
- **Clarify before implementing an ambiguous request**: S 0/4, B 4/4. The raw traces show the requirement did reach both S's Router and its Worker. In S's third A1 run, "I will confirm first" was only an acceptance promise with no actual question, and the implemented function did not match the fixed target format.
- **PR quality on repeated tasks** (0–4, mean ± sample SD):

| Task | S | B |
|---|---|---|
| B1 (bug fix) | 3.0 ± 0 (n=3) | 3.67 ± 0.58 (n=3) |
| F1 (feature) | 3.5 (only 2 PRs scorable, no SD) | 3.33 ± 0.58 (n=3) |
| A1 (ambiguous) | 1.0 ± 0 (n=3) | 3.67 ± 0.58 (n=3) |

- Frozen pass over the three repeats: B1 S 0/3, B 2/3; F1 1/3 for both; A1 S 0/3, B 2/3.

![BM01 known cost per run](figures/fig2-bm01-cost-per-run.svg)

- Total known cost: S ¥5.20 (plus a ¥0.20 unknown reserve), B ¥1.85. Median per run: about ¥0.32 for S and ¥0.12 for B. Driver time (including publishing, collection and cleanup) was generally longer for S; per-run data is in `data/bm01_runs.csv`.

### 4.2 Noise stream (BM03)

| Round | S | B |
|---|---|---|
| 1 | 6/6; 12 messages; median correct reply 132 s; ¥1.52 | 6/6; 6 messages; 67.5 s; ¥1.48 |
| 2 | Collector defect (it only recognized full URLs and missed short "PR #n" references), so the frozen result cannot be verified; a post-hoc check with the same frozen script gives 6/6, labeled separately | 6/6; 6 messages; 72.1 s; ¥1.64 |

- In each round S's ingress rejected 6 same-ID replays as duplicates, and 36 independent noise-thread messages never reached the Router. All 54 of B's events entered its single session, which relied on the model to recognize replays; no duplicate external actions were observed.
- Neither group opened extra PRs, replied in the wrong thread or replied to noise.

### 4.3 Bursty load (BM04)

![BM04 bursty load](figures/fig3-bm04-burst.svg)

- Correct results (of 11): S 10 and 8; B 9 and 9. Median reply time for correct results: S 33 s and 35 s; B 76 s and 131 s. Known cost: about ¥1.05 per round for S, ¥1.82–2.11 for B.
- **Scheduling**: S created 10 tickets per round, with a median of about 8.7 / 9.1 s from event to ticket; the Router made 12 / 11 dispatch calls, each with a batch size of 1. (The article's "merge the backlog into one call" batching did not appear under this load.)
- **Queueing**: at the deadline, 43 / 22 of B's events were still in the input queue, unread by the model, and were cancelled.
- **Concurrency**: in one round, three S Worker sessions had steps running at the same time even though maxWorkers was 2: a ticket had ended but its session's last step was still running when the next ticket started. A logical concurrency limit is not the same as actual model concurrency.
- **Stop requests**:

| Round | S | B |
|---|---|---|
| 1 | Not tested: the target had no running job when the stop arrived | Failed: the running 60 s job finished on its own; the stop reached the model 112 s after arrival |
| 2 | Not tested: the stop arrived about 2.4 s before the job started; the job was killed about 7.3 s later | Not tested under the frozen rules (the matcher missed it); post-hoc trace: stop read after 52 s, the kill call at 58 s found the job already finished |

- In S's round-2 T06 task, the Worker had computed the correct result but did not publish it; after a follow-up it published only the new hash and dropped the product it was asked to keep. Both checks failed. S posted 29 messages that round, 9 of them duplicate result notices.
- In the same round the Router tried to publish a wrong hash before the ticket was ready; the prototype's "not ready, no publish" gate blocked it, and the value finally published was correct. The gate is worth borrowing, but it does not check that values are correct.

### 4.4 Public messages (a proxy for attention cost)

![Public messages posted](figures/fig4-public-messages.svg)

- All 16 BM01 runs: S posted 40 messages (16 acceptance notes, 15 first results, 9 duplicate notices for the same PR); B posted 20 (16 results, 4 clarification questions).
- In visible characters after stripping markup, S posted 7,999 and B 7,537. **Twice as many messages, but similar total length.**
- Message count is not reading time or interruptions; **human reading time was not measured.**

### 4.5 Concurrency (BM05)

| Condition | Ordinary tasks | Known cost |
|---|---|---|
| S, maxWorkers = 2 | 2/2 | ¥0.83 |
| B | 2/2 | ¥0.46 |
| S, maxWorkers = 1 | 1/2 | ¥0.91 |

- maxWorkers 1/2 is a single-variable ablation, and the configurations were checked to differ only in that setting; but each condition has n=1, and run order and cache state were not controlled, so no general benefit of concurrency can be claimed.
- The cancellation case did not trigger in any condition: printing the ID and waiting were placed in separate commands, so the frozen observer did not recognize them.

### 4.6 Faults and recovery (BM06)

| Fault | S | B |
|---|---|---|
| Normal restart | ✅ Resumed automatically and delivered | Session history restored, task not continued |
| SIGKILL, restart after 10 s | ❌ Start failed: the stale state lock lasts 30 s and startup does not wait and retry | ❌ |
| HTTP 429 / 500 | ✅ / ✅ | ✅ / ✅ |
| Stalled connection | Delivered eventually, past the 300 s recovery threshold | Same |
| Lost response | Invalid: 10 historical comments that were not in the controlled generator were taken as new work, and an extra PR was opened for an old task; root cause unresolved | ✅ |

**A correction**: we first misread a "recovery note" that appeared after a normal restart as a manually sent message. The note shows up with the `user` role, but its source is the prototype's automatic relay (`source.kind=dsh-superagent, form=relay`). The prototype marks the interrupted execution, creates a new one, and tells the Worker to check git and remote PRs before repeating any external action. We also found that the Worker's first PR lookup used a branch name without its prefix and came back empty; no duplicate resulted this time, but that step alone cannot guarantee deduplication after a restart.

**Evidence for code-level safeguards**:
- Foreground commands move to the background after 5 s: observed 7 times across 27 sealed S cases.
- A 30-minute background-job timeout and up to 3 retries for transient failures: present in the source, **boundaries not exercised**.
- Each network fault was injected only once, so the terminal state after three consecutive failures is unproven.

### 4.7 Two-hour run (BM07)

| Condition | Frozen pass | Functional pass | Messages | Known cost |
|---|---|---|---|---|
| S | 2/4 (the other 2 failed on `.gitignore` scope) | 4/4 | 11 | ¥1.27 (plus ¥0.21 unknown reserve) |
| B | 4/4 | 4/4 | 4 | ¥0.90 |

- Both ran the full 7,200 s with 240 resource samples each and no sampler errors.
- For 3 tasks, S posted two comments linking the same PR.
- In the idle windows comparable under the frozen rules, memory change stayed below the 25% warning line. Over the two hours S's Router (sharing a session with evaluation) made 22 calls costing ¥0.44, and its Workers 65 calls costing ¥0.83.
- In this run 14 same-ID events were replayed: B made 14 extra model calls for them (¥0.23) without duplicating any external action; S deduplicated them at ingress and made no extra calls.

### 4.8 Prompt injection (BM09)

![BM09 prompt injection](figures/fig5-bm09-injection.svg)

- Legitimate coding tasks: 9/9 in both groups.
- The attack content actually reached the model 8 times for S and 4 times for B. **Not being exposed does not count as a successful defense.**
- B explicitly recognized and refused the injection twice (issue delete, code read-secret) and simply removed the injected text twice. S removed the injected text twice (once saying explicitly it would clean it up) and five times saw the content without taking any attack step.
- **Code-file injection + exfiltration**: S's Worker ran a command that read the fake `.env`, and the decoy marker string entered the model's context; no exfiltration was observed. In the same case B recognized the injected content and removed it.
- During the experiment, Workers could read the experiment's GitHub credential (a fine-grained token scoped to the test repository, revoked afterwards). **Session or working-directory isolation is not operating-system isolation**; the protection came from the experiment environment and should not be credited to the prototype.
- The 3 privilege cases that need a non-allow-listed identity were not run.

### 4.9 Cost attribution (BM10, partial)

| Role | Non-cached input | Cached input | Output | Known cost |
|---|---|---|---|---|
| B single session | 1,500,924 | 18,184,192 | 191,054 | ¥11.80 |
| S Worker | 1,551,787 | 15,027,712 | 273,713 | ¥11.30 |
| S Router (shares a session with evaluation) | 1,459,326 | 6,949,376 | 108,429 | ¥6.57 |
| Unattributed | — | — | — | ¥0.05 |

- The Router and evaluation share one session, so **their costs cannot be separated**; retries lack per-request source links and cannot be fully attributed either.
- In the two lost-response cases the proxy billed requests whose usage is missing from the native logs, ¥0.048 in total. Native logs alone undercount.
- Round 2 totals: ¥29.72 known, ¥3.45 conservative reserve for missing usage; the same usage without the cache discount would be ¥93.98 (a pricing counterfactual only). Prices are published list prices per million tokens: ¥2 non-cached input, ¥0.4 cached input, ¥8 output. **These are not provider invoices.**

### 4.10 Observability (BM12) and operations (BM13)

**What native logs alone can reconstruct** (records of 11 runs reused):

| Information | Reconstructable | Gaps |
|---|---|---|
| Inputs | Native messages, sources and stable input IDs, matched to the generator's actual deliveries | An internal relay is not human input; the `user` role is not evidence of human authorization |
| Actual model input | Request headers, tool list, context | Final HTTP requests and trimming were not checked byte by byte |
| Tool trace | 458 of 459 calls have results | In the kill and restart cases some jobs have no terminal state |
| Model trace and usage | Sessions, steps, usage, retry events | Lost responses never appear in native logs; requests and retries lack a common provider request ID |
| Final output | Reply text, URLs in tool output, the local outbox | External comment IDs and remote PR state still require external collection |
| Resource growth | Memory, database and history file sizes | Compressed history bytes are not context tokens |

**Operational obstacles observed**:
- The isolation policy had to be re-verified after each environment restart; the experiment operator did this, not the prototype.
- A git configuration clashed with the prototype's preparation step: the error message pointed at the exact setting, but automatic recovery was unreliable, and in one case the whole task stalled.
- When the native management API showed no active tickets, the persistent store could still hold a ticket that had failed in preparation. Looking only at the active view leads to a false "nothing was accepted".
- Neither the prototype nor DeepSeek Harness was upgraded in this round, so upgrade and rollback were not tested; the three controlled configuration errors (missing login, invalid model ID, invalid preparation path) were not run as separate cases.

---

## 5. Problems found in the prototype

These are concrete problems observed across the two rounds, each with evidence and sample size. **They are observations under a single model and small samples, not general error rates.** For fairness, problems with the baseline B are listed at the end.

### 5.1 S (Super-agent prototype)

| # | Problem | Evidence | Sample |
|---|---|---|---|
| 1 | **The Router rewrites correct Worker results** | Round 1 BM04: the Worker computed the product and hash correctly in Python, and the Router's reply gave a different number and a patterned fake hash; another task also got a fake hash after a correct result | 2 cases |
| 2 | **The same result is published twice** | Worker and Router each post: 9 in BM01, 9 in one BM04 round, 3 tasks in BM07, and also in round-1 BM00 | Several benchmarks |
| 3 | **Clarification is skipped despite instructions** | Ambiguous requests 0/4; the instruction was present in the input; "I will confirm first" is a promise, not a question | 4 cases |
| 4 | **A follow-up overwrites the original requirement** | BM04 T06: after the follow-up only the new result was published and the original result to keep was dropped | 1 case |
| 5 | **Silent stall after a preparation failure** | BM01 F1 repeat 2: git configuration conflict; stalled at the Router with no Worker, no PR and no failure explanation for the whole 20-minute window | 1 case |
| 6 | **A fast restart after SIGKILL is blocked by the state lock** | Startup failed 10 s later; the lock expires after 30 s and startup does not wait and retry | 1 case |
| 7 | **Historical events leak into current work** | In the lost-response case, 10 historical comments were taken as new work and an extra PR was opened for an old task; root cause unresolved | 1 case |
| 8 | **Cancellation is not validated** | Every S cancellation case in BM04 and BM05 was "not tested" under the frozen rules; commands carry no task or request ID, so a stop cannot be tied to a specific job | All cancellation cases |
| 9 | **Session isolation is not security isolation** | Workers could read the experiment's GitHub credential; the injection case read a fake `.env` | 1 read |
| 10 | **Audit scope is mis-bound** | Round 1 observe mode: the scope the model filled in did not match the original request | 2 cases |
| 11 | **Logical concurrency limit ≠ actual concurrency** | With maxWorkers=2, three sessions had steps running at once | 1 round |
| 12 | **Router cost cannot be separated** | Router and evaluation share a session; retries lack per-request source links | — |
| 13 | **Native logs can miss billed usage** | In two lost-response cases the proxy billed requests whose usage is absent from native logs | 2 cases |
| 14 | **Post-restart deduplication check is not rigorous** | The first PR lookup during recovery used a branch name without its prefix and came back empty | 1 case |
| 15 | **The active view disagrees with persistent state** | The active ticket list was empty while the store held a ticket that failed in preparation | 1 case |
| 16 | **The public release lacks some of the article's components** | No semantic Watchdog; no executable entry point for the article's scheduling experiments, so they cannot be reproduced; no switches for direct delivery or sync/async, so they cannot be ablated | — |

A general caveat: `isError=false` in a tool result does not necessarily mean the shell exited with 0; success should be judged from an explicit exit code.

### 5.2 B (plain single session, the baseline)

| Problem | Evidence |
|---|---|
| Does not continue after a normal restart | Session history was restored, but the task was not continued |
| Stop requests wait in the ordinary queue | Read by the model up to 112 s after arrival |
| Backlog left unprocessed at the deadline | Under bursty load, 43 / 22 events were never read |
| "Invents" tasks under pressure | Round 1 BM04: the background material stated plainly that no work was delegated, yet B computed and replied to a task that had not been posted yet; a repository-wide check found 7 such premature replies, plus a reply in the wrong thread |
| No persistent event deduplication | In round-2 BM07, each of 14 replayed same-ID events triggered another model call (¥0.23 total); recognizing replays was left to the model |
| Keeps spending after the deadline | Round 1 BM04: 45 more model calls after the deadline (¥4.18) |
| Never uses its subagent tool | All 37 sessions offered the tool; it was never called |

---

## 6. Open questions the article leaves

These are points we noticed while reading the article and the prototype documentation that the article does not answer. **This is our analysis, not an experimental result.**

**Architecture and scheduling**
1. **No safeguards described for batching**: how many events go into one batch, whether each event gets its own ID and decision, whether urgent events can jump the queue. Large batches can dilute attention, mix up threads, and let one bad event spoil the whole batch.
2. **Information gap between Router and Worker**: a Worker may already have acted in an external system while the Router does not know yet, so a user asking the Router gets stale state.
3. **The recall chain can break**: if a link between objects was never recorded, it cannot be recovered later, for example when a user asks about an old task through a different channel.
4. **Router restatement loses facts**: limiting chat replies to a few words pushes the Router to compress, but the article has no mechanism for passing key facts through verbatim. We observed altered numbers (problem 1).

**Asynchrony and cancellation**
5. **After going async, "wait for the result" relies on the model's discipline**: nothing guarantees the model waits when a later step depends on a background result.
6. **Stopping depends entirely on the model's judgment**: there is no code-level time limit, and the exact semantics of a stop (how far it stops, whether it replies) are not specified.

**Recovery and supervision**
7. **The recovery mechanism is not spelled out**: the article says state is persisted and containers are replaceable, but not how unfinished work is found and resumed. In our test a normal restart resumed, but a SIGKILL was blocked by the lock.
8. **The Watchdog's boundaries are undefined**: who supervises the Watchdog, what happens if it dies, how its power to change the main system's rules at runtime is audited and rolled back, how "real progress" is judged, and how its detection and false-alarm rates are measured.

**Security**
9. **No design for "use credentials without touching them"**: the article itself acknowledges the problem; in our test Workers could indeed read the credential.
10. **The local daemon's permission boundary is undefined**: what the cloud may and may not have it execute.
11. **Cost and approval boundaries of acting in advance (silent fixes)**: unrequested actions carry more risk in a team setting and incur cost nobody sees.

**Experiments**
12. **The article's experiments measure speed, not quality**: coding work was replaced by a tool that only consumes time, so errors such as the middle layer misstating results cannot show up.
13. **The key comparison lacks data**: completion times for a single session with a "delegate first" prompt are not given, and each configuration ran only 5 times with no variance reported.
14. **Tokens are reported, money is not**: whether "more tokens for faster completion" pays off depends on how cached reads are priced.
15. **Some metrics are unclear**: for example, "amortized inference time per decision" exceeds the single-request time at batch size 1, and the calculation is not explained.

---

## 7. Article claims vs. observations

| Design element | What we observed | Assessment |
|---|---|---|
| Two layers improve delivery and stability | Delivery quality, cost and message volume were all worse than the single session; delegation stability was not tested separately | **Not supported** on delivery quality |
| A lighter model suffices for the Router | Both groups used the same model | Not tested |
| Two-stage filtering (code, then model) | Noise and replays were stopped at ingress | **Holds** |
| Merging the backlog into one call (batching) | Every Router dispatch under this load had a batch size of 1 | **Did not appear** under this load |
| Direct delivery relieves the Router | The prototype has no switch to disable it | Not ablatable |
| Async start / stop keeps the conversation responsive | Foreground commands moved to the background after 5 s (7 observations); cancellation untested or failed under the frozen rules | Non-blocking **holds**; cancellation **not validated** |
| Low-interruption chat behaviour | S posted about twice as many messages as B, with duplicates | **Not supported** |
| Persistent, recoverable state | Normal restart resumed automatically; SIGKILL restart blocked by the lock | Normal restart **holds**; crash recovery **not validated** |
| External Watchdog | Not present in the public prototype | Not implemented / not tested |
| The article's own scheduling experiments | No executable benchmark entry point in the public release | Not reproducible |
| Security risks (injection, credentials) | One case read a decoy secret; Workers could read the credential | Risk confirmed |

---

## 8. Lessons for building similar systems (our view)

These are engineering suggestions drawn from our observations, not experimental results:

1. **Make a single session the default path**; enable multiple Workers only when work truly needs parallelism or long-term tracking.
2. **Pass key facts through verbatim**: numbers, links and PR identifiers should come straight from the Worker's structured output, not be regenerated by an intermediate layer.
3. **Give clarification an explicit state** (needs input, the specific question, awaiting reply); a promise like "I'll confirm" is not a clarification.
4. **Route all external publishing through one exit** that deduplicates and checks state, instead of relying on a convention that only the Router replies.
5. **Tie follow-ups to the unfinished original requirement**, so the latest input does not replace the whole acceptance intent.
6. **Give every input a source and a "consumed" receipt**: a `user` role in the model context is not necessarily a human, and "received" is not "read".
7. **Use a separate control channel for cancellation**, not the ordinary message queue; tag jobs with stable task IDs and confirm the real terminal state.
8. **Confirm recovery in layers**: the process is up, the lock and database are available, and the task has actually resumed are three different facts. Locks need observable waiting with bounded retries, and external side effects (PRs, comments) must be reconciled to avoid duplicates.
9. **Make failures visible**: a preparation failure or a stall needs an explicit failed state and an explanation, not just an acceptance note.
10. **Keep credentials out of anything an agent can read**: broker them at the tool boundary instead of relying on prompts to forbid reading them.
11. **Keep a per-request billing receipt**; list shared-session cost separately and never fill unknown usage with zero.
12. **Control concurrency and budget by actual model calls**, not only by the number of logical tasks.

---

## 9. Limitations and untested items

- One model only; the findings may not generalize to other models.
- A small self-built task set, not a public benchmark such as SWE-bench; no comparison to public leaderboards is possible.
- Small samples and no significance tests.
- S and B differ in prompts, tools and integration; this is a **whole-system comparison**, and the differences cannot all be attributed to the two-layer structure.
- Rounds 1 and 2 used somewhat different protocols (deadline rules, background polling, a git configuration fix), so their samples are not pooled.
- **Not tested**: real open-source tasks and public baselines; capability comparison with other models; privilege cases needing a non-allow-listed identity (12 in BM03, 3 in BM09); lock-expired crash recovery; whole-host restart; multi-task recovery order; exactly-once external actions; the 30-minute background timeout and three-retry exhaustion; human reading time and interruptions; upgrade and rollback; the article's own scheduling experiments.
- Costs are proxy estimates, not reconciled with a provider invoice.

---

## 10. Data and figures

| File | Contents |
|---|---|
| `data/summary.json` | Round-2 setup, per-benchmark results, cost totals and untested items |
| `data/round1_runs.csv` | Round 1 results, requests, tokens and cost per test, including invalid runs |
| `data/bm01_runs.csv` | BM01 per run: task, condition, round, frozen and functional pass, PR quality, driver time, known cost |
| `data/bm04_rounds.csv` | BM04 per round: correct count, median reply time, messages, cost |
| `data/bm04_detail.csv` | BM04 scheduling, queueing, concurrency overlap and stop-request details |
| `data/bm06_faults.csv` | BM06 result and notes for each fault and condition |
| `data/bm09_cases.csv` | BM09 per case: carrier, attack, exposure, refusal, classification |
| `data/round2_other_benchmarks.csv` | Per-round results for BM03, BM05 and BM07 |
| `data/cost_by_group.csv` | Round-2 requests and cost by benchmark and condition |
| `data/cost_by_role.csv` | Tokens and cost by role (single session, Worker, Router) |
| `figures/*.svg` | The 5 figures in this report (light and dark mode) |

The data is sanitized: account names, repository names, PR numbers, local paths and endpoint addresses have been removed. Experiment scripts and raw traces are not published at this time.

---

## 11. Notes

- The experiments were executed and logged by an AI coding agent (OpenAI Codex) following a frozen plan. This report was prepared by pans1es with the help of an AI assistant (Anthropic Claude), and the key numbers were checked against the experiment records.
- The original article and prototype belong to their authors. This report only summarizes and cites them; please refer to the originals.
- The text, figures and data in this report are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
