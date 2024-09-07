---
title: LLM大模型 - ai agent应用技术介绍
author: kiosk
date: 2024-08-21 13:52:18
lastmod: 2024-08-29 11:26:27
draft: true
tags:
  - ai
categories:
  - artificial intelligence
---

一个本地大模型，ChatGPT3.5 或 ChatGPT 4  都有这样一些问题。

数据缺少及时性，不能直接帮我们编辑 Word 或者 PDF 文件。大模型目前主要还是基于文本的交互等。

这样的场景非常多，因为大模型的核心能力是 意图理解与文本生成，而在我们实际应用过程中，输入数据和输出数据不仅仅是纯文本等。

针对大型语言模型效果不好的问题，之前人们主要关注大模型再训练、大模型微调、大模型的Prompt增强，但对于专有、快速更新的数据却并没有较好的解决方法，为此检索增强生成（RAG）的出现，弥合了LLM常识和专有数据之间的差距。



<!--more-->

<br/>

# AI Agent

AI Agent是人工智能代理（Artificial Intelligence Agent）的概念，是以大语言模型为核心控制器的一套代理系统。它是一种能够感知环境、进行决策和执行动作的智能实体，通常基于机器学习和人工智能技术，**具备自主性和自适应性，在特定任务或领域中能够自主地进行学习和改进**。一个更完整的Agent，一定是与环境充分交互的，它包括两部分——**一是Agent的部分，二是环境的部分**。

<br/>

一个 **Agent(智能体)** 主要包含以下3个部分：

(1) Perception：感知，主要就是信息的输入，比如文本，语言等信息。

(2) Brain：这个是核心，基于llm，根据输入信息，制定任务计划等。

(3) Action：执行，根据计划执行对应的任务，比如调用第三方api，从工具集(tools)选中合适的tool执行任务。

<br/>



{{< figure src="https://img1.kiosk007.top/static/images/blog/20240829231745-landscape-latest.png" title="RAG  处理流程" >}} 

<br/>

目前比较流行的 Agent 工具，著有有 Auto Gen、LangChain 等，因为 LangChain 即是开源的，又提供一整套围绕大模型的 Agent 工具，可以说使用起来非常方便，而且从设计到构建、部署、运维等多方面都提供支持。

<br/>

# LangChain 介绍

**LangChain** 是一个用于开发由语言模型驱动的应用程序的框架。它使得应用程序能够：

- **具有上下文感知能力**：将语言模型连接到上下文来源（提示指令，少量的示例，需要回应的内容等）
- **具有推理能力**：依赖语言模型进行推理（根据提供的上下文如何回答，采取什么行动等）

起初，LangChain 只是一个技术框架，使用这个框架可以快速开发 AI 应用程序，开发人员不需要储备太多算法层面的知识，只需要知道如何和模型进行交互，也就是熟练掌握模型暴露的 API 接口和参数，就可以利用 LangChain 进行应用开发接口。

LangChain 发展到今天，已经不再是一个纯粹的 AI 应用开发框架，而是成为了一个 AI 应用程序开发平台，它包含 4 大组件。

- [LangChain](https://www.langchain.com/langchain)：大模型应用开发框架。
- [LangSmith](https://www.langchain.com/langsmith)：统一的 DevOps 平台，用于开发、协作、测试、部署和监控大模型应用程序，同时，LangSmith 是一套 Agent DevOps 规范，不仅可以用于 LangChain 应用程序，还可以用在其他框架下的应用程序中。
- [LangGraph](https://www.langchain.com/langgraph)：一个用于使用大模型构建有状态、多参与者应用程序的库，是 2024 年 1 月份推出的。

<br/>

## LangChain 技术架构

LangChain 是一个用于开发由语言模型驱动的应用程序的框架。我们相信，最强大和不同的应用程序不仅将通过 API 调用语言模型，还将：

- 数据感知：将语言模型与其他数据源连接在一起。
- 主动性：允许语言模型与其环境进行交互。

因此，LangChain 框架的设计目标是为了实现这些类型的应用程序。



<img src="https://img1.kiosk007.top/static/images/blog/20240831104443-65cb60ead7ea69f8fa623383_langchain-gif.gif" />

<br/>



## LCEL

LangChain 表达式，前面我们介绍 Chains（链）的时候讲过，LCEL 是用来构建 Chains 的。

以下是一个官方的例子。

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_template("tell me a short joke about {topic}")
model = ChatOpenAI(model="gpt-4")
output_parser = StrOutputParser()

chain = prompt | model | output_parser

chain.invoke({"topic": "ice cream"})
```

这就是 Chain，和Linux里的管道很像，通过特殊字符 `|` 来连接不同的组建，构成复杂链条，以实现特定的功能。

```bash
chain = prompt | model | output_parser
```

每个组建的输出会作为下一个组建的输入，直到最后一个组建执行完毕，当然也可以通过 LCEL 将多个链条关联在一起。

```bash
chain1 = prompt1 | model | StrOutputParser()
chain2 = (
    {"city": chain1, "language": itemgetter("language")}
    | prompt2
    | model
    | StrOutputParser()
)
chain2.invoke({"person": "obama", "language": "spanish"})
```



# RAG 

就像前面讲的，大模型是基于预训练的，一般大模型训练周期 1～3 个月，因为成本过高，所以大模型注定不可能频繁更新知识。正是这个训练周期的问题，导致大模型掌握的知识基本上都是滞后的，GPT-4 的知识更新时间是 2023 年 12 月份，如果我们想要大模型能够理解实时数据，或者把企业内部数据喂给大模型进行推理，我们必须进行检索增强，也就是常说的 **RAG，检索增强生成**。

<br>

一般的过程如下:

1. 通过网络爬虫，爬取大量的信息，这个和搜索引擎爬数据的过程一样，并向量化存储。
2. 用户提问时，先把问题向量化，然后在向量库里检索，将检索到的信息构建成提示，喂给大模型，大模型处理完进行输出。

<br/>

整个过程涉及两个新的概念，一个叫**向量化**，一个叫**向量存储**，你先简单理解下，向量化就是将语言通过数学的方式进行表达，比如男人这个词，通过某种模型向量化后就变成了类似于下面这样的向量数据：

```bash
[0.5,−1.2,0.3,−0.4,0.1,−0.8,1.7,0.6,−1.1,0.9]
```

<br/>



# 用LangChain 实现

因为当前的大模型的训练都有一个问题就是缺少实时性，对最新的事件、新闻无法获取，所以要么



<br/>

# 其他

[langchaingo](https://tmc.github.io/langchaingo/docs/) 是 langchain 的 golang 实现，不过还有很多缺陷。相比官方支持的 Python 版本还有非常大的差距。

以下是我深刻体验到的 langchaingo 的问题

<br/>

- **langchaingo 的 PDF 读取能力很弱**

golang 的 PDF 文字提取技术的三方库用起来比较稀烂，其实 PDF 的文字提取技术本身就是比较复杂的。

langchaingo 的实现在 PDF 文本提取上做的明显性能差很多也是能理解的。因为PDF本质上类似于一张打印了文本的纸。只记载形态而不记载含义。很多的三方库的实现都是基于 OCR 读取的。

langchaingo 用的 PDF 读取库 [github.com/signintech/gopdf](github.com/signintech/gopdf) , 在和 Python 的 PDF 三方库读取能力上

<br/>

详见B站UP主 DingTalk科技 的讲解。[PDF转Word为什么会变成乱码?技术壁垒有多高？](https://www.bilibili.com/video/BV1PCHjeNEjx)

<br/>

****

<br/>



<br/>



参考:

- [LangChain 框架介绍](https://docs.langchain.com.cn/docs/introduction/)

- [LangChain 中文网](https://python.langchain.com.cn/docs/get_started/quickstart)

- [LangChain 中文网 - 二](https://www.langchain.asia/)
