# LLM大模型 - langchain应用技术介绍


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

LangChain 是一个用于开发由语言模型驱动的应用程序的框架。

<br/>

## 架构

<img src="https://img1.kiosk007.top/static/images/blog/20240922105925-llm-langchain.png" />

<br/>

- **Agent**: 负责规划，定义Plan，根据输入返回要执行哪个 Agent Action 或者结束迭代。
- **LLMs**: 大模型，子类需要实现 generate 方法，目前已集成的 LLM 有 80+ 个
- **Tools**: 可以理解为一个 动作 Action，比如创建韵味东，调用API，发邮件等，定义run方法，目前有 80+ 个 Tool 集成，例如 github 、 gitlab 等api工具
- **Memory**: 为大模型提供上下文，定义 load 方法，为 Prompt 添加上下文使得 Agent 带有状态，LangChain Memory 的实现逻辑有8种。
- **Retrieval**：Retrieval 是 DocumentLoader、Transformer、Embedding 等文档提取工具的总称。







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

<br/>

### Runnable 接口

为了实现 LCEL Chain，定义 Runnable ，它有一些同步和异步的接口，并且支持组合为一个新的 Runnable。

```python
class Runnable(Generic[Input, Output], ABC):
    # 同步版本的接口
    def stream(): block # 流式返回 block
    def invoke() # 调用
    def batch(): # 批量调用

    # 异步版本的接口
    def astream()
    def ainvoke()
    def abatch()
    def astream_log()

    # 所有 Runnable 都支持以 | 进行组合
    def __or__():
        return RunnableSequence(first=self, last=coerce_to_runnable(other))
    def __ror__():
        return RunnableSequence(first=coerce_to_runnable(other), last=self)
```

<br/>

Runnable 是一个泛型接口，不同的实现中，InputType 和 OutputType 都是不一样的，下面是一些例子：

> schema: 
>
> https://python.langchain.com/docs/expression_language/interface#input-schema
>
> https://python.langchain.com/docs/expression_language/interface#output-schema

<br/>

### RunnableLambda 函数算子

用于将 python 函数转成 Runnable

```Python
def add_one(x: int) -> int:
    return x + 1

runnable = RunnableLambda(add_one)

runnable.invoke(1) # returns 2
runnable.batch([1, 2, 3]) # returns [2, 3, 4]

# Async is supported by default by delegating to the sync implementation
await runnable.ainvoke(1) # returns 2
await runnable.abatch([1, 2, 3]) # returns [2, 3, 4]
```

<br/>

### Sequence 串行执行 | 上一步的输出作为下一步的输入

`RunnableSequence` 类 LCEL 中最关键的一个类，用于创建一个可运行的对象，每个对象的输出作为下一个对象的输入。几乎用于所有的 chain 中。

可以使用 `|` 运算符来实例化，`|` 运算符的左侧或右侧（或两侧）必须是可运行对象。这个也是 Composition 组合的前提（上面那个架构图中）。

任何 `RunnableSequence` 自动支持同步、异步和批处理操作。

例子：

```Python
def add_one(x: int) -> int:
    return x + 1

def mul_two(x: int) -> int:
    return x * 2

runnable_1 = RunnableLambda(add_one)
runnable_2 = RunnableLambda(mul_two)
sequence = runnable_1 | runnable_2
# 等价于
# sequence = RunnableSequence(first=runnable_1, last=runnable_2)
sequence.invoke(1) # (x+1)*2 = 4
await runnable.ainvoke(1)

sequence.batch([1, 2, 3]) # [4, 6, 8]
await sequence.abatch([1, 2, 3])
```

<br/>

### Parallel 并行执行

RunnableParallel 接受一个 Runnable 的 map，然后并行的执行他们，返回一个 map。

```Python
sequence = RunnableLambda(lambda x: x + 1) | {
    'mul_2': RunnableLambda(lambda x: x * 2),
    'mul_5': RunnableLambda(lambda x: x * 5)
}
sequence.invoke(1) # {'mul_2': 4, 'mul_5': 10}
```

### Fallback 默认逻辑/回退逻辑

有 2 种使用方式：

- 在 chain 上调用 `with_fallbacks` 函数注入一个 Runnable 列表，会返回一个新的 Runnable：`RunnableWithFallbacks`
- 直接使用 RunnableWithFallbacks: `RunnableWithFallbacks(runnable=x, fallbacks=[a, b, c], ...)`

```Python
model = ChatAnthropic().with_fallbacks([ChatOpenAI()])
# 大部分情况下使用 ChatAnthropic, 在 ChatAnthropic 无法工作的情况下使用 ChatOpenAI
model.invoke('hello')

def when_all_is_lost(inputs):
    return ("Looks like our LLM providers are down. "
            "Here's a nice 🦜️ emoji for you instead.")

# 在整个 chain 失败的情况下, 返回固定字符串
chain_with_fallback = (
    PromptTemplate.from_template('Tell me a joke about {topic}')
    | model
    | StrOutputParser()
).with_fallbacks([RunnableLambda(when_all_is_lost)])
```

<br/>

### RemoteRunnable 将 LangServe 当做本地的 LangChain 运行

```Python
# 一个普通的 chain
chain = prompt_template | model | parser

app = FastAPI(
  title="LangChain Server",
  version="1.0",
  description="A simple API server using LangChain's Runnable interfaces",
)

# 这里可以直接将这个 chain 作为 REST api 启动
add_routes(
    app,
    chain,
    path="/chain",
)

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="localhost", port=8000)
from langserve import RemoteRunnable

# 然后在客户端，可以使用 RemoteRunnable 运行上面那个 chain
remote_chain = RemoteRunnable("http://localhost:8000/chain/")
remote_chain.invoke({"language": "italian", "text": "hi"})
```

<br/>

## Model I/O 模型交互

Model I/O 简单理解就是负责和 LM 语言模型交互。

Model I/O 由三部分组成：prompt、LM、parser，通过 LCEL，可以非常简单的将这三部分组成一个 chain：

> 这里注意，PromptTemplate.__or__ 继承了 Runnable.__or__ 的默认实现，返回了 RunnableSequence，所以可以通过 | 组合，并且返回的是一个 chain

```Python
prompt = PromptTemplate.from_template("What is a good name for a company that makes {product}?")

chain = prompt | ChatOpenAI() | StrOutputParser()

chain.invoke({"product": "apple"})
```

<br/>

### Prompt

调用 LLM 前，对 prompt 进行处理，抽象了 PromptTemplate 组件。PromptTemplate 是一个 Runnable，所以继承了 LCEL 语言的特点。

- Message 类型：text 纯文本，和 ChatMessage
- ChatMessage：这里分了角色 role，包含 human，system，ai
  - human 就是用户发的消息
  - system 是人格定义部分
  - ai 是 LLM 返回的

<br/>

### LLM

也就是 GPT3.5，GPT4，Seed 这些语言模型。

LLM 也是 Runnable，拥有 LCEL 语言的特点。

<br/>

### Parser

对 LLM 输出的处理，抽象了 BaseOutputParser 能力

- 比如说格式化字符串、字符串转列表、解析 json、解析 markdown 等等

<br/>

## Agents

以LangChain为例，如果没有代理人，就要把一系列的动作和逻辑“硬编码”到代码之中，也就是LangChain的“链条”之中。这种“硬编码”方法无法充分发挥大语言模型（LLM）的推理能力，或者说智商。

而有了代理人（Agent）之后，大语言模型（LLM）就可以成为一个“推理引擎”，不光是可以跟我们进行“问答”交互，还可以调用一些列的工具，API（程序接口）等，这样就能完成一些更加复杂的任务，并且由大预言模型（LLM）自己去决定调用的顺序，灵活了很多。

为了后续行文方便，这里引入一些常用概念和术语：

- **Tool**：工具

- **Toolkit**：工具包

- **AgentExcutor**：调度器，调度逻辑全靠它

<br/>

### Agent 的工作过程

他的基本行为模式还是由一串提示词（Prompt）和大预言模型交互，然后再去执行任务的。通常Agent可以有一个“角色”，也可以陈述任务的背景，还可以指定执行逻辑的策略。





# 用LangChain 实现

用Python和LangChain来演绎一段Agent的实际任务：就是让Agent自己去网上搜索“北京的面积”和“纽约的面积”，然后计算出来两者的差值。

虽然这个任务非常简单，用提示词+大模型本身就有可能可以得到答案，但是通过这个简单的任务，可以展示Agent的工作流程。



```
import os

from langchain.agents import load_tools
from langchain.agents import initialize_agent
from langchain.agents import AgentType
from langchain.llms import Tongyi

from langchain.tools import DuckDuckGoSearchRun

# duck duck search
search = DuckDuckGoSearchRun()

res = search.run("北京的面积有多大？")
print(res)

res = search.run("纽约的面积有多大？")
print(res)


os.environ["DASHSCOPE_API_KEY"] = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

llm = Tongyi()

tools = load_tools(["ddg-search", "llm-math"], llm=llm)

agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True)

agent.run("北京的面积和纽约的面积差是多少？")
```





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





参考：

- [AI 大语言模型链 LangChain 实战 （三）—— Agent 初探](https://zhuanlan.zhihu.com/p/665223637)

****

<br/>



参考:

- [LangChain 框架介绍](https://docs.langchain.com.cn/docs/introduction/)

- [LangChain 中文网](https://python.langchain.com.cn/docs/get_started/quickstart)

- [LangChain 中文网 - 二](https://www.langchain.asia/)

