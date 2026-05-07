---
title: Spring AI 和 LangGraph 怎么搭在一起：我的 Agent 分层设计
date: 2026-04-10
excerpt: 这篇从学生项目视角整理 Spring AI 与 LangGraph 的职责边界，以及我会怎么设计一个 Java 后端里的自动化运维 Agent。
tags: [Spring AI, LangGraph, Agent, Java, 系统设计]
---

我一开始看 Spring AI 和 LangGraph 的时候，有点分不清它们的关系。好像两个都能做 Agent，但又不是同一种东西。后来梳理了一下，我现在更愿意这么理解：

Spring AI 解决的是“Java 应用怎么接入大模型能力”，LangGraph 解决的是“复杂 Agent 的执行流程怎么组织”。

这个区别挺重要。因为如果只用 Spring AI，可以很方便地调模型、做 RAG、做工具调用；但复杂任务可能还是会写成一大段逻辑。如果只看 LangGraph，又会发现它更关注状态图和执行流程，具体模型调用、工具封装、权限和日志还是要后端系统自己做。

所以我觉得它们不是替代关系，而是可以分层组合。

## Spring AI 更像能力层

Spring AI 对 Java 后端比较友好，因为它的风格很 Spring。比如：

- `ChatClient` 用来和模型对话。
- `Advisor` 可以做记忆、RAG、日志这类横切逻辑。
- `VectorStore` 抽象向量数据库。
- Tool Calling 可以把 Java 方法封装成模型可调用工具。
- Streaming 可以做 SSE 流式输出。

如果我做自动化运维 Agent，Spring AI 这层主要负责“能力”：

```text
模型能力：总结、规划、分类、生成报告
检索能力：查知识库、查历史故障
工具能力：查日志、查指标、查发布记录
记忆能力：保留多轮对话上下文
观测能力：记录调用链路和耗时
```

它让 Java 项目接 AI 的成本低很多，不用每个模型供应商都手写一套调用代码。

## LangGraph 更像流程层

LangGraph 的重点是状态图。也就是把一个复杂任务拆成节点，然后根据状态决定下一步走哪条边。

比如一个告警排查任务可以是：

```text
接收告警
 -> 生成排查计划
 -> 查日志
 -> 查指标
 -> 查发布记录
 -> 检索知识库
 -> 排序根因
 -> 是否需要人工确认
 -> 生成报告
```

这里每一步都可以是一个节点。节点不一定都调用大模型，有些只是普通后端逻辑。比如查日志节点本质就是调日志平台接口。

这种方式比一个 while 循环更清楚，因为你知道当前执行到哪一步，也知道失败后可以从哪里恢复。

## 我会怎么分层

如果让我设计一个 Java 后端里的运维 Agent，我会分成几层：

```text
API 层
Agent Orchestrator 层
AI Capability 层
Tool Service 层
Storage 层
```

API 层负责接收用户问题或告警事件，也负责 SSE 输出。比如前端问一句“订单服务为什么超时”，后端先创建一个会话和工作流实例。

Agent Orchestrator 层负责状态图。它不直接写太多业务逻辑，而是决定当前节点是什么、节点执行后下一步去哪。

AI Capability 层封装 Spring AI，比如 ChatClient、RAG、Tool Calling、Memory Advisor。

Tool Service 层封装真实系统能力，比如日志平台、指标平台、发布平台、知识库、工单系统。

Storage 层保存会话、消息、checkpoint、工具调用记录、审计日志。

这样分层以后，模型能力和业务工具不会混在一起，后面维护会舒服很多。

## State 应该设计得具体一点

Agent 的 State 不能只有 messages。我会把它设计成一个比较明确的 JSON：

```json
{
  "conversationId": "c-1001",
  "userId": "u-1001",
  "alert": {
    "service": "order-service",
    "level": "P1",
    "message": "接口超时率升高"
  },
  "plan": [],
  "evidence": [],
  "hypotheses": [],
  "toolCalls": [],
  "riskLevel": "LOW",
  "approval": null,
  "finalReport": null
}
```

每个节点只更新自己负责的字段。比如 Planner 写入 `plan`，查日志节点写入 `evidence`，根因排序节点写入 `hypotheses`，人工确认节点写入 `approval`。

这样做的好处是，状态可读、可保存、可恢复，也方便前端展示进度。

## 哪些节点需要模型，哪些不需要

不是所有节点都应该调用大模型。这个点我觉得很容易被忽略。

适合调用模型的节点：

- 根据告警生成排查计划。
- 根据证据总结可能根因。
- 把工具结果整理成人能读懂的报告。
- 对用户追问做上下文理解。

不适合或者不需要调用模型的节点：

- 查数据库。
- 查日志。
- 查指标。
- 判断权限。
- 写 checkpoint。
- 做幂等校验。

能用确定性代码解决的，就不要交给模型。模型适合处理模糊语义和总结，不适合当权限系统或事务系统。

## 风险控制要放在架构里

Agent 如果只能聊天，风险还好；如果能调用工具，风险就上来了。

我会把工具按风险分级：

```text
READ_ONLY：查日志、查指标、查知识库
LOW_RISK_WRITE：创建草稿工单、生成报告
HIGH_RISK_WRITE：重启服务、修改配置、切换流量
```

Orchestrator 根据工具风险决定是否需要人工确认。比如只读工具可以自动执行，高风险写工具必须进入 `human_review` 节点。

权限也必须在后端校验，不能靠 prompt。用户没有权限查某个服务的日志，模型传了那个服务名也要拒绝。

## 这个设计和简历项目的关系

如果在简历里写“基于 Spring AI 实现自动化运维 Agent”，面试官可能会继续问：Agent 怎么规划？工具怎么调用？怎么避免乱调？失败怎么恢复？怎么记录过程？

这时候如果只回答“用了大模型和 RAG”，会显得比较浅。更好的回答是：

- Spring AI 负责模型调用、RAG、Tool Calling 和流式输出。
- 状态图负责长流程编排。
- checkpoint 支持失败恢复。
- 工具按风险分级，高风险动作需要人工确认。
- 工具调用有权限校验、幂等和审计日志。

这套回答就更像后端项目，而不是简单套壳。

## 小结

Spring AI 和 LangGraph 的组合，我理解成“能力层 + 流程层”。Spring AI 让 Java 后端可以方便接入模型、知识库和工具；LangGraph 的状态图思想让复杂 Agent 变得可控、可恢复、可审计。

作为学生项目，不一定要把所有能力都做得像生产系统一样完整，但架构上要能讲清楚边界。哪些交给模型，哪些交给代码，哪些需要人工确认，哪些需要持久化，这些问题想清楚以后，Agent 项目才不会只是一个聊天 Demo。
