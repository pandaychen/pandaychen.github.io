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
参考资料

-   [Eino：Cookbook](https://www.cloudwego.io/zh/docs/eino/eino-cookbook/)

本文源码基于[v0.7.30](https://github.com/cloudwego/eino/blob/v0.7.30)

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
在 Eino 框架中对应 `ChatTemplate` 组件，它是一种包含占位符（变量）的字符串。由于模型输入通常是动态的（比如不同的用户问题、不同的上下文），模板允许你安全地注入这些变量。如**你现在的任务是处理用户关于 `{{"{{"}}.topic{{"}}"}}` 的提问，当前时间是 `{{"{{"}}.time{{"}}"}}`**

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

####    Eino架构
![eino_structure.png]()

####    Eino组件列表
Eino 应用的基本构成元素是功能各异的组件，就像足球队由不同位置角色的队员组成：

![]()

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
-   强类型安全： Eino 在编译期检查类型，需要习惯为每个节点定义清晰的输入（Input）和输出（Output）结构体

####    Eino: 核心模块（开发）

1、Components 组件，抽象出来的大模型应用中常用的组件，例如 `ChatModel`、`Embedding`、`Retriever` 等，这是实现一个大模型应用搭建的积木，是应用能力的基础，也是复杂逻辑编排时的原子对象

2、Chain/Graph 编排，多个组件混合使用来实现业务逻辑的串联

3、Flow 集成工具（agents），将常用的大模型应用模式封装成简单、易用的工具，目前提供了 ReAct Agent 和 Host Multi Agent

4、ADK（Agent Development Kit）

####    Eino：component
[Components 组件](https://www.cloudwego.io/zh/docs/eino/core_modules/components/)

大模型应用开发和传统应用开发的区别在于，大模型开发更强调：

1.  基于语义的文本处理能力：能够理解和生成人类语言，处理非结构化的内容语义关系
2.  智能决策能力：能够基于上下文进行推理和判断，做出相应的行为决策

伴随三种主要的应用模式：

1.  直接对话模式：处理用户输入并生成相应回答
2.  知识处理模式：对文本文档进行语义化处理、存储和检索
3.  工具调用模式：基于上下文做出决策并调用相应工具

####    Eino：编排
[Chain & Graph & Workflow 编排功能](https://www.cloudwego.io/zh/docs/eino/core_modules/chain_and_graph_orchestration/)

在大模型应用中，Components 组件是提供原子能力的最小单元，比如：

-   `ChatModel`：提供了大模型的对话能力
-   `Embedding`：提供了基于语义的文本向量化能力
-   `Retriever`：提供了关联内容召回的能力
-   `ToolsNode`：提供了执行外部工具的能力

一个大模型应用，除了需要这些原子能力之外，还需要根据场景化的业务逻辑，对这些原子能力进行组合、串联，这就是编排。此外，大模型应用的典型特征是自定义的业务逻辑本身不会很复杂，几乎主要都是对原子能力的组合串联

Eino 提供了**基于 Graph 模型 (node + edge) 的，以组件为原子节点的，以上下游类型对齐为基础的编排**的解决方案。Graph 本身是强大且语义完备的，可绘制出所有的数据流动网络（如 分支、并行、循环等）。但 Graph 并不是没有缺点的，基于点、边模型的 Graph 在使用时，要求开发者要使用 `graph.AddXXXNode()` 和 `graph.AddEdge()` 两个接口来创建一个数据通道，强大但是略显复杂。而在现实的大多数业务场景中，往往仅需要按顺序串联即可，因此，Eino 封装了接口更易于使用的 Chain。Chain 是对 Graph 的封装，除了环之外，Chain 暴露了几乎所有 Graph 的能力

####    Eino：Flow集成
对大模型应用中通用场景和模式的抽象，目前 Eino 已经集成了 react agent、host multi agent 两个常用的 Agent 模式，以及 MultiQueryRetriever, ParentIndexer 等Flow

[Flow 集成](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/)

##  0x  多智能体的应用

####     ReAct 单智能体
基本的 ReAct 构建的单智能体，是由一个 Agent 既负责计划拆解，也负责 Function Call，如下：

![react-agent-1]()

在业务场景中，可能存在的问题如下：

-   问题一：对 LLM 的要求高：既要擅长推理规划，也要擅长做 Function Call
-   问题二：LLM 的 prompt 复杂：既要能正确规划，又要正确的做 Function Call，还要能输出正确的结果
-   问题三：没有计划：每次 Function Call 之后，LLM 需要重新推理，没有整体的可靠计划

由此，AI提供了multiAgent（多智能体）来解决上述问题

####    multiAgent
如上，解决的思路，首先是把单个的 LLM 节点拆分成两个，一个负责计划，一个负责执行。这样就解决了上面的问题三，Planner 会给出完整计划，Executor 依据这个完整计划来依次执行。即**Planner 只需要擅长推理规划，Executor 则需要擅长做 Function Call 和总结，各自的 prompt 都是原先的一个子集**。但同时带来一个新的问题，缺少纠错能力：最开始的计划，在执行后，是否真的符合预期、能够解决问题？

因此在 Executor 后面增加一个 LLM 节点，负责反思和调整计划，如下图

![plan_execute-1]()

这样就彻底解决了上面列出的问题，Executor 只需要按计划执行 Function Call，Reviser 负责反思和总结

此计划--执行模式的多智能体，通过将任务解决过程拆解为负责计划的 Planner 和 Reviser，以及负责执行的 Executor，实现了智能体的单一职责以及任务的有效计划与反思，同时也能够充分发挥 DeepSeek 这种推理模型的长项、规避其短板（Function Call）

其他细节参考[此文](https://www.infoq.cn/article/zhgae6llqwo9cjwr9lks), 场景是实现让 DeepSeek 做（指挥）Function Call的能力，即**计划--执行**多智能体的协同范式，由计划智能体负责推理和生成计划，由执行智能体负责执行计划

-   由 DeepSeek 负责指挥
-   由擅长 Function Call 的其他大模型LLM去听指挥进行函数调用

![plan-execute-replan-1]()

优点如下；

-   专业的智能体干专业的事情：比如 DeepSeek 负责推理和计划，豆包大模型负责 Function Call
-   智能体层面的单一职责原则：每个智能体的职责是明确的，解耦的，方便 Prompt 调优和评测
-   在提供解决问题整体方案的同时，保持灵活性：符合人类解决问题的通用模式

####    参考实现
[主题乐园行程规划助手](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute)，功能是用 Eino 实现基于 DeepSeek 的计划--执行多智能体。这个多智能体的功能是根据用户的游园需求，规划出具体、符合要求、可操作的行程安排

TODO

##  0x0 eino组件：Message
在 Eino 中，对话是通过 `schema.Message` 来表示的，即对一个对话消息的抽象定义，`Message` 字段如下：

-   `Role`：消息的角色（可选）
    -   `system`：系统指令，用于设定模型的行为和角色
    -   `user`：用户的输入，`schema.UserMessage`
    -   `assistant`：LLM大模型的回复，`schema.AssistantMessage`
    -   `tool`：工具调用的结果
-   `Content`：消息的具体内容

####    []*Message VS *Message的使用场景

TODO

##  0x0 组件：ChatTemplate
[文档](https://www.cloudwego.io/zh/docs/eino/core_modules/components/chat_template_guide/)

`ChatTemplate`用于创建对话模板并生成消息，Eino 提供了如下**模板化功能来构建要输入给大模型的消息**：

-   FString：Python 风格的简单字符串格式化（例如：`你好，{name}！`）
-   Jinja2：支持丰富表达式的 Jinja2 风格模板（例如：`你好，{{"{{"}}name{{"}}"}}！`）
-   GoTemplate：Go 语言内置的 `text/template` 格式（例如：`你好，{{"{{"}}.name{{"}}"}}！`）
-   消息占位符：支持插入一组消息（如对话历史）

##  0x0 组件：Tool
[Tool](https://www.cloudwego.io/zh/docs/eino/core_modules/components/tools_node_guide/) 是 Agent 的执行器，提供了具体的功能实现。每个 Tool 都有明确的功能定义和参数规范，使 `ChatModel` 能够准确地调用它们

####    Tool创建（N种方式）
1、方式一：使用 `NewTool` 构建，这种方式适合简单的工具实现，通过定义工具信息和处理函数来创建 Tool

```go
import (
    "context"

    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/components/tool/utils"
    "github.com/cloudwego/eino/schema"
)

// 处理函数
func AddTodoFunc(_ context.Context, params *TodoAddParams) (string, error) {
    // Mock处理逻辑
    return `{"msg": "add todo success"}`, nil
}

func getAddTodoTool() tool.InvokableTool {
    // 工具信息
    info := &schema.ToolInfo{
        Name: "add_todo",
        Desc: "Add a todo item",
        ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
            "content": {
                Desc:     "The content of the todo item",
                Type:     schema.String,
                Required: true,
            },
            "started_at": {
                Desc: "The started time of the todo item, in unix timestamp",
                Type: schema.Integer,
            },
            "deadline": {
                Desc: "The deadline of the todo item, in unix timestamp",
                Type: schema.Integer,
            },
        }),
    }

    // 使用NewTool创建工具
    return utils.NewTool(info, AddTodoFunc)
}
```

方式一的缺点，需要在 `schema.ToolInfo` 中手动定义参数信息（`ParamsOneOf`），和实际的参数结构（`TodoAddParams`）是分开定义的。这样不仅造成了代码的冗余，而且在参数发生变化时需要同时修改两处地方，容易导致不一致，维护起来也比较麻烦

2、方式二：使用 `InferTool` 构建，此方式更加简洁，通过结构体的 `tag` 来定义参数信息，就能实现参数结构体和描述信息同源，无需维护两份信息

```go
// 参数结构体
type TodoUpdateParams struct {
    ID        string  `json:"id" jsonschema:"description=id of the todo"`
    Content   *string `json:"content,omitempty" jsonschema:"description=content of the todo"`
    StartedAt *int64  `json:"started_at,omitempty" jsonschema:"description=start time in unix timestamp"`
    Deadline  *int64  `json:"deadline,omitempty" jsonschema:"description=deadline of the todo in unix timestamp"`
    Done      *bool   `json:"done,omitempty" jsonschema:"description=done status"`
}

// 处理函数
func UpdateTodoFunc(_ context.Context, params *TodoUpdateParams) (string, error) {
    // Mock处理逻辑
    return `{"msg": "update todo success"}`, nil
}

// 使用 InferTool 创建工具
updateTool, err := utils.InferTool(
    "update_todo", // tool name 
    "Update a todo item, eg: content,deadline...", // tool description
    UpdateTodoFunc)
```

3、方式三：实现 `Tool` 接口，对于需要更多自定义逻辑的场景，可以通过实现 `Tool` 接口来创建

```go
type ListTodoTool struct {}

func (lt *ListTodoTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
    return &schema.ToolInfo{
        Name: "list_todo",
        Desc: "List all todo items",
        ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
            "finished": {
                Desc:     "filter todo items if finished",
                Type:     schema.Boolean,
                Required: false,
            },
        }),
    }, nil
}

func (lt *ListTodoTool) InvokableRun(ctx context.Context, argumentsInJSON string, opts ...tool.Option) (string, error) {
    // Mock调用逻辑
    return `{"todos": [{"id": "1", "content": "在2024年12月10日之前完成Eino项目演示文稿的准备工作", "started_at": 1717401600, "deadline": 1717488000, "done": false}]}`, nil
}
```

4、方式四：eino实现的自带Tool，如`github.com/cloudwego/eino-ext/components/tool/duckduckgo`

####    构建toolsNode
上面定义（实现）`Tool`之后，在构建Agent时，需要把`Tool`转换为`ToolsNode`。`ToolsNode` 是一个核心组件，它负责管理和执行工具调用。`ToolsNode` 可以集成多个工具，并提供统一的调用接口。它支持同步调用（`Invoke`）和流式调用（`Stream`）两种方式，能够灵活地处理不同类型的工具执行需求。要创建一个 `ToolsNode`，需要提供一个工具列表配置：

```go
import (
    "context"

    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
)

conf := &compose.ToolsNodeConfig{
    Tools: []tool.BaseTool{tool1, tool2},  // 工具可以是 InvokableTool 或 StreamableTool
}
toolsNode, err := compose.NewToolNode(context.Background(), conf)
```

##  0x0   组件：ChatModel
`ChatModel` 是 Eino 框架中对对话大模型的抽象，它提供了统一的接口来与不同的大模型服务（如 OpenAI、Ollama 等）进行交互

####    信息流：输出完整消息`generate`和输出消息流`stream`
流式响应，主要用于提升用户体验，即`stream` 运行模式让 `ChatModel` 提供类似打字机的输出效果，使用户更早得到模型响应

可以看到，`Generate`/`Stream`方法的输入都为`[]*schema.Message`，输出都为`*schema.Message`

```go
type BaseChatModel interface {
	Generate(ctx context.Context, input []*schema.Message, opts ...Option) (*schema.Message, error)
	Stream(ctx context.Context, input []*schema.Message, opts ...Option) (
		*schema.StreamReader[*schema.Message], error)
}

// Deprecated: Please use ToolCallingChatModel interface instead, which provides a safer way to bind tools
// without the concurrency issues and tool overwriting problems that may arise from the BindTools method.
type ChatModel interface {
	BaseChatModel

	// BindTools bind tools to the model.
	// BindTools before requesting ChatModel generally.
	// notice the non-atomic problem of BindTools and Generate.
	BindTools(tools []*schema.ToolInfo) error
}
```

##  0x0 组件：

##  0x  组件：ReactAgent

[文档](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/react_agent_manual/)

Eino React Agent 是实现了 React 逻辑 的智能体框架，框架[代码](https://github.com/cloudwego/eino/tree/main/flow/agent/react)


##  0x  组件：Eino ADK
[文档](https://www.cloudwego.io/zh/docs/eino/overview/eino_adk0_1/)

##  0x  核心：编排能力

-   Chain：链式有向图，始终向前，简单。适合数据单向流动，没有复杂分支的场景
-   Graph：有向图，有最大的灵活性；或有向无环图，不支持分支，但有清晰的祖先关系

####    Chain
![simple_template_and_chatmodel.png]()


####    Graph
最多执行一次 ToolCall 的 Agent，其编排图如下：

![eino_practice_graph_tool_call.png]()


####	设计理念
[编排的设计理念](https://www.cloudwego.io/zh/docs/eino/core_modules/chain_and_graph_orchestration/orchestration_design_principles/)


##  0x  入门示例

-   [实现一个最简 LLM 应用](https://www.cloudwego.io/zh/docs/eino/quick_start/simple_llm_application/)
-   [Agent-让大模型拥有双手](https://www.cloudwego.io/zh/docs/eino/quick_start/agent_llm_with_tools/)

####    基础ChatModel 例子：程序员鼓励师
本例子实现一个程序员鼓励师，这个助手不仅能提供技术建议，还能在程序员感到难过时给予心理支持，主要步骤如下：

1.  创建对话模板并生成消息（用户消息）
2.  创建`ChatModel`
3.  运行 `ChatModel`并等待输出结果

核心代码如下：

```go
func createTemplate() prompt.ChatTemplate {
	// 创建模板，使用 FString 格式
	return prompt.FromMessages(schema.FString,
		// 系统消息模板
		schema.SystemMessage("你是一个{role}。你需要用{style}的语气回答问题。你的目标是帮助程序员保持积极乐观的心态，提供技术建议的同时也要关注他们的心理健康。"),

		// 插入需要的对话历史（新对话的话这里不填）
		schema.MessagesPlaceholder("chat_history", true),

		// 用户消息模板
		schema.UserMessage("问题: {question}"),
	)
}

func createMessagesFromTemplate() []*schema.Message {
	template := createTemplate()

	// 使用模板生成消息

    //template.Format 返回的类型是[]*Message
	messages, err := template.Format(context.Background(), map[string]any{
		"role":     "程序员鼓励师",
		"style":    "积极、温暖且专业",
		"question": "我的代码一直报错，感觉好沮丧，该怎么办？",
		// 对话历史（这个例子里模拟两轮对话历史）

        // 构造的数据，会被提交给LLM
		"chat_history": []*schema.Message{
			schema.UserMessage("你好"),
			schema.AssistantMessage("嘿！我是你的程序员鼓励师！记住，每个优秀的程序员都是从 Debug 中成长起来的。有什么我可以帮你的吗？", nil),
			schema.UserMessage("我觉得自己写的代码太烂了"),
			schema.AssistantMessage("每个程序员都经历过这个阶段！重要的是你在不断学习和进步。让我们一起看看代码，我相信通过重构和优化，它会变得更好。记住，Rome wasn't built in a day，代码质量是通过持续改进来提升的。", nil),
		},
	})
	if err != nil {
		log.Fatalf("format template failed: %v\n", err)
	}
	return messages
}

func generate(ctx context.Context, llm model.ToolCallingChatModel, in []*schema.Message) *schema.Message {
	result, err := llm.Generate(ctx, in)
	if err != nil {
		log.Fatalf("llm generate failed: %v", err)
	}
	return result
}

func stream(ctx context.Context, llm model.ToolCallingChatModel, in []*schema.Message) *schema.StreamReader[*schema.Message] {
	result, err := llm.Stream(ctx, in)
	if err != nil {
		log.Fatalf("llm generate failed: %v", err)
	}
	return result
}

func main() {
	ctx := context.Background()

	// 使用模版创建messages
	log.Printf("===create messages===\n")
	messages := createMessagesFromTemplate()
	log.Printf("messages: %+v\n\n", messages)

	// 创建llm
	log.Printf("===create llm===\n")
	cm := createOpenAIChatModel(ctx)
	// cm := createOllamaChatModel(ctx)
	log.Printf("create llm success\n\n")

    // 普通回复
	log.Printf("===llm generate===\n")
	result := generate(ctx, cm, messages)
	log.Printf("result: %+v\n\n", result)

    // 流式回复
	log.Printf("===llm stream generate===\n")
	streamResult := stream(ctx, cm, messages)
	reportStream(streamResult)
}
```

####    基础Agent例子：TODO Agent
正如前文描述，Agent（智能代理）是一个能够感知环境并采取行动以实现特定目标的系统。在 AI 应用中，Agent 通过结合大语言模型的理解能力和预定义工具的执行能力，可以自主地完成复杂的任务

本例子需要构建一个Chain模式的[todoAgent](https://github.com/cloudwego/eino-examples/tree/main/quickstart/todoagent)，构建步骤如下：

1.  定义（实现）&&初始化 Tool
2.  初始化&&配置`ChatModel`
3.  将 `tools` 绑定到 `ChatModel`
4.  创建 `tools` 节点
5.  构建完整的处理链Chain
6.  编译并运行 Chain

上述步骤构建出来如下的编排流程（Chain）：

![todoagent-1]()

工具定义的实现如下：

```go

// 获取添加 todo 工具
// 使用 utils.NewTool 创建工具
func getAddTodoTool() tool.InvokableTool {
	info := &schema.ToolInfo{
		Name: "add_todo",
		Desc: "Add a todo item",
		ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
			"content": {
				Desc:     "The content of the todo item",
				Type:     schema.String,
				Required: true,
			},
			"started_at": {
				Desc: "The started time of the todo item, in unix timestamp",
				Type: schema.Integer,
			},
			"deadline": {
				Desc: "The deadline of the todo item, in unix timestamp",
				Type: schema.Integer,
			},
		}),
	}

	return utils.NewTool(info, AddTodoFunc)
}

// ListTodoTool
// 获取列出 todo 工具
// 自行实现 InvokableTool 接口
type ListTodoTool struct{}

func (lt *ListTodoTool) Info(_ context.Context) (*schema.ToolInfo, error) {
	return &schema.ToolInfo{
		Name: "list_todo",
		Desc: "List all todo items",
		ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
			"finished": {
				Desc:     "filter todo items if finished",
				Type:     schema.Boolean,
				Required: false,
			},
		}),
	}, nil
}

type TodoUpdateParams struct {
	ID        string  `json:"id" jsonschema_description:"id of the todo"`
	Content   *string `json:"content,omitempty" jsonschema_description:"content of the todo"`
	StartedAt *int64  `json:"started_at,omitempty" jsonschema_description:"start time in unix timestamp"`
	Deadline  *int64  `json:"deadline,omitempty" jsonschema_description:"deadline of the todo in unix timestamp"`
	Done      *bool   `json:"done,omitempty" jsonschema_description:"done status"`
}

type TodoAddParams struct {
	Content  string `json:"content"`
	StartAt  *int64 `json:"started_at,omitempty"` // 开始时间
	Deadline *int64 `json:"deadline,omitempty"`
}

func (lt *ListTodoTool) InvokableRun(_ context.Context, argumentsInJSON string, _ ...tool.Option) (string, error) {
	logs.Infof("invoke tool list_todo: %s", argumentsInJSON)

	// Tool处理代码
	// ...

	return `{"todos": [{"id": "1", "content": "在2024年12月10日之前完成Eino项目演示文稿的准备工作", "started_at": 1717401600, "deadline": 1717488000, "done": false}]}`, nil
}

func AddTodoFunc(_ context.Context, params *TodoAddParams) (string, error) {
	logs.Infof("invoke tool add_todo: %+v", params)

	// Tool处理代码
	// ...

	return `{"msg": "add todo success"}`, nil
}

func UpdateTodoFunc(_ context.Context, params *TodoUpdateParams) (string, error) {
	logs.Infof("invoke tool update_todo: %+v", params)

	// Tool处理代码
	// ...

	return `{"msg": "update todo success"}`, nil
}
```

```go
func main() {
	......
	var handlers []callbacks.Handler
	......
	updateTool, err := utils.InferTool("update_todo", "Update a todo item, eg: content,deadline...", UpdateTodoFunc)
	if err != nil {
		logs.Errorf("InferTool failed, err=%v", err)
		return
	}

	// 创建 DuckDuckGo 工具
	searchTool, err := duckduckgo.NewTextSearchTool(ctx, &duckduckgo.Config{})
	if err != nil {
		logs.Errorf("NewTextSearchTool failed, err=%v", err)
		return
	}

	// 初始化 tools（不同的Tool实现方法）
	todoTools := []tool.BaseTool{
		getAddTodoTool(), // 使用 NewTool 方式
		updateTool,       // 使用 InferTool 方式
		&ListTodoTool{},  // 使用结构体实现方式, 此处未实现底层逻辑
		searchTool,
	}

	// 创建并配置 ChatModel
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		BaseURL:     openAIBaseURL,
		Model:       openAIModelName,
		APIKey:      openAIAPIKey,
		Temperature: gptr.Of(float32(0.7)),
	})
	if err != nil {
		logs.Errorf("NewChatModel failed, err=%v", err)
		return
	}

	// 获取工具信息, 用于绑定到 ChatModel
	toolInfos := make([]*schema.ToolInfo, 0, len(todoTools))
	var info *schema.ToolInfo
	for _, todoTool := range todoTools {
		info, err = todoTool.Info(ctx)
		if err != nil {
			logs.Infof("get ToolInfo failed, err=%v", err)
			return
		}
		toolInfos = append(toolInfos, info)
	}

	// 将 tools 绑定到 ChatModel（LLM）
	err = chatModel.BindTools(toolInfos)
	if err != nil {
		logs.Errorf("BindTools failed, err=%v", err)
		return
	}

	// 创建 tools 节点
	todoToolsNode, err := compose.NewToolNode(ctx, &compose.ToolsNodeConfig{
		Tools: todoTools,
	})
	if err != nil {
		logs.Errorf("NewToolNode failed, err=%v", err)
		return
	}

	// 构建完整的处理链
	chain := compose.NewChain[[]*schema.Message, []*schema.Message]()
	chain.
		AppendChatModel(chatModel, compose.WithNodeName("chat_model")).
		AppendToolsNode(todoToolsNode, compose.WithNodeName("tools"))

	// 编译并运行 chain
	agent, err := chain.Compile(ctx)
	if err != nil {
		logs.Errorf("chain.Compile failed, err=%v", err)
		return
	}

	// 运行示例
	resp, err := agent.Invoke(ctx, []*schema.Message{
		{
			Role:    schema.User,
            //TODO：需要添加system prompt
			Content: "添加一个学习 Eino 的 TODO，最后完整列举下tools的功能",
		},
	})
	if err != nil {
		logs.Errorf("agent.Invoke failed, err=%v", err)
		return
	}

	// 输出结果
	for idx, msg := range resp {
		logs.Infof("\n")
		logs.Infof("message %d: %s: %s", idx, msg.Role, msg.Content)
	}
}
```

测试选择模型`qwen-max`，输出如下：
```
[INFO] 2026-02-04 17:42:56 invoke tool add_todo: &{Content:学习 Eino StartAt:0xc0002ad5c0 Deadline:0xc0002ad5b8}
[INFO] 2026-02-04 17:42:56 

[INFO] 2026-02-04 17:42:56 message 0: tool: {"msg": "add todo success"}
```

上面的输出结果未命中工具`list_todo`，将`Content`调整为：`Content: "添加一个学习 Eino 的 TODO，最后完整列举list下tools的功能"`

```
[INFO] 2026-02-04 17:28:35 invoke tool list_todo: {}
[INFO] 2026-02-04 17:28:35 invoke tool add_todo: &{Content:学习 Eino StartAt:0xc00020b6d0 Deadline:0xc00020b6c8}
[INFO] 2026-02-04 17:28:35 

[INFO] 2026-02-04 17:28:35 message 0: tool: {"msg": "add todo success"}
[INFO] 2026-02-04 17:28:35 

[INFO] 2026-02-04 17:28:35 message 1: tool: {"todos": [{"id": "1", "content": "在2024年12月10日之前完成Eino项目演示文稿的准备工作", "started_at": 1717401600, "deadline": 1717488000, "done": false}]}
```

除了Chain式的Agent，框架还提供了[ReAct Agent](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/react_agent_manual/)、[Multi Agent](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/multi_agent_hosting/)等构建方式


##  0x0 其他智能体学习（TODO）

####    `eino_assistant`
[eino_assistant](https://github.com/cloudwego/eino-examples/tree/main/quickstart/eino_assistant)

构建一个基于从 Redis VectorStore 中召回的 Eino 知识回答用户问题，帮用户执行某些操作的 ReAct Agent，即典型的 RAG ReAct Agent。可根据对话上下文，自动帮用户记录任务、Clone 仓库，打开链接等

![eino_practice_agent_graph.png]()

关联文档

-   [Eino 实践](https://www.cloudwego.io/zh/docs/eino/overview/bytedance_eino_practice/)


####    excel agent
[integration-excel-agent](https://github.com/cloudwego/eino-examples/tree/main/adk/multiagent/integration-excel-agent)：Excel Agent 是一个能够听懂你的话、看懂你的表格、写出并执行代码的智能助手。它把复杂的 Excel 处理工作拆解为清晰的步骤，通过自动规划、工具调用与结果校验，稳定完成各项 Excel 数据处理任务

Excel Agent 是一个看得懂 Excel 的智能助手，它先把问题拆解成步骤，再一步步执行并校验结果。它能理解用户问题与上传的文件内容，提出可行的解决方案，并选择合适的工具（系统命令、生成并运行 Python 代码、网络查询等等）完成任务。Excel Agent 整体是基于 Eino ADK 实现的 Multi-Agent 系统，完整架构如下图所示：

![eino_adk_excel_agent_architecture.png]()

关联文档
-   [用 Eino ADK 构建你的第一个 AI 智能体：从 Excel Agent 实战开始](https://www.cloudwego.io/zh/docs/eino/overview/eino_adk_excel_agent/)


##  0x  Cookbook：学习指引

#### 📦 ADK (Agent Development Kit) & Compose 目录规范文档

Hello World

| 目录名称 | 说明 |
| :--- | :--- |
| `adk/helloworld` | **Hello World Agent**：最简单的 Agent 示例，展示如何创建一个基础的对话 Agent |

---

入门示例 （Intro）

| 目录名称 | 说明 |
| :--- | :--- |
| `adk/intro/chatmodel` | **ChatModel Agent**：展示如何使用 ChatModelAgent 并配合 Interrupt 机制 |
| `adk/intro/custom` | **自定义 Agent**：展示如何实现符合 ADK 定义的自定义 Agent |
| `adk/intro/workflow/loop` | **Loop Agent**：展示如何使用 LoopAgent 实现循环反思模式 |
| `adk/intro/workflow/parallel` | **Parallel Agent**：展示如何使用 ParallelAgent 实现并行执行 |
| `adk/intro/workflow/sequential` | **Sequential Agent**：展示如何使用 SequentialAgent 实现顺序执行 |
| `adk/intro/session` | **Session 管理**：展示如何通过 Session 在多个 Agent 之间传递数据和状态 |
| `adk/intro/transfer` | **Agent 转移**：展示 ChatModelAgent 的 Transfer 能力，实现 Agent 间的任务转移 |
| `adk/intro/agent_with_summarization` | **带摘要的 Agent**：展示如何为 Agent 添加对话摘要功能 |
| `adk/intro/http-sse-service` | **HTTP SSE 服务**：展示如何将 ADK Runner 暴露为支持 Server-Sent Events 的 HTTP 服务 |

---

Human-in-the-Loop （人机协作）

| 目录名称 | 说明 |
| :--- | :--- |
| `adk/human-in-the-loop/1_approval` | **审批模式**：展示敏感操作前的人工审批机制，Agent 执行前需用户确认 |
| `adk/human-in-the-loop/2_review-and-edit` | **审核编辑模式**：展示工具调用参数的人工审核和编辑，支持修改、批准或拒绝 |
| `adk/human-in-the-loop/3_feedback-loop` | **反馈循环模式**：多 Agent 协作，Writer 生成内容，Reviewer 收集人工反馈，支持迭代优化 |
| `adk/human-in-the-loop/4_follow-up` | **追问模式**：智能识别信息缺失，通过多轮追问收集用户需求，完成复杂任务规划 |
| `adk/human-in-the-loop/5_supervisor` | **Supervisor + 审批**：Supervisor 多 Agent 模式结合审批机制，敏感操作需人工确认 |
| `adk/human-in-the-loop/6_plan-execute-replan` | **计划执行重规划 + 审核编辑**：Plan-Execute-Replan 模式结合参数审核编辑，支持预订参数修改 |
| `adk/human-in-the-loop/7_deep-agents` | **Deep Agents + 追问**：Deep Agents 模式结合追问机制，在分析前主动收集用户偏好 |
| `adk/human-in-the-loop/8_supervisor-plan-execute` | **嵌套多 Agent + 审批**：Supervisor 嵌套 Plan-Execute-Replan 子 Agent，支持深层嵌套中断 |

---

Multi-Agent （多 Agent 协作）

| 目录名称 | 说明 |
| :--- | :--- |
| `adk/multiagent/supervisor` | **Supervisor Agent**：基础的 Supervisor 多 Agent 模式，协调多个子 Agent 完成任务 |
| `adk/multiagent/layered-supervisor` | **分层 Supervisor**：多层 Supervisor 嵌套，一个 Supervisor 作为另一个的子 Agent |
| `adk/multiagent/plan-execute-replan` | **Plan-Execute-Replan**：计划-执行-重规划模式，支持动态调整执行计划 |
| `adk/multiagent/integration-project-manager` | **项目管理器**：使用 Supervisor 模式的项目管理示例，包含 Coder、Researcher、Reviewer |
| `adk/multiagent/deep` | **Deep Agents (Excel Agent)**：智能 Excel 助手，分步骤理解和处理 Excel 文件，支持 Python 代码执行 |
| `adk/multiagent/integration-excel-agent` | **Excel Agent (ADK 集成版)**：ADK 集成版 Excel Agent，包含 Planner、Executor、Replanner、Reporter |

---

GraphTool （图工具）

| 目录名称 | 说明 |
| :--- | :--- |
| `adk/common/tool/graphtool` | **GraphTool 包**：将 Graph/Chain/Workflow 封装为 Agent 工具的工具包 |
| `adk/common/tool/graphtool/examples/1_chain_summarize` | **Chain 文档摘要**：使用 compose.Chain 实现文档摘要工具 |
| `adk/common/tool/graphtool/examples/2_graph_research` | **Graph 多源研究**：使用 compose.Graph 实现并行多源搜索和流式输出 |
| `adk/common/tool/graphtool/examples/3_workflow_order` | **Workflow 订单处理**：使用 compose.Workflow 实现订单处理，结合审批机制 |
| `adk/common/tool/graphtool/examples/4_nested_interrupt` | **嵌套中断**：展示外层审批和内层风控的双层中断机制 |

---

#### 🔗 Compose （编排）

Chain （链式编排）

| 目录名称 | 说明 |
| :--- | :--- |
| `compose/chain` | **Chain 基础示例**：展示如何使用 compose.Chain 进行顺序编排，包含 Prompt + ChatModel|

Graph （图编排）

| 目录名称 | 说明 |
| :--- | :--- |
| `compose/graph/simple` | **简单 Graph**：Graph 基础用法示例 |
| `compose/graph/state` | **State Graph**：带状态的 Graph 示例 |
| `compose/graph/tool_call_agent` | **Tool Call Agent**：使用 Graph 构建工具调用 Agent |
| `compose/graph/tool_call_once` | **单次工具调用**：展示单次工具调用的 Graph 实现 |
| `compose/graph/two_model_chat` | **双模型对话**：两个模型相互对话的 Graph 示例 |
| `compose/graph/async_node` | **异步节点**：展示异步 Lambda 节点，包含报告生成和实时转录场景 |
| `compose/graph/react_with_interrupt` | **ReAct + 中断**：票务预订场景，展示 Interrupt 和 Checkpoint 实践 |

Workflow （工作流编排）

| 目录名称 | 说明 |
| :--- | :--- |
| `compose/workflow/1_simple` | **简单 Workflow**：最简单的 Workflow 示例，等价于 Graph |
| `compose/workflow/2_field_mapping` | **字段映射**：展示 Workflow 的字段映射功能 |
| `compose/workflow/3_data_only` | **纯数据流**：仅数据流的 Workflow 示例 |
| `compose/workflow/4_control_only_branch` | **控制流分支**：仅控制流的分支示例 |
| `compose/workflow/5_static_values` | **静态值**：展示如何在 Workflow 中使用静态值 |
| `compose/workflow/6_stream_field_map` | **流式字段映射**：流式场景下的字段映射 |

Batch （批处理）

| 目录名称 | 说明 |
| :--- | :--- |
| `compose/batch` | **BatchNode**：批量处理组件，支持并发控制、中断恢复，适用于文档批量审核等场景 |

---

#### 🌊 Flow （流程模块）

ReAct Agent

| 目录名称 | 说明 |
| :--- | :--- |
| `flow/agent/react` | **ReAct Agent**：ReAct Agent 基础示例，餐厅推荐场景 |
| `flow/agent/react/memory_example` | **短期记忆**：ReAct Agent 的短期记忆实现，支持内存和 Redis 存储 |
| `flow/agent/react/dynamic_option_example` | **动态选项**：运行时动态修改 Model Option，控制思考模式和工具选择 |
| `flow/agent/react/unknown_tool_handler_example` | **未知工具处理**：处理模型幻觉产生的未知工具调用，提高 Agent 鲁棒性 |

Multi-Agent

| 目录名称 | 说明 |
| :--- | :--- |
| `flow/agent/multiagent/host/journal` | **日记助手**：Host Multi-Agent 示例，支持写日记、读日记、根据日记回答问题 |
| `flow/agent/multiagent/plan_execute` | **Plan-Execute**：计划执行模式的 Multi-Agent 示例 |

完整应用示例

| 目录名称 | 说明 |
| :--- | :--- |
| `flow/agent/manus` | **Manus Agent**：基于 Eino 实现的 Manus Agent，参考 OpenManus 项目 |
| `flow/agent/deer-go` | **Deer-Go**：参考 deer-flow 的 Go 语言实现，支持研究团队协作的状态图流转 |

---

#### 🧩 Components （组件）

Model （模型）

| 目录名称 | 说明 |
| :--- | :--- |
| `components/model/abtest` | **A/B 测试路由**：动态路由 ChatModel，支持 A/B 测试和模型切换 |
| `components/model/httptransport` | **HTTP 传输日志**：cURL 风格的 HTTP 请求日志记录，支持流式响应和敏感信息脱敏 |

Retriever （检索器）

| 目录名称 | 说明 |
| :--- | :--- |
| `components/retriever/multiquery` | **多查询检索**：使用 LLM 生成多个查询变体，提高检索召回率 |
| `components/retriever/router` | **路由检索**：根据查询内容动态路由到不同的检索器 |

Tool （工具）

| 目录名称 | 说明 |
| :--- | :--- |
| `components/tool/jsonschema` | **JSON Schema 工具**：展示如何使用 JSON Schema 定义工具参数 |
| `components/tool/mcptool/callresulthandler` | **MCP 工具结果处理**：展示 MCP 工具调用结果的自定义处理 |
| `components/tool/middlewares/errorremover` | **错误移除中间件**：工具调用错误处理中间件，将错误转换为友好提示 |
| `components/tool/middlewares/jsonfix` | **JSON 修复中间件**：修复 LLM 生成的格式错误 JSON 参数 |

Document （文档）

| 目录名称 | 说明 |
| :--- | :--- |
| `components/document/parser/customparser` | **自定义解析器**：展示如何实现自定义文档解析器 |
| `components/document/parser/extparser` | **扩展解析器**：使用扩展解析器处理 HTML 等格式 |
| `components/document/parser/textparser` | **文本解析器**：基础文本文档解析器示例 |

Prompt （提示词）

| 目录名称 | 说明 |
| :--- | :--- |
| `components/prompt/chat_prompt` | **Chat Prompt**：展示如何使用 Chat Prompt 模板 |

Lambda

| 目录名称 | 说明 |
| :--- | :--- |
| `components/lambda` | **Lambda 组件**：Lambda 函数组件的使用示例 |

---

#### 🚀 QuickStart （快速开始）

| 目录名称 | 说明 |
| :--- | :--- |
| `quickstart/chat` | **Chat 快速开始**：最基础的 LLM 对话示例，包含模板、生成、流式输出 |
| `quickstart/eino_assistant` | **Eino 助手**：完整的 RAG 应用示例，包含知识索引、Agent 服务、Web 界面 |
| `quickstart/todoagent` | **Todo Agent**：简单的 Todo 管理 Agent 示例 |

---

#### 🛠️ DevOps （开发运维）

| 目录名称 | 说明 |
| :--- | :--- |
| `devops/debug` | **调试工具**：展示如何使用 Eino 的调试功能，支持 Chain 和 Graph 调试 |
| `devops/visualize` | **可视化工具**：将 Graph/Chain/Workflow 渲染为 Mermaid 图表 |

##  0x0 参考
-   [eino官方文档](https://github.com/cloudwego/eino/blob/main/README.zh_CN.md)
-   [实现一个最简 LLM 应用](https://www.cloudwego.io/zh/docs/eino/quick_start/simple_llm_application/)
-   [Agent-让大模型拥有双手](https://www.cloudwego.io/zh/docs/eino/quick_start/agent_llm_with_tools/)
-   [用 Eino ADK 构建你的第一个 AI 智能体：从 Excel Agent 实战开始](https://segmentfault.com/a/1190000047399985)
-   [从原理到实践：万字长文深入浅出教你优雅开发复杂AI Agent](https://zhuanlan.zhihu.com/p/1919338285160965135)
-   [DeepSeek + Function Call：基于 Eino 的计划--执行多智能体范式实战](https://www.infoq.cn/article/zhgae6llqwo9cjwr9lks)
-   [Eino: Cookbook](https://www.cloudwego.io/zh/docs/eino/eino-cookbook/)
-   [Eino: ReAct Agent 使用手册](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/react_agent_manual/)
-   [Eino Tutorial: Host Multi-Agent（日记助手实现）](https://www.cloudwego.io/zh/docs/eino/core_modules/flow_integration_components/multi_agent_hosting/)
-   [字节跳动大模型应用 Go 开发框架：Eino 实践](https://www.cloudwego.io/zh/docs/eino/overview/bytedance_eino_practice/)
-   [Agent 还是 Graph？AI 应用路线辨析](https://www.cloudwego.io/zh/docs/eino/overview/graph_or_agent/)
-	[DeepResearch之DeerFlow](https://zhuanlan.zhihu.com/p/1905211481391366801)
-	[使用Eino框架实现DeerFlow系统](https://mp.weixin.qq.com/s?__biz=Mzg2MTc0Mjg2Mw==&mid=2247495153&idx=1&sn=e207794d53c6bc8256c5f8784aa13218&scene=21&poc_token=HGcIj2mjT2Tq4suIikAy3dGkk5fZthr4OP4gJjcA)