---
layout:     post
title:      AI agent开发入门：基于eino开发agent（五）
subtitle:   ADK应用：a excel agent
date:       2026-02-11
author:     pandaychen
catalog:    true
tags:
    - AI
    - eino
---


##  0x00    前言
Eino ADK 参考 Google-ADK 的设计，提供了 Go 语言 的 Agents 开发的灵活组合框架，即 Agent、特别是是Multi-Agent 开发框架，并为多 Agent 交互场景沉淀了通用的上下文传递、事件流分发和转换、任务控制权转让、中断与恢复、通用切面等能力


####    Eino ADK 中的 Agent
Eino ADK 抽象设计：

```go
type Agent interface {
    Name(ctx context.Context) string
    Description(ctx context.Context) string
    Run(ctx context.Context, input *AgentInput) *AsyncIterator[*AgentEvent]
}
```

基于 Agent 抽象，ADK 提供了三大类基础拓展：

-   ChatModel Agent: 应用程序的思考部分，利用 LLM 作为核心，理解自然语言，进行推理、规划、生成响应，并动态决定如何执行或使用哪些工具
-   Workflow Agents：应用程序的协调管理部分，基于预定义的逻辑，按照自身类型（顺序 / 并发 / 循环）控制子 Agent 执行流程。Workflow Agents 产生确定性的，可预测的执行模式，不同于 ChatModel Agent 生成的动态随机的决策
    -   顺序（Sequential Agent）：按顺序依次执行子 Agents
    -   循环（Loop Agent）：重复执行子 Agents，直至满足特定的终止条件
    -   并行（Parallel Agent）：并行执行多个子 Agents
-   Custom Agent：通过接口实现自己的 Agent，允许定义高度定制的复杂 Agent

Eino 内置了几种开箱即用的 Multi-Agent 最佳范式：

-   Supervisor: 监督者模式，监督者 Agent 控制所有通信流程和任务委托，并根据当前上下文和任务需求决定调用哪个 Agent
-   Plan-Execute：计划-执行模式，Plan Agent 生成含多个步骤的计划，Execute Agent 根据用户 query 和计划来完成任务。Execute 后会再次调用 Plan，决定完成任务 / 重新进行规划（前文已介绍）

##  0x0 eino ADK 核心知识点梳理


##  0x0 excel agent实现解读

```mermaid
graph TB
    subgraph TOP["SequentialAgent (顶层入口)"]
        direction TB
        START_TOP((START))

        subgraph PER["① plan_execute_replan (SequentialAgent)"]
            direction TB

            subgraph PLANNER["Planner (Chain)"]
                P_IN["Input"] --> P_L1["Lambda: GenInputFn"]
                P_L1 --> P_CM["ChatModel<br/>JSON Schema 输出"]
                P_CM --> P_L2["Lambda: CollectStream"]
                P_L2 --> P_L3["Lambda: Parse → Plan<br/>→ Session"]
            end

            subgraph LOOP["execute_replan (LoopAgent, ≤20轮)"]
                direction TB

                subgraph EXECUTOR["Executor (ChatModelAgent)"]
                    E_GEN["Lambda: GenModelInput<br/>plan + step + executed_steps"]
                    subgraph E_REACT["ReAct Graph"]
                        E_CM["ChatModel"] --> E_BR{"hasToolCalls?"}
                        E_BR -->|否| E_END["→ Output"]
                        E_BR -->|是| E_TN["ToolNode"]
                        E_TN -->|继续| E_CM

                        subgraph E_TOOLS["Tools"]
                            AT_CA["AgentTool:<br/>CodeAgent"]
                            AT_WS["AgentTool:<br/>WebSearchAgent"]
                        end
                        E_TN --- E_TOOLS
                    end
                    E_GEN --> E_REACT
                end

                subgraph REPLANNER["Replanner (Chain)"]
                    R_L1["Lambda: GenInput<br/>executed_steps +<br/>remaining_steps"]
                    R_CM["ChatModel<br/>ToolChoiceForced"]
                    R_L2["Lambda: CollectStream"]
                    R_DECIDE{"Lambda: 判断 ToolCall"}
                    R_L1 --> R_CM --> R_L2 --> R_DECIDE
                    R_DECIDE -->|create_plan| R_PLAN["更新 Plan → Session"]
                    R_DECIDE -->|submit_result| R_BREAK["BreakLoopAction"]
                end

                EXECUTOR --> REPLANNER
                R_PLAN -->|"继续循环"| EXECUTOR
            end

            PLANNER --> LOOP
        end

        subgraph REPORT["② ReportAgent (ChatModelAgent)"]
            direction TB
            RP_GEN["Lambda: GenModelInput<br/>file_path + work_dir<br/>+ plan + files"]
            subgraph RP_REACT["ReAct Graph"]
                RP_CM["ChatModel"] --> RP_BR{"hasToolCalls?"}
                RP_BR -->|否| RP_END["→ Output"]
                RP_BR -->|是| RP_TN["ToolNode"]
                RP_TN -->|继续| RP_CM
                RP_TN -->|submit_result<br/>ReturnDirectly| RP_CVT["→ 直接返回"]
                RP_CVT --> RP_END

                subgraph RP_TOOLS["Tools"]
                    RPT1["bash"]
                    RPT2["tree"]
                    RPT3["read_file"]
                    RPT4["edit_file"]
                    RPT5["submit_result"]
                    RPT6["image_reader?"]
                end
                RP_TN --- RP_TOOLS
            end
            RP_GEN --> RP_REACT
        end

        START_TOP --> PER --> REPORT --> END_TOP((END))
    end

    subgraph CA_DETAIL["CodeAgent内部(ChatModelAgent)"]
        CA_GEN["Lambda: GenModelInput<br/>working_dir + user_query"]
        subgraph CA_REACT["ReAct Graph"]
            CA_CM["ChatModel"] --> CA_BR{"hasToolCalls?"}
            CA_BR -->|否| CA_END["→ Output"]
            CA_BR -->|是| CA_TN["ToolNode"]
            CA_TN -->|继续| CA_CM
            CA_TN --- CA_T1["python_runner"]
            CA_TN --- CA_T2["bash"]
            CA_TN --- CA_T3["read_file"]
            CA_TN --- CA_T4["edit_file"]
            CA_TN --- CA_T5["tree"]
        end
        CA_GEN --> CA_REACT
    end

    AT_CA -.->|"展开"| CA_DETAIL

    style TOP fill:#fafafa,stroke:#424242
    style PER fill:#e3f2fd,stroke:#1565c0
    style PLANNER fill:#e8f5e9,stroke:#2e7d32
    style LOOP fill:#fff3e0,stroke:#e65100
    style EXECUTOR fill:#fce4ec,stroke:#c62828
    style E_REACT fill:#ffebee,stroke:#d32f2f
    style REPLANNER fill:#fff9c4,stroke:#f9a825
    style REPORT fill:#e0f7fa,stroke:#00838f
    style RP_REACT fill:#e0f2f1,stroke:#00695c
    style CA_DETAIL fill:#f3e5f5,stroke:#6a1b9a
    style CA_REACT fill:#ede7f6,stroke:#4527a0
```

##  0x0 参考
-	[Eino ADK: Quickstart](https://www.cloudwego.io/zh/docs/eino/core_modules/eino_adk/agent_quickstart/)
-	[Eino: ADK - Agent Development Kit](https://www.cloudwego.io/zh/docs/eino/core_modules/eino_adk/)
-   [用 Eino ADK 构建你的第一个 AI 智能体：从 Excel Agent 实战开始](https://segmentfault.com/a/1190000047399985)
-   [Eino ADK：一文搞定 AI Agent 核心设计模式，从0到1搭建智能体系统](https://www.51cto.com/article/829335.html)