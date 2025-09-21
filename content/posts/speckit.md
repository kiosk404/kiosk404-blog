---
title: 规范驱动的AI开发：SpecKit
author: kiosk
date: 2025-09-19 11:40:25
categories:
  - ai
tags:
  - ai
---

[Spec Kit](https://github.com/github/spec-kit) 是一个开源工具包，帮助你用任意的 AI编码代理（如 qwen coder、 github copilot、 claude code、 gemini cli） 开展规范驱动开发，整个流程分为 <span style="color:red"> 4 </span> 个明确阶段，并在每一个阶段设置校验点，确保“**先对齐意图，再动手实现**”。

<!------>

<br/>

# 什么是规范驱动流程？

作为一个程序员， 最害怕的是听到一些含糊不清的需求，像 “将照片共享添加到我的应用程序” 这样的模糊提示会迫使模型猜测潜在的数千个未声明的需求。人工智能会做出合理的假设，有些假设会是错误的。:sweat_smile:



> 为什么需要 Spec Kit 呢？
>
> 1. 大语言模型擅长 “**补全模式**”，不擅长 “**读心术**” 。 清晰的规范与计划能显著减少歧义。
>
> 2. 将 "意图->技术决策->可验证任务" 分层，跨技术栈同样适用。
>
> 3. 把安全/合规/设计系统等组织约束放进规范与计划中，从一开始就被实现所遵循。
> 4. 需要转向时，只需要更新规范，重生成计划，剩下的交给代理实现。

<br/>

用 Spec Kit 时，**规范（Spec）** 会称为工程过程的中心，他不是静态文档，而是会随着项目演进的 “可执行产物” 。每推进一步，先验证再前进，避免 “看起来对、却跑不起来” 的 vibe-coding 。



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
    <div class="card-header bg-blue">1. Specify (定义)</div>
    <div class="card-body">
      <p>用户描述您想要构建的内容。关注内容和原因，而不是技术栈，需要明确目标阐明 “是什么和为什么”，让代理生成细化规范，作为后续一切工作的单一事实来源</p>
    </div>
  </div>
  <div class="card">
    <div class="card-header bg-green">2. Plan (规划)</div>
    <div class="card-body">
      <p>给出技术栈、架构与约束，代理据此生成实现计划，纳入合规、安全、性能与既有系统集成等要求</p>
    </div>
  </div>
  <div class="card">
    <div class="card-header bg-yellow">3. Tasks (任务)</div>
    <div class="card-body">
      <p>将规范与计划拆解为可独立验证的小任务，例如 “创建带邮箱格式的校验接口”</p>
    </div>
  </div>
</div>
<div class="cards-container">
  <div class="card">
    <div class="card-header bg-red">4. Implement (实现)</div>
    <div class="card-body">
        <p>逐一实现并评<b>审聚焦且可验证</b>的变更，规范告诉要构建什么，计划决定如何构建，任务明确先后顺序。</p>
    </div>
  </div>
</div>    


安装命令行工具 specify 初始化项目结构，然后用 /specify、/plan 、/tasks 三步驱动代理生成规范、计划与任务清单：

```bash
uvx --from git+https://github.com/github/spec-kit.git specify init <PROJECT_NAME>
# 1) /specify 描述愿景与用户体验，让代理生成完整规范
# 2) /plan 传达技术栈与约束，让代理输出实现计划
# 3) /tasks 产出可执行任务列表，供代理逐项实现
```

这套结构化流程可把模糊的诉求，转换为代理能稳定执行的清晰意图。



# ⚡ 快速开始

1. 安装 Specify ， 如果没有安装 uvx 的话，先试用 pipx 安装 uv

```
(optional) pipx install --index-url https://pypi.tuna.tsinghua.edu.cn/simple uv
uvx --from git+https://github.com/github/spec-kit.git specify init demo

```

2. 安装一个 AI 代理，这块很简单，我使用的是 qwen ，安装 qwen-coder 参考: [https://qwenlm.github.io/qwen-code-docs/zh/deployment/](https://qwenlm.github.io/qwen-code-docs/zh/deployment/)



![speckit-1](https://img1.kiosk007.top/static/images/blog/20250919234150-speckit-1.png)



{{< admonition type=tip title="" open=true >}}

我在使用的时候发现即便我的提示词是中文，但是整个过程，AI 代理总是在用英文交互，为了交互方便一些。我最开始先运行了 以下提示词。

```
/constitution Create principles focused on code quality, testing standards, user experience consistency, and performance requirements. at the end, use chinese 
```

![speckit-2](https://img1.kiosk007.top/static/images/blog/20250919235307-speckit-2.png)

{{< /admonition >}}

<br/>

## Step 1：项目引导 (Bootstrap)

进入到项目目录，启用你的 AI 代理（实例使用的是 qwen coder），看到有 /specify 、/plan 、/tasks 等命令即表示环境就绪。

{{< admonition type=example title="重要" open=true >}}

首要任务是用 /specify 写清楚 What 和 Why

{{< /admonition >}}



<style>
pre code {
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>



```bash
/specify 开发 Agentify，这是一个私人 AI Agent 的功能开发平台的前端（仅前端），这个平台是提供的可视化设计与编排工具赋能私人 AI Agent 助手，你可以通过零代码或低代码的方式。可以为单个Agent做在线提示词调试可对话，给 Agent 添加 MCP 或者 Plugin Tool 赋能，或者创建 Workflow ，这块的逻辑可以参考 Coze-Studio （代码是 https://github.com/coze-dev/coze-studio，网站文档介绍是 https://www.coze.cn/open/docs）。
我需要2个类别共5名用户。1 名产品经理 和 4名工程师，请创建先创建2个示例页面，1是工作空间-Develop，该页面需要支持Agent Prompt 编排，2是工作空间-Explore 该页面支持开发 Workflow 或者 Plugin, 本阶段不需要登录，这是一次最初的测试，用于确保基础功能已经就绪。
```

完成后通常会得到一个新分支（如 001-agentify-ai-agent）与 以下目录。

```
➜  my-app git:(001-agentify-ai-agent) ✗ tree -a -I '.git|..'

.
├── .qwen
│   └── commands
│       ├── constitution.toml
│       ├── implement.toml
│       ├── plan.toml
│       ├── specify.toml
│       └── tasks.toml
├── .specify
│   ├── memory
│   │   └── constitution.md
│   ├── scripts
│   │   └── bash
│   │       ├── check-implementation-prerequisites.sh
│   │       ├── check-task-prerequisites.sh
│   │       ├── common.sh
│   │       ├── create-new-feature.sh
│   │       ├── get-feature-paths.sh
│   │       ├── setup-plan.sh
│   │       └── update-agent-context.sh
│   └── templates
│       ├── agent-file-template.md
│       ├── plan-template.md
│       ├── spec-template.md
│       └── tasks-template.md
└── specs
    └── 001-agentify-ai-agent
        └── spec.md

10 directories, 18 files

```



<br/>



## STEP 2: 澄清功能规格 (Clarify)

在同一会话中补充/矫正未覆盖的规则与边界：

```
对于每个示例项目，每个项目应包含 5个到15个不等数量的任务，这些任务随机分布在不同的完成阶段。
确保每个完成阶段至少有一个任务。
```

让代理审查 [Review & Acceptance Checklist]:

{{< admonition type=tip title="" open=true >}}

建议让首次产出的规格作为草案，通过问答澄清需求，不要 “默认确认”。

{{< /admonition >}}



这里打开 `/specs/001-agentify-ai-agent/spec.md` 文件进行修改。

下面的必填项 都必须填写清楚

```
## User Scenarios & Testing *(mandatory)*   用户场景和测试 *(必填)*
## Requirements *(mandatory)*				需求 *(必填)*
## Performance Requirements *(mandatory)*
## Observability Requirements *(mandatory)*
## Testing Requirements *(mandatory)*
```

<br/>

## Step 3: 生成实现计划 (Plan)

此时指定技术栈与详细约束：

```
/plan 该应用程序使用 Vite + React + TypeScript + MonoRepo（Rush）。要尽可能使用 tailwind css\ postcss\ antd.design 等技术 。图像不会上传到任何地方，与后端交互的地方先使用 Mock 的方式进行。
```



完成后将生成实现详细文档，目录结构大致如下：

```
── specs
    └── 001-agentify-ai-agent
        ├── contracts
        │   ├── api.md
        │   ├── README.md
        │   ├── test_agent_get.js
        │   ├── test_agent_put.js
        │   ├── test_plugins_get.js
        │   ├── test_plugins_id_delete.js
        │   ├── test_plugins_id_put.js
        │   ├── test_plugins_post.js
        │   ├── test_workflows_get.js
        │   ├── test_workflows_id_delete.js
        │   ├── test_workflows_id_get.js
        │   ├── test_workflows_id_put.js
        │   └── test_workflows_post.js
        ├── data-model.md
        ├── plan.md
        ├── quickstart.md
        ├── research.md
        └── spec.md

11 directories, 35 files

```

{{< admonition type=example title="研究校验" open=true >}}

核对 research.md 与本地环境版本，对快速演进的框架（如 React/TS） 要求代理补充 "版本与差异点"

{{< /admonition >}}

此外，可使用 AI 代理对快速演进的部分发起针对性的研究，提示词如下：

```
我任务我们需要把问题拆成一系列的步骤，首先，列出实现过程中不确定，或者需要进一步研究才能推进的任务清单。并把这些任务完整的列出来。然后针对每一个任务，各自启动独立的研究子任务，使我们能够并行研究这些具体问题。
```



## STEP 4：计划自校验 (Validate Plan)

让代理审阅计划与细节文件，明确实施步骤与溯源引用：

指令化实现（实例路径）：

```
implement specs/001-agentify-ai-agent/plan.md
```

{{< admonition type=tip title="重要" open=true >}}

实施阶段会执行本地的 CLI，请确保这些 CLI 都已经完成了安装

{{< /admonition >}}