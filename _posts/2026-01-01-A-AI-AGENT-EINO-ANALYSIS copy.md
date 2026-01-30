---
layout:     post
title:      AI agent开发入门：基于eino开发agent（一）
subtitle:   
date:       2026-01-01
author:     pandaychen
catalog:    true
tags:
    - AI
---


##  0x00    前言

##  0x01    AI 基础

####    提示词
1、系统提示词（System Prompt）

-   主要作用于 ReAct 模式的 Reason（推理/思考） 步骤
-   概念：给 LLM 的出厂设置。它定义了 Agent 的角色、性格、知识边界、遵循的协议（如必须以 JSON 输出）以及如何使用工具
-   类比：就像给演员的剧本大纲，规定了是一个专业的 Golang 架构师还是一个调皮的聊天机器人


提示词的作用：

-   控制幻觉：通过系统提示词强制模型在没有把握时必须调用 search_tool，而不是胡编乱造
-   结构化输出：ReAct 依赖解析模型的 Action 字段。如果系统提示词写得不好，模型输出格式乱了，你的 `ToolsNode` 就无法解析出要执行哪个函数

####    消息模版（推荐模版）：Chat Template
在 Eino 框架中对应 `ChatTemplate` 组件，它是一种包含占位符（变量）的字符串。由于模型输入通常是动态的（比如不同的用户问题、不同的上下文），模板允许你安全地注入这些变量。如**你现在的任务是处理用户关于 {{.topic}} 的提问，当前时间是 {{.time}}**

####    react模式下的应用
ReAct 的循环是**思考 (Thought) -> 行动 (Act) -> 观察 (Observation)**

1、作用于 Thought（推理）环节时，用于注入推理框架，在 ReAct 模式下，系统提示词必须包含特定的指导语，如

```
你必须按以下格式思考：Thought: 思考用户的问题。 Action: 选择工具。 Action Input: 工具参数。 Observation: 工具返回的结果... (以此类推)
```

模板的作用：通过模板，可以将用户的实时 Query 和查到的历史状态（State）拼接到一起送给模型，驱动模型产生第一次 Thought

2、作用于 Observation 后的再推理

当工具返回结果（Observation）后，模板会将结果重新包装，送回模型。此时，系统提示词依然在起作用，对模型约束

```
现在你拿到了工具数据，请判断是继续调用工具还是给出最终回答。
```

消息模版的作用：

-   上下文隔离：模板确保了系统指令和用户数据在 Prompt 里的位置是清晰的，防止用户通过提问（Prompt Injection）恶意修改 Agent 的系统设定

##  0x02    eino基础
AI 核心概念和工程范式：

一、 核心 AI 基础概念

-   ChatModel（对话模型）： Agent 的大脑。需要了解 Chat Completion API 的基本结构（如 System Message 设定角色、User Message 提问、Assistant Message 回复）
-   Prompt Engineering（提示词工程）： 如何通过文本引导模型。在 Eino 中，会用到 ChatTemplate，需要理解变量注入和模板化 Prompt 的技巧
-   Token（令牌）： 模型的计费和长度单位。需要有上下文长度限制的概念，避免因输入过长导致模型报错
-   Function Calling / Tool Use（工具调用）： 这是 Agent 区别于普通聊天机器人的核心。模型不会真的跑代码，而是输出一个 JSON 结构告知使用者想调用哪个函数。Eino 封装了 ToolsNode 来自动处理这些逻辑

####    Agent 核心架构模式
Eino 也提供了编排（Orchestration）能力，介绍下目前业界主流的几种 Agent 构建模式：

1、ReAct 模式 （Reason + Act）
最经典的 Agent 模式。模型通过：思考（Thought） -> 行动 （Action） -> 观察 （Observation） 的循环来解决问题。在 Eino 中可以构建一个包含 `ChatModel` 和 `ToolsNode` 的 Graph（图），并允许数据在两者之间循环流转

2、 RAG (检索增强生成)

当模型不知道私有数据时，需要先去数据库查一下。这里涉及到知识点：

-   **Embedding**（向量化）： 将文字转为向量
-   **Vector Database**（向量数据库）： 如 Milvus, Pinecone等
-   **Retriever**（检索器）： Eino 中对应的组件，负责从向量数据库捞取相关片段并塞给模型

3、多智能体协作 （Multi-Agent）

Eino 也支持多智能体控制权转移，需要理解主智能体（Host Agent）与子智能体（Sub-Agent）的关系，以及它们之间如何传递上下文（Context）等

####    Eino 框架的术语

-   Graph 编排： Eino 的核心是 Graph（有向图），需要理解节点（Node） 和边（Edge） 的概念
-   流式处理 （Streaming）： AI 模型通常是打字机式输出。Eino 提供了流式合并、分发和转换机制，通常在 Go 中处理 Iter（迭代器）类型的返回结果
-   强类型安全： 与 Python 的 LangChain 不同，Eino 在编译期检查类型，需要习惯为每个节点定义清晰的输入（Input）和输出（Output）结构体

##  参考
-   [eino官方](https://github.com/cloudwego/eino/blob/main/README.zh_CN.md)
-   [实现一个最简 LLM 应用](https://www.cloudwego.io/zh/docs/eino/quick_start/simple_llm_application/)
-   [Agent-让大模型拥有双手](https://www.cloudwego.io/zh/docs/eino/quick_start/agent_llm_with_tools/)
-   [用 Eino ADK 构建你的第一个 AI 智能体：从 Excel Agent 实战开始](https://segmentfault.com/a/1190000047399985)