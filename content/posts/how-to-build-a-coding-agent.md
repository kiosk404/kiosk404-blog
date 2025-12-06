---
title: 搭建编程智能体：实战工作坊
author: kiosk
tags: []
categories: []
date: 2025-12-06 11:19:00
---



自 OpenAI 公司于 2022 年 11 月 30 日发布 ChatGPT 以来，经过 这短短3年的发展之后，大语言模型的概念已逐渐普及，各种基于大语言模型的周边产品，以及集成层出不穷，可以说已经玩出花来了。

在 AI 大模型爆发的时代，智能编程能代理（Coding Agent）正逐渐成为开发者的得力助手。它们能理解需求、编写代码、操作文件甚至调用系统工具，仿佛一位不知疲倦的编程助理。当然要做到 cursor、google antigravity 那样强大的编程LLM还有很长的路要走。



本文打算以入门科普的方式介绍一下 一个 LLM 是如何逐渐 “进化“ 成一个智能体的。

> 代码参见 https://github.com/kiosk404/how-to-build-a-coding-agent

<!--more-->

## 认识 function call -- 发展历史

Function Call（函数调用）作为大语言模型（LLM）从纯文本交互迈向实际任务执行的核心能力，其诞生并非偶然，而是 AI 技术发展、实际应用需求与模型能力边界共同作用的结果。

### 大语言模型的能力瓶颈：从 “说” 到 “做” 的鸿沟

2020-2022 年，以 GPT-3、PaLM、LLaMA 为代表的大语言模型迎来爆发式发展，在文本生成、理解、翻译等领域展现出惊人能力，但很快暴露出核心瓶颈：**模型仅能输出自然语言文本，无法直接与外部系统或工具交互以完成实际任务**。

- **知识时效性限制**：大语言模型的训练数据存在 “截止日期”，无法获取实时信息（如 2023 年后的新闻、实时股票价格、天气数据）。例如，当用户询问 “今天的北京气温是多少” 时，模型只能基于训练数据推测，无法给出准确答案。
- **复杂任务执行短板**：对于需要多步骤操作或工具辅助的任务（如 “计算 1998+2025 的结果”“生成并保存一个 Excel 文件”“调用地图 API 查询路线”），模型仅能描述操作步骤，却无法直接执行。
- **精准性与可靠性不足**：模型在数值计算、逻辑推理等任务中易出现 “幻觉” 或错误，若能调用计算器、代码解释器等工具，可大幅提升结果准确性。

这种 “能说不能做” 的局限，使得大语言模型难以落地到实际生产场景，成为制约其产业化应用的关键障碍。

<br/>

### 技术铺垫：模型能力的成熟与交互范式的探索

Function Call 的诞生也依赖于前期技术的积累与验证：

1. **模型理解与生成能力的提升**：2022 年前后，GPT-3.5、ChatGPT 等模型展现出更强的上下文理解、指令遵循和结构化输出能力（如 JSON 格式生成），使得模型能够精准识别用户需求中的工具调用意图，并按指定格式输出调用参数。例如，模型可根据用户指令 “查询上海到广州的高铁票”，生成包含 “出发地”“目的地”“日期” 等参数的 JSON 结构，为函数调用提供标准化输入。
2. **Prompt 工程的实践突破**：开发者通过 Prompt 设计引导模型输出工具调用指令的尝试，验证了 “自然语言意图→结构化调用指令” 的可行性。例如，在 Prompt 中明确告知模型 “若需要调用工具，请按 {"function":"工具名","parameters":{"参数名": 参数值}} 的格式输出”，模型可稳定遵循该规则。
3. **工具集成的早期探索**：在 Function Call 标准化之前，已有开发者尝试通过 “文本解析 + API 调用” 的方式实现 AI 与工具的联动（如 LangChain 框架的早期工具链集成），但这种方式依赖手动解析模型输出，兼容性和稳定性差，亟待标准化的交互方案。

<br/>

### 产业标准化推动：OpenAI 的率先定义与行业跟进

2023 年 3 月，OpenAI 正式推出 ChatGPT 的 Function Call 功能，首次将函数调用能力标准化为模型的原生接口：

- **定义了标准化交互流程**：模型接收用户输入后，可输出包含 “函数名称”“参数列表” 的结构化响应；开发者通过 API 调用外部函数后，将结果回传给模型，模型再生成自然语言反馈给用户，形成闭环。
- **简化了工具集成的技术门槛**：开发者无需手动解析模型的自然语言输出，只需按 OpenAI 定义的 Schema 注册函数，模型即可自动完成意图匹配和参数生成。

这一举措迅速引发行业跟进，Anthropic、Google、百度等厂商相继为其大模型推出函数调用功能，并逐步形成了相对统一的技术范式。Function Call 由此从技术探索走向产业标准化，成为大语言模型实现 “智能代理” 能力的核心技术模块。

<br/>

## 实践 function call 

### 准备 LLM 环境 Ollama

在开始前，需要安装一下 ollama，这里需要注意的是，Ollama 作为本地大模型运行框架，其模型对 Function Call 的支持并非由 Ollama 本身直接决定，而是取决于模型的训练特性、是否适配 Ollama 的工具调用规范，以及是否加载了对应的 Modelfile 配置。以下是具体的判定方法和关键依据：

<br/>

Ollama 的官方模型仓库（[Ollama Library](https://ollama.com/library)）是最直接的参考来源：

**模型标签与描述**：官方标注 “function calling”,“tools”,“agent” 等标签的模型，明确支持 Function Call。例如：

- Llama 3.1（70B/400B）、Llama 3.2（3B/7B/11B）：官方明确说明支持工具调用（Tool Use）和 Function Call；
- Gemma（Ollama 适配版）：<span style="color:red">不支持 Function Call ，需要注意！！！</span>；
- Qwen 3（7B/14B/72B）：官方描述中包含 “支持函数调用与工具扩展”；
- gpt-oss（Ollama 适配版）：明确支持 Function Call 和结构化输出。

<br/>

![ollama_library](https://img1.kiosk007.top/static/images/blog/20251206115558-ollama_library.png)



### 让大模型读取本地文件

就以 [read/read.go](https://github.com/kiosk404/how-to-build-a-coding-agent/blob/master/read/read.go) 为例。

这里我使用的是 Ollama 本地部署的 qwen3:1.7b 。

首先需要定义一个 function，如下所示。其中有一个 `read_file` 读取本地文件的函数，我们需要对其做一些简单的描述。如 

`Read the contents of a given relative file path. Use this tool when you need to read the contents of a file in the working directory. -- 读取给定相对文件路径的内容。当你需要读取工作目录中的文件内容时，请使用此工具。`

另外包含1个参数，就是文件路径。

```go
type ToolDefinition struct {
	Name        string                     `json:"name"`
	Description string                     `json:"description"`
	InputSchema api.ToolFunctionParameters `json:"input_schema"`
	Function    func(input json.RawMessage) (string, error)
}

var ReadFileDefinition = ToolDefinition{
	Name:        "read_file",
	Description: "Read the contents of a given relative file path. Use this tool when you need to read the contents of a file in the working directory.",
	InputSchema: api.ToolFunctionParameters{
		Type:     "object",
		Required: []string{"path"},
		Properties: map[string]api.ToolProperty{
			"path": {
				Type:        api.PropertyType{"string"},
				Description: "The relative path of a file in the working directory.",
			},
		},
	},
	Function: ReadFile,
}
```



模型调用

```bash
➜  how-to-build-a-coding-agent git:(master) go run read/read.go --model=qwen3:1.7b

Chat with Ollama (type 'exit' to quit)
? You: 请读取 read/demo_read.txt 文件的内容，如果其内容是一个问题的话，告诉我问题答案！

Tool Input: {"path":"read/demo_read.txt"}
ReadFile path: read/demo_read.txt
Successfully read file read/demo_read.txt, content length: 174
Tool Output: I have a mane but I'm not a lion,
I have four legs but I'm not a table,
I can gallop but I'm not running,
People say I'm disagreeable because I always say "neigh."
What am I?
Tool read_file executed successfully
Ollama: 这个谜语的答案是“马”！🐴✨
**解析**：  
1. "Mane" 指马的鬃毛，符合“不是狮子”的描述；  
2. 马有四条腿，符合“不是桌子”的描述；  
3. 马能 gallop（跳跃），但“running”指跑步，符合“不是跑步”的描述；  
4. 马的叫声“neigh”与“disagreeable”（不耐烦）的关联，符合谜语最后的描述。  

因此，答案是：**马**。
? You: 

```

可以看到模型不但可以读取到 本地文件 `demo_read.txt` 还将其内容回答出来了，虽然我个人感觉不太像是答案...



### Function Call 的交互细节

从 wireshark 抓包可以看到，当我问了一个问题时，实际上 调用了2次 Ollama API。

![ollama-function-call](https://img1.kiosk007.top/static/images/blog/20251206124414-ollama-function-call.png)



**第一步**：`Request  POST` 客户端发送请求，这一步可以看到，请求的同时将 Tool 工具也带入到了参数中

```json
{
    "model": "qwen3:1.7b",
    "messages": [
        {
            "role": "user",
            "content": "读取 read/demo_read.txt 下的内容，并告诉我这个问题的答案是什么？"
        }
    ],
    "stream": false,
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "read_file",
                "description": "Read the contents of a given relative file path. Use this tool when you need to read the contents of a file in the working directory.",
                "parameters": {
                    "type": "object",
                    "required": [
                        "path"
                    ],
                    "properties": {
                        "path": {
                            "type": "string",
                            "description": "The relative path of a file in the working directory."
                        }
                    }
                }
            }
        }
    ],
    "options": null
}
```



**第二步**：`Response 200` qwen3 经过 思考，知道要想回答问题的答案需要先调用工具知道文件的内容，所以返回需要调用 tool_calls 调用方法是 read_file ，参数是 read/demo_read.txt

```json
{
    "model": "qwen3:1.7b",
    "created_at": "2025-12-06T09:29:19.042559649Z",
    "message": {
        "role": "assistant",
        "content": "",
        "thinking": "好的，用户让我读取read/demo_read.txt文件的内容，并告诉我问题的答案。首先，我需要确认这个文件是否存在。不过，作为工具，我只能调用read_file函数来读取文件。用户的问题可能涉及到文件内容的解析，但我不知道具体的问题是什么。可能需要先读取文件内容，然后根据内容来回答问题。不过用户可能希望直接得到答案，而不仅仅是文件内容。这时候可能需要进一步的信息，但根据当前工具，只能调用函数读取文件。所以，我应该先调用read_file函数，参数是path为read/demo_read.txt。然后等待返回内容，再根据内容来回答问题。不过用户的问题可能已经包含在文件中，所以需要先获取文件内容再处理。因此，正确的步骤是调用函数读取文件，然后在得到内容后，根据问题的答案进行回答。但这里用户可能希望直接得到答案，所以可能需要先读取文件，再解析问题。不过目前只能调用函数，所以先执行函数调用。\n",
        "tool_calls": [
            {
                "function": {
                    "index": 0,
                    "name": "read_file",
                    "arguments": {
                        "path": "read/demo_read.txt"
                    }
                }
            }
        ]
    },
    "done": true,
    "done_reason": "stop",
    "total_duration": 10480779011,
    "load_duration": 1431494404,
    "prompt_eval_count": 185,
    "prompt_eval_duration": 1663983690,
    "eval_count": 242,
    "eval_duration": 7289456350
}
```



**第三步**：`Request POST` 客户端再次发起请求并将这个文件的内容也携带上了，（Agent 每次向 大模型请求都是携带完整的 历史记录的，这也意味着随着对话轮数越多，交互携带的信息将越来越大）

```json
{
    "model": "qwen3:1.7b",
    "messages": [
        {
            "role": "user",
            "content": "读取 read/demo_read.txt 下的内容，并告诉我这个问题的答案是什么？"
        },
        {
            "role": "assistant",
            "content": "",
            "thinking": "好的，用户让我读取read/demo_read.txt文件的内容，并告诉我问题的答案。首先，我需要确认这个文件是否存在。不过，作为工具，我只能调用read_file函数来读取文件。用户的问题可能涉及到文件内容的解析，但我不知道具体的问题是什么。可能需要先读取文件内容，然后根据内容来回答问题。不过用户可能希望直接得到答案，而不仅仅是文件内容。这时候可能需要进一步的信息，但根据当前工具，只能调用函数读取文件。所以，我应该先调用read_file函数，参数是path为read/demo_read.txt。然后等待返回内容，再根据内容来回答问题。不过用户的问题可能已经包含在文件中，所以需要先获取文件内容再处理。因此，正确的步骤是调用函数读取文件，然后在得到内容后，根据问题的答案进行回答。但这里用户可能希望直接得到答案，所以可能需要先读取文件，再解析问题。不过目前只能调用函数，所以先执行函数调用。\n",
            "tool_calls": [
                {
                    "function": {
                        "index": 0,
                        "name": "read_file",
                        "arguments": {
                            "path": "read/demo_read.txt"
                        }
                    }
                }
            ]
        },
        {
            "role": "tool",
            "content": "I have a mane but I'm not a lion,\nI have four legs but I'm not a table,\nI can gallop but I'm not running,\nPeople say I'm disagreeable because I always say \"neigh.\"\nWhat am I?",
            "tool_name": "read_file"
        }
    ],
    "stream": false,
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "read_file",
                "description": "Read the contents of a given relative file path. Use this tool when you need to read the contents of a file in the working directory.",
                "parameters": {
                    "type": "object",
                    "required": [
                        "path"
                    ],
                    "properties": {
                        "path": {
                            "type": "string",
                            "description": "The relative path of a file in the working directory."
                        }
                    }
                }
            }
        }
    ],
    "options": null
}
```



**第四步**：`Response 200` 大模型再次经过思考，将结果回答输出

```json
{
    "model": "qwen3:1.7b",
    "created_at": "2025-12-06T09:29:35.971407828Z",
    "message": {
        "role": "assistant",
        "content": "The answer to the riddle is **\"cow\"** (牛). \n\nHere's the breakdown:\n1. \"I have a mane but I'm not a lion\" → Cow's鬃毛 (mane)  \n2. \"I have four legs but I'm not a table\" → Cow's legs  \n3. \"I can gallop but I'm not running\" → Cow's gallop  \n4. \"People say I'm disagreeable because I always say 'neigh'\" → Cow's叫声 (neigh)  \n\nThe riddle plays on the literal and metaphorical meanings of these traits.",
        "thinking": "好的，用户让我解决这个谜语。首先，我需要理解谜语的内容。谜语的每一句都在描述一个事物，但最后一句的答案应该是谜底。让我仔细看看每一句：\n\n第一句：“I have a mane but I'm not a lion,” 这里提到“mane”通常是指狮子的鬃毛，但谜底不是狮子，所以可能是指某种动物的鬃毛，或者别的东西。\n\n第二句：“I have four legs but I'm not a table,” 四条腿但不是桌子，可能是指动物，比如狗，但桌子是四条腿的，但谜底不是桌子。\n\n第三句：“I can gallop but I'm not running,” 马是能奔腾的，但谜底不是马。\n\n第四句：“People say I'm disagreeable because I always say 'neigh.'” 这里“neigh”是牛的叫声，所以可能谜底是牛。\n\n不过需要确认每句话的逻辑。第一句可能指牛的鬃毛，第二句指牛的腿，第三句指牛的奔跑，第四句明确是牛的叫声。所以综合起来，谜底应该是牛，因为它的鬃毛、四条腿、能奔腾，而且叫声是neigh。不过需要确认是否符合所有句子的描述。\n"
    },
    "done": true,
    "done_reason": "stop",
    "total_duration": 16927164947,
    "load_duration": 79364527,
    "prompt_eval_count": 487,
    "prompt_eval_duration": 2756058956,
    "eval_count": 404,
    "eval_duration": 13934454335
}
```

<br/>

### Function Call 的关键技术

#### 1. 提示词工程（Prompt Engineering）

这是实现 Function Call 的基础手段，通过在提示词中明确告知模型：

- 可用的函数列表及元数据；

- 生成函数调用指令的格式要求（如 JSON 结构、字段名约束）；

- 何时需要调用函数（如 “当需要查询实时数据时，调用指定函数”）。

  

  例如，典型的 Prompt 模板：

```json
你可以调用以下函数：
{函数列表的文本描述}
如果需要调用函数，请按以下JSON格式输出：
{"name": "函数名", "arguments": {"参数名": "参数值"}}
如果不需要调用函数，请直接回答用户问题。

用户问题：{用户输入}
```

模型通过学习 Prompt 中的规则，生成符合要求的函数调用指令。



#### 2. 意图识别与参数提取 (Intent Recognition & Parameter Extraction)

LLM 通过其训练和**微调**后的能力，分析用户的**自然语言输入**。

- **作用：**
  - 首先，识别用户想要执行的**任务**（**意图**）。
  - 其次，将用户输入中的相关信息映射到所需函数的**参数**中，并以结构化的 JSON 格式输出。



#### 3. **模型微调与结构化输出 (Model Fine-tuning & Structured Output)**

这是 Function Call 的**核心技术**，确保模型能够生成可被机器解析的指令。

- **原理：**
  - 为了让 LLM 稳定且可靠地输出特定格式（如 JSON）的函数调用指令，模型需要经过**特定数据集的微调（Fine-tuning）**。
  - 微调数据集中包含大量的用户查询、相应的可用函数描述，以及模型应该输出的**结构化 JSON 响应**。这教会了模型在特定条件下**放弃自然语言生成**，转而生成代码或指令。
  - 部分先进模型采用 **约束解码(Constrained Decoding)** 技术，确保生成的序列符合预定义的语法结构（例如 JSON Schema），从而提高输出的准确性和可靠性。
- **作用：**
  - 将非结构化的用户意图（自然语言）转化为**结构化、可执行**的函数调用指令（JSON）。
  - 保证输出的格式严谨性，避免因为格式错误导致外部执行器无法解析。

<br/>



## 模型上下文协议 (Model Context Protocol, MCP)

在 MCP 出现之前，AI 应用的开发面临一个巨大的挑战：**信息孤岛和非标准化集成**。尽管 Function Call 已经提供了 LLM 操作外部的能力。但是假设有 $N$ 个 LLM（如 GPT-4、Claude、Gemini）和 $M$ 个外部工具，理论上就需要 $N \times M$ 种不同的集成方案和大量的维护工作。这种**重复、脆弱且难以扩展**的架构极大地限制了 AI 代理技术的普及和落地。

Anthropic 观察到这一痛点，认为要让 AI 代理真正发挥作用，它们必须能**安全、高效、统一**地访问外部信息并执行动作。**2024 年 11 月 25 日**，Anthropic 正式宣布推出并开源 Model Context Protocol (MCP)。将其定位为一个**开放标准（Open Standard）**，邀请整个行业共同参与和维护。

MCP 旨在为大型语言模型（LLM）应用程序提供一个**标准化接口**，使其能够安全、双向地连接和使用**外部数据源、工具和应用程序**。



#### MCP 的定义



<style>
.cards-container {
  display: flex;
  gap: 16px;
  margin: 16px 0;
}
.card {
  flex: 1;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  overflow: hidden;
}
.card-header {
  padding: 10px;
  font-weight: bold;
  color: white;
}
.card-body {
  padding: 12px;
}
.bg-blue { background: #007bff; }
.bg-green { background: #28a745; }
.bg-red { background: #dc3545; }
.bg-yellow { background: #ffc107; }
</style>
<div class="cards-container">
  <div class="card">
    <div class="card-header bg-blue">1. Function Call (函数调用) </div>
    <div class="card-body">
      <p>是一种机制，让 LLM 能够识别用户意图，并生成一个结构化的指令（如 JSON），请求调用一个外部函数或工具。</p>
    </div>
  </div>
  <div class="card">
    <div class="card-header bg-green">2. MCP (Model Context Protocol)</div>
    <div class="card-body">
      <p>是一种标准，定义了如何以统一、安全的方式描述这些外部函数、如何传递调用指令和函数执行结果，从而简化并加速 AI Agent 的开发。</p>
    </div>
  </div>
</div>

Function Call 就像是 LLM 拥有了“**打电话**”的能力，而 MCP 则定义了这通电话的**通信网络、接听标准和通话内容格式**，确保任何 AI 手机（LLM）都能和任何外部服务（工具）顺畅连接。



```go
// registerTools 注册所有工具
func registerTools(server *mcp.Server) {
	// 1. read_file 工具 - 读取文件内容
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "read_file",
			Description: "读取指定文件的内容。支持文本文件，返回文件的完整内容。",
		},
		handleReadFile,
	)

	// 2. list_directory 工具 - 列出目录内容
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "list_directory",
			Description: "列出指定目录下的所有文件和子目录。",
		},
		handleListDirectory,
	)

	// 3. write_file 工具 - 写入文件内容
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "write_file",
			Description: "将指定内容写入到文件中。如果文件不存在，会创建新文件；如果文件已存在，会覆盖原有内容。",
		},
		handleWriteFile,
	)

	// 4. edit_file 工具 - 编辑文件内容
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "edit_file",
			Description: "编辑指定文件的内容。如果文件不存在，会创建新文件；如果文件已存在，会在原有内容基础上进行编辑。",
		},
		handleEditFile,
	)

	// 5. get_file_info 工具 - 获取文件信息
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "get_file_info",
			Description: "获取文件或目录的详细信息，包括大小、修改时间、权限等。",
		},
		handleGetFileInfo,
	)

	// 6. search_files 工具 - 搜索文件
	mcp.AddTool(server,
		&mcp.Tool{
			Name:        "search_files",
			Description: "在指定目录中搜索匹配模式的文件。",
		},
		handleSearchFiles,
	)
}
```



#### 大模型的通信模式 

**模型上下文协议 (MCP)** 定义了两种主要的通信模式，用于 LLM 代理（Agent）与其外部环境（MCP 服务器）进行交互。这两种模式分别是 **Server-Sent Events (SSE)** 和 **Stdio (标准输入/输出)**。

##### 模式一：Server-Sent Events (SSE) - 异步 Web 通信模式

**SSE 模式**主要用于需要**持久连接、实时更新**的场景，通常应用于基于 **HTTP/Web** 的环境。

MCP Agent（客户端）向 MCP Server（服务器）发起一个 HTTP 请求。Server 不会立即关闭连接，而是持续通过这个连接**推送（Push）**数据流给客户端。

<br/>

##### 模式二：Stdio (Standard Input/Output) - 命令行通信模式

**Stdio 模式**是最简单的交互方式，通常用于**命令行工具、本地脚本或需要快速启动/销毁的进程**。

MCP Agent（客户端）启动一个外部程序（MCP Server），并通过程序的**标准输入 (stdin)** 发送请求，然后从程序的**标准输出 (stdout)** 读取响应。