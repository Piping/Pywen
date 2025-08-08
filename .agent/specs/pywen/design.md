# Pywen 架构和设计文档

## 1. 概述

Pywen 是一个基于 Qwen3-Coder 大语言模型的 Python CLI 工具，专为智能软件工程任务设计。它提供了一个对话式界面，能够理解自然语言指令并执行复杂的开发工作流程。

### 1.1 项目背景

Pywen 基于 [Qwen3-Coder](https://github.com/QwenLM/Qwen3-Coder) 大语言模型构建，旨在为开发者提供一个高效且智能的代码助手。项目主要改编自 [Qwen-Code](https://github.com/QwenLM/qwen-code)，并对 Python 开发者和 Qwen3-Coder 模型进行了深度优化。

### 1.2 核心特性

- **Qwen3-Coder-Plus 驱动**：基于阿里云最新的代码专用大模型
- **模块化架构**：构建在模块化架构之上，可扩展且可定制
- **丰富的工具生态系统**：文件编辑、bash 执行、顺序思考等
- **轨迹记录**：详细记录所有 Agent 操作，便于调试和分析
- **智能配置**：首次运行时自动引导配置，支持环境变量
- **会话统计**：实时跟踪 API 调用、工具使用和 token 消耗

## 2. 架构设计

### 2.1 整体架构

Pywen 采用分层架构设计，主要分为以下几个层次：

1. **用户界面层 (UI Layer)**：提供命令行交互界面，处理用户输入和输出显示
2. **智能体层 (Agent Layer)**：核心逻辑层，负责解析用户请求、调用工具和生成响应
3. **工具层 (Tool Layer)**：提供各种工具实现，包括文件操作、网络搜索等
4. **配置管理层 (Configuration Layer)**：处理配置文件和环境变量
5. **核心服务层 (Core Services Layer)**：提供模型交互、工具调度等核心服务

### 2.2 模块结构

```
Pywen/
├── agents/                             # 智能体系统
|   └── qwen/                           # Qwen 智能体实现
|       ├── qwen_agent.py               # Qwen 智能体主类
|       ├── turn.py                     # Qwen 智能体回合管理
|       ├── task_continuation_checker.py    # 任务续写检查
|       └── loop_detection_service.py   # Qwen 智能体循环检测服务
|   base_agent.py                       # 基础智能体类
├── config/                             # 智能体配置管理模块
│   ├── config.py                       # 配置类定义，支持环境变量和配置文件加载
|   └── loader.py                       # 智能体配置加载器
├── core/                               # 智能体核心模块
│   ├── client.py                       # 智能体级LLM 客户端，用于与大模型交互
│   ├── tool_scheduler.py               # 工具调度器
│   ├── tool_executor.py                # 工具执行器
│   ├── tool_registry.py                # 工具注册器
|   └── trajectory_recorder.py          # 轨迹记录器
├── tools/                              # 工具生态系统
│   ├── base.py                         # 工具基类，定义所有工具的抽象接口
│   ├── bash_tool.py                    # Shell 命令执行工具
│   ├── edit_tool.py                    # 文件编辑工具
│   ├── file_tools.py                   # 文件工具（写文件、读文件）
│   ├── glob_tool.py                    # 文件 glob 工具
│   ├── grep_tool.py                    # 文件 grep 工具
│   ├── ls_tool.py                      # 文件 ls 命令工具
│   ├── memory_tool.py                  # 内存工具
│   ├── read_many_files_tool.py         # 批量读取文件工具
│   ├── web_fetch_tool.py               # 网络抓取工具
│   └── web_search_tool.py              # 网络搜索工具（基于 Serper API）
├── docs/                               # 项目文档
│   └── project-structure.md            # 项目结构说明文档
├── trajectories/                       # 执行轨迹记录（自动生成）
├── ui/                                 # CLI 界面
|   ├── commands/                       # 命令模块  
|   |   ├── __init__.py                 # 命令模块入口
|   |   ├── about_command.py            # 关于版本命令
|   |   ├── auth_command.py             # 配置信息命令
|   |   ├── base_command.py             # 命令基类
|   |   ├── clear_command.py            # 清空命令
|   |   ├── help_command.py             # 帮助命令
|   |   ├── memory_command.py           # 记忆命令
|   |   ├── quit_command.py             # 退出命令
|   |   └── ...
|   |── utils/                          # 界面工具模块
|   |   └── keyboard.py                 # 键盘绑定
│   ├── cli_console.py                  # CLI 界面实现
│   ├── command_processor.py            # 命令处理
|   └── config_wizard.py                # 配置向导│   
├── utils                               # 系统级工具模块
|   ├── __init__.py
|   ├── base_content_generator.py       # 抽象类，用于生成内容
|   ├── qwen_content_generator.py       # Qwen 模型内容生成
|   ├── llm_basics.py                   # LLM 基础数据结构
|   ├── llm_client.py                   # LLM 客户端
|   ├── llm_config.py                   # LLM 配置
|   └── token_limits.py                 # Token 限制管理
├── cli.py                              # CLI 入口点，支持 `pywen` 启动
├── pyproject.toml                      # 项目配置文件（依赖管理、构建配置、开发工具配置）
├── README.md                           # 项目说明（英文）
├── README_ch.md                        # 项目说明（中文）
└── pywen_config.json                   # 运行时配置文件（API 密钥、模型设置、用户偏好）
```

## 3. 核心组件设计

### 3.1 智能体系统 (Agent System)

智能体系统是 Pywen 的核心，负责处理用户请求并协调各种工具的执行。

#### 3.1.1 BaseAgent

`BaseAgent` 是所有智能体的基类，提供了通用的功能和接口：

- 工具注册和管理
- 与大语言模型的交互
- 会话历史管理
- 轨迹记录

#### 3.1.2 QwenAgent

`QwenAgent` 是基于 Qwen3-Coder 模型的具体实现，具有以下特性：

- 流式输出处理
- 任务续写检查
- 循环检测
- 多轮对话管理

### 3.2 工具系统 (Tool System)

工具系统为智能体提供各种功能实现，所有工具都继承自 `BaseTool` 基类。

#### 3.2.1 BaseTool

`BaseTool` 定义了工具的基本接口：

- `execute`：执行工具的具体逻辑
- `validate_parameters`：验证参数
- `is_risky`：判断操作是否需要用户确认
- `get_function_declaration`：获取工具声明供 LLM 调用

#### 3.2.2 具体工具实现

Pywen 提供了多种工具实现，包括：

- 文件操作工具：`read_file`, `write_file`, `edit_file`, `ls`, `glob`, `grep`
- 系统工具：`bash`
- 网络工具：`web_fetch`, `web_search`
- 内存工具：`memory`

### 3.3 配置管理系统 (Configuration Management)

配置管理系统负责处理 Pywen 的各种配置，包括模型配置、工具 API 密钥等。

#### 3.3.1 Config

`Config` 类定义了 Pywen 的整体配置，包括：

- 模型配置 (`ModelConfig`)
- 最大迭代次数
- 日志设置
- 轨迹记录设置
- 审批模式

#### 3.3.2 ModelConfig

`ModelConfig` 类定义了模型相关的配置：

- 模型提供商
- 模型名称
- API 密钥
- 基础 URL
- 温度、最大 token 等参数

### 3.4 核心服务 (Core Services)

#### 3.4.1 ToolRegistry

工具注册器负责管理和注册所有可用的工具。

#### 3.4.2 ToolExecutor

工具执行器负责执行工具调用，并处理执行结果。

#### 3.4.3 LLMClient

LLM 客户端负责与大语言模型进行交互，发送请求并接收响应。

## 4. 数据模型

### 4.1 会话历史

Pywen 使用 `LLMMessage` 对象来维护会话历史，每个消息包含角色和内容。

### 4.2 工具调用

工具调用通过 `ToolCall` 对象表示，包含工具名称和参数。

### 4.3 工具结果

工具执行结果通过 `ToolResult` 对象表示，包含执行结果或错误信息。

## 5. 错误处理

Pywen 采用多层次的错误处理机制：

1. **工具级别错误处理**：每个工具负责处理自身的错误，并返回适当的错误信息
2. **智能体级别错误处理**：智能体捕获工具执行错误，并决定如何处理
3. **系统级别错误处理**：处理系统级错误，如配置错误、网络错误等

## 6. 测试策略

### 6.1 单元测试

为每个模块编写单元测试，确保核心功能的正确性。

### 6.2 集成测试

测试模块间的集成，确保各组件协同工作。

### 6.3 端到端测试

模拟用户操作，测试整个系统的功能。

## 7. 安全考虑

### 7.1 工具审批机制

对于高风险操作，Pywen 实现了工具审批机制，需要用户确认后才能执行。

### 7.2 API 密钥管理

API 密钥通过配置文件和环境变量管理，避免硬编码在代码中。

### 7.3 输入验证

对所有用户输入和工具参数进行验证，防止恶意输入。

## 8. CLI 增强功能

### 8.1 命令隐藏机制

为了提升用户体验，Pywen 将实现命令隐藏机制，对于未实现的命令将不会在帮助列表中显示。

#### 8.1.1 设计

- 在命令基类中添加 `is_implemented` 属性，默认为 `True`
- 对于未完全实现的命令，设置该属性为 `False`
- 在帮助命令中过滤掉 `is_implemented=False` 的命令

### 8.2 UV 包管理器集成

Pywen 将使用 `uv` 作为默认的包管理器，以提高包安装速度。

#### 8.2.1 设计

- 在工具开发时引入 `venv` / `uv`
- 修改 bash 工具，检测 pip 命令并自动替换为 uv pip
- 提供配置选项以启用/禁用此功能

### 8.3 CLI 自动补全

为了提高易用性，Pywen 将实现 CLI 自动补全功能。

#### 8.3.1 设计

- 实现文件名自动补全
- 在 shell 模式和自然语言模式下都支持自动补全
- 使用 readline 或类似库实现补全功能

### 8.4 手动停止 Agent 交互

新增 `m` 选项用于手动停止 Agent 交互。

#### 8.4.1 设计

- 在 Agent 交互循环中添加对 `m` 输入的检测
- 当用户输入 `m` 时，立即停止当前模型交互
- 提供相应的用户提示信息

## 9. 显示增强功能

### 9.1 多语言命令说明

Pywen 将支持多语言命令说明。

#### 9.1.1 设计

- 为每个命令提供多语言描述
- 根据系统语言环境或用户配置选择显示语言
- 提供语言切换命令

### 9.2 双列显示 read_files 结果

read_files 结果将支持双列显示，方便查看代码的同时查看 agent 后续输出。

#### 9.2.1 设计

- 修改 read_files 工具的输出格式
- 实现双列显示布局
- 提供配置选项以启用/禁用此功能

### 9.3 模型流式显示

支持模型流式显示，提供更流畅的用户体验。

#### 9.3.1 设计

- 优化 QwenAgent 的流式输出处理
- 在 CLI 界面中实现实时显示
- 处理流式显示中的中断情况

## 10. 多 Agent 支持

### 10.1 预定义 Agent

支持通过 `#agent_name` 使用预定义的 Agent。

#### 10.1.1 设计

- 实现预定义 Agent 配置管理
- 在命令解析中识别 `#agent_name` 语法
- 动态加载和切换 Agent

### 10.2 多 Agent 编排模式

通过 `/multi-agent` 命令提供多种编排模式。

#### 10.2.1 设计

- 实现多种编排模式：leader_workers, solver_verifier, group_chat 等
- 提供编排模式配置和管理
- 在 CLI 中实现 `/multi-agent` 命令