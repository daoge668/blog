---
title: LangGraph 学习笔记：为什么 Agent 需要状态图
date: 2026-01-14
excerpt: 这篇不是照着文档翻译，而是从一个学生做自动化运维 Agent 的角度，记录我为什么觉得 LangGraph 的状态、节点、边和检查点很重要。
tags: [LangGraph, Agent, 状态机, 工作流, 大模型]
---

我一开始接触 Agent 的时候，脑子里想的其实很简单：模型先分析问题，然后调用工具，拿到结果后继续分析，最后输出答案。也就是大家经常说的 ReAct。这个思路很直观，而且写一个 Demo 也不难。

但后面我在想自动化运维 Agent 的时候，发现一个问题：真实任务不是三五步就结束的。一次排障可能要查日志、查指标、查最近发布、查知识库、比较多个根因，还可能要等人确认。这个时候如果只是写一个 while 循环，让模型自己决定下一步，就会有点不可控。

LangGraph 吸引我的地方就在这里。它不是单纯让 Agent “更聪明”，而是让 Agent 的执行过程更像一个可管理的工作流。

## ReAct 的问题不是不能用，而是不够稳

ReAct 很适合短任务，比如：

```text
用户问：Redis big key 怎么排查？
模型：调用知识库搜索。
工具：返回文档。
模型：总结回答。
```

这种场景下没什么问题。但如果任务变成：

```text
订单服务超时率升高，请定位可能原因并给出处理建议。
```

Agent 可能需要做很多事情：

- 先判断告警等级。
- 查询最近 10 分钟错误日志。
- 查询接口耗时指标。
- 查询数据库连接池指标。
- 查询最近发布记录。
- 检索历史故障文档。
- 根据证据排序根因。
- 生成报告。
- 如果建议重启服务，还要人工确认。

如果所有步骤都放在一个循环里，出了问题很难判断：到底是模型规划错了，还是工具返回错了，还是上下文太长导致模型忘了前面的证据？

## State 不只是聊天记录

LangGraph 里最核心的概念之一是 State。刚开始我以为 State 就是 messages，也就是对话历史。后来发现如果这么理解就太窄了。

对一个运维 Agent 来说，State 应该保存的是“任务继续执行所需的信息”，比如：

```json
{
  "alert": {
    "service": "order-service",
    "level": "P1",
    "message": "接口超时率升高"
  },
  "plan": [
    "查询错误日志",
    "查询数据库连接池指标",
    "查询最近发布"
  ],
  "evidence": [],
  "currentStep": 0,
  "riskLevel": "LOW",
  "finalReport": null
}
```

这样每个节点都不是从一大段聊天记录里猜当前状态，而是直接读结构化字段。这个设计对后端开发很友好，因为它像是在维护一个工作流实例。

## Node 要小一点

我以前写代码也容易犯一个问题：一个方法里塞太多逻辑。Agent 节点也是一样。如果写一个万能节点，让它既规划、又查日志、又总结、又决定是否人工确认，那后面一定不好调。

我会把节点拆得细一点：

- `planner`：只负责生成排查计划。
- `query_logs`：只负责查日志。
- `query_metrics`：只负责查指标。
- `query_deployments`：只负责查发布记录。
- `knowledge_search`：只负责查知识库。
- `rank_hypotheses`：根据证据排序可能根因。
- `human_review`：等待人工确认。
- `reporter`：生成最终报告。

这样做的好处是，每个节点的输入输出都比较清楚。比如 `query_logs` 节点输入服务名、时间范围、关键词，输出日志摘要和原始证据 ID。它不需要知道最后报告怎么写。

## Edge 让流程可控

边决定下一步执行哪个节点。这个东西看起来像流程图，但我觉得它的意义比流程图大，因为它把“控制逻辑”从 prompt 里拿出来了。

比如可以写一些条件：

- 如果日志里出现大量数据库连接超时，就进入 `query_metrics`。
- 如果最近 30 分钟有发布，就进入 `query_deployments`。
- 如果证据不足，就回到 `planner` 补充排查步骤。
- 如果下一步动作是重启、切流、修改配置，就进入 `human_review`。
- 如果根因置信度够高，就进入 `reporter`。

这比完全相信模型自由决策更稳。模型还是可以参与判断，但它的判断被状态图限制在合理范围内。

## Checkpoint 是我觉得最工程化的点

如果只是课堂作业或者 Demo，失败了重新跑一遍也没关系。但真实系统不行。假设 Agent 已经查了 5 个工具，结果生成报告时服务重启，如果没有 checkpoint，就只能重新查一遍。

这会带来几个问题：

- 重复消耗 token。
- 重复调用日志和指标平台。
- 某些工具调用可能不是幂等的。
- 用户不知道任务执行到哪一步。

所以 checkpoint 很重要。每执行完一个节点，就把 State 保存下来。失败后可以从最近的节点继续执行。

如果我用 Java 后端实现类似能力，可能会设计几张表：

```text
agent_workflow_instance
- id
- conversation_id
- current_node
- status
- created_at
- updated_at

agent_workflow_checkpoint
- id
- workflow_id
- node_name
- state_json
- created_at

agent_tool_invocation
- id
- workflow_id
- tool_name
- request_summary
- response_summary
- cost_ms
- status
```

这套表不一定完美，但至少能支撑恢复、审计和排查。

## Human-in-the-loop 不是为了显得高级

很多 Agent 文章都会提 Human-in-the-loop，但有时候写得像概念。我觉得它在运维场景里是刚需。

比如 Agent 分析后建议：

```text
可以尝试重启 order-service 的两个异常实例。
```

这个动作不能让模型直接执行。更合理的方式是进入人工确认节点：

1. Agent 汇总当前证据。
2. 说明为什么建议重启。
3. 列出可能影响。
4. 等待用户批准、拒绝或修改方案。
5. 根据用户选择继续执行。

这样 Agent 的定位更清楚：它可以帮忙分析、整理证据、提出建议，但高风险操作还是由人负责确认。

## 和 Spring AI 怎么结合

我现在的理解是，Spring AI 更偏“模型能力接入”，LangGraph 更偏“Agent 流程组织”。如果放在 Java 项目里，可以用 Spring AI 做：

- `ChatClient` 调模型。
- Tool Calling 调内部服务。
- RAG 查知识库。
- Advisor 做记忆、日志、权限。

然后用 LangGraph 的思想组织流程：

- State 保存任务上下文。
- Node 封装每一步动作。
- Edge 控制下一步。
- Checkpoint 保存状态。
- Human node 等待人工确认。

这两个东西结合起来，才比较像一个能落地的 Agent 系统。

## 小结

LangGraph 对我最大的启发是：复杂 Agent 不能只靠一个大 prompt 和一个循环。它需要状态、流程、恢复和边界。

如果说 ReAct 适合快速做出一个能用的 Demo，那么状态图更适合把 Demo 往工程系统推进。作为学生项目，我觉得不一定要一上来实现得特别完整，但至少要能讲清楚为什么需要 State、为什么要拆 Node、为什么高风险动作要人工确认。这样项目听起来会比“我接了一个大模型接口”扎实很多。
