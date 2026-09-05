---
title: "Kanban Swarm 机制解读:Hermes 多智能体如何用一张 SQLite 任务板协作"
date: "2026-09-05"
tags: ["hermes-agent", "kanban-swarm", "multi-agent", "ai-engineering"]
author: MTdaibo
---

当你在本地同时跑着 researcher、writer、reviewer、publisher 好几个 LLM profile 来协作一篇博客时,最头疼的问题通常不是模型不够聪明,而是:它们怎么知道彼此在干什么、谁该接谁的活、出了问题怎么不互相踩踏?Hermes 的 Kanban Swarm 给出的答案朴素得有点意外——把一切都压到一张 SQLite 任务板上。

## 一张 SQLite 板撑起的多智能体

Kanban Swarm 本质上是一个 SQLite-backed 的持久任务板,跨所有 Hermes profile 共享:任务就是板上的行,任何 profile(或人)都能读、能写,数据刻意做成 profile 无关。它不上 RPC、不建内存消息总线,协作的唯一载体是你机器上一个 `.db` 文件。

板的状态机(当前实现)是九个状态:triage → todo → scheduled → ready → running → blocked → review → done,外加 archived。关键的"所有权分离"让状态不会乱跳:dispatcher 只拥有 ready→running,worker 只拥有 running→done|blocked,而 blocked→ready 由人或对等 agent 恢复。谁踢谁一脚,边界是清晰的。

dispatcher 内嵌在 gateway 进程里长驻循环调度(默认每 60 秒一个 tick):回收 stale 任务、提升就绪任务、原子认领,然后为每个 assignee 派生一个独立 Hermes 进程作为 worker。

把一张卡走一遍大概是这样:orchestrator 建一张带 assignee 的卡(比如 researcher),它先落在 todo;若它的父任务都已 done,dispatcher 下个 tick 用 recompute_ready 把它提升为 ready(事件 promoted);随即 dispatcher 用 claim 把 ready 换成 running 并记录本次 run;派生的 researcher worker 第一件事是 `kanban_show` 读整卡上下文,干活,最后 `kanban_complete` 把卡置为 done——于是一条 parent link 打通,下游的 writer 子卡从 todo 被 promote 成 ready。卡走到 done 后,若配了预建的下游 review 子卡,评审批次就会接手。整条链路没有任何进程内共享状态,靠的全是板上那一行。

## 角色不是内置概念,是约定

读者可能期待系统里有张"角色注册表",但 Hermes 刻意没有。dispatcher 不认识任何角色名——板结构里没有 role 字段。角色完全是用户态约定,通过两个已有原语表达:profile(工具集限制)和 skill(行为引导)。比如 orchestrator profile 的 toolsets 被限制为 `[kanban, gateway, memory]`,从工具层面就杜绝它"顺手把活干了";reviewer 则由 dispatcher 在任务进入 review 时,用内置的 sdlc-review skill 派生。

worker 之间没有共享会话记忆。dispatcher spawn 出来的 worker 是全新进程,看到的上下文按序是:任务标题、任务 body、全部评论、每个父任务的完成结果(summary+metadata)、自己 profile 的 skills/memory。这是设计的关键:没有隐藏通道,没有带外状态,"如果它不在 `hermes kanban show <id>` 上,worker 也看不见"。跨任务接力全靠 parent link——子任务上下文里那条 "## Parent task results" 区,原样带着父任务的完成摘要。你正在读的这篇,就是这条流水线的一环。

## 核心机制逐条看

让开发者真正读懂,得看状态机怎么被驱动起来。挑几条关键的:

**原子认领(claim)**。并发下怎么保证同一任务不被两个 worker 重复执行?答案是 compare-and-swap:

```sql
BEGIN IMMEDIATE;
UPDATE tasks
SET status='running', claim_lock=?, claim_expires=?, started_at=?
WHERE id=? AND status='ready' AND claim_lock IS NULL;
```

这个事务用 SQLite WAL + BEGIN IMMEDIATE 串行化写者,检查受影响行数:1 就是胜出,0 就是输了。认领默认 TTL 15 分钟,claim_lock 形如 `<host>:<pid>:<uuid>`。存活但跑得慢的 worker,认领会自动延长而不是被杀——只有 PID 真正消失才被回收。

**依赖与 promote**。child 停留在 todo,直到所有 parent 都 done,dispatcher 的 recompute_ready 才把它提升为 ready。依赖既是调度闸门,也是上下文交接通道。

**review 流**。`kanban_request_review` 把任务移入 review,这是一等状态,官方特意强调"这不是一个 block"。评审者三选一:批准→complete;要求返工→`request_changes`(任务回 ready,不占 block-loop 计数);真正外部阻塞→block。这里有个重要区分:request-review 是任务流内的检查环节,block 是外部升级——前者不是故障。官方教程里一次完整同卡评审往返长这样:run 历史是 review_requested → changes_requested → review_requested → completed,每次尝试各有 actor/summary/metadata/outcome;第二轮实现的 worker 能在自己的 worker_context 里看到评审者拒掉的证据——这正是依赖自包含上下文的好处。

**失败恢复**。spawn 失败连续计数到 failure_limit(默认 2)就 gave_up 并自动 block;max_runtime 超时先 SIGTERM 再 SIGKILL 后重派;stale 回收是"超过 4 小时且最近 1 小时无心跳"则收回重派但**不计失败计数**——那是 dispatcher 侧的缺席检测,不是 worker 的错。依赖阻塞(kind='dependency')自动路由到等父任务,父完成即恢复,无需人介入。还有一些边角:claim 簿记损坏但无活 worker 的孤儿卡会被 reconciled 回 ready(可配 reconcile_orphans);上一 run 是配额/429 错误、刚成功过、或近期评论里已带 PR 链接时,respawn guard 让 dispatcher 本 tick 先不重派,免得 worker 风暴。

**调度**。priority 只是 ORDER BY 的 tiebreaker——官方明说"没有比这更聪明的路由了"。每个任务恰好一个 assignee;想延迟,用 `--scheduled-at`。

**工作区**。三类:scratch(任务专用临时目录,完成即删);dir(既有共享目录,绝对路径强制,防 confused-deputy);worktree(git worktree,隔离提交)。用 `--project` 锚定到项目主仓库时,工作区落在 `<repo>/.worktrees/<task-id>`,分支是确定性的 `<project-slug>/<task-id>`。

**heartbeat 与 goal_mode**。长操作要 `kanban_heartbeat` 报活,可能超 1 小时的任务至少每小时一次,否则有被 stale 回收的风险;`--goal` 则把开放式的"一直做到 X 成立"的卡放进 goal 循环,由辅助 judge 每轮对照标题+body 研判是否完工,预算(默认 20 轮)耗尽就转人工而不是静默退出。

## Swarm v1 图拓扑

官方文档里还有一种单命令生成的复合拓扑 `hermes kanban swarm`:一条命令原子地产出一张四层图——一个完成的 root/blackboard 卡、N 个并行 worker 卡、一个 gated on all 的 verifier 卡、一个 gated on verifier 的 synthesizer 卡。黑板上下文以结构化 JSON 评论存在 root 卡上,供所有 worker 读取。CLI 是一行:`hermes kanban swarm "goal" --worker researcher:task-title:skill1,skill2 --verifier reviewer --synthesizer writer`;`--worker` 可重复,于是你一次就能把"并行研究 + 一个总把关 + 一个总聚合"整张图落下来,不用手工 create+link。它与流水线(pipeline)的区别在于:流水线是可任意手工组合的"链"(scout→editor→writer),而 swarm 是"一键生成、固定形状"的分层门控。

## 对 AI 开发者的启示

最后回到可借鉴的工程要点。

一是**任务自包含**。worker 无共享内存的前提下,把"够做成这件事的一切"都塞进任务卡,是最便宜也最可靠的上下文传递方式——比任何进程内共享状态都稳。

二是**原子状态机**。用一条 SQL 把一个"从 ready 到 running"的竞态消解掉,比写一堆分布式锁轻太多。所有状态迁移都有明确所有者,状态机像协议一样清晰。

三是**失败恢复是一等公民**。不是"出错就报异常",而是 spawn 失败有重试、超时有回收、消失有崩溃检测、依赖有自动恢复——每个坑都预先铺了一条路。

四是**低成本模型分工**。官方明确建议:拆解任务需要前沿级判断(planner/orchestrator 配 frontier 模型),而执行一张已经写清目标、上下文、交接证据的卡通常不需要——token 大头在 worker 上,所以 worker 配便宜模型。你正在读的这篇,上游 researcher 就跑在 flash 级模型上。

五是**过程可回放**。每个 worker 的一次尝试都记成一条 run(actor/summary/metadata/outcome),整卡的历史就是一条可审计的时间线;返工时,新 worker 能在自己卡上下文里看到上一次的完整记录——让"交接"不是口头传话,而是板上留有结构化凭据。

一句话收尾:多智能体系统最难的不是让模型变聪明,而是把"协作"降级成"对一张共享表做确定性操作"。状态机、原子认领、失败回收——这些全是数据库的老本行,却恰好是 agent 编排最缺的确定性。
