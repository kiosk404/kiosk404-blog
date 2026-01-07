---
title: 停止造Agent, 开始造Skills
author: kiosk
date: 2026-01-01 11:40:25
categories:
  - ai
tags:
  - ai
---

10月份，Claude 上线了 [Skills 功能](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)， 这篇来自《[Lenny's Newsletter](https://www.lennysnewsletter.com/p/claude-skills-explained)》的文章（发布于 2025 年 10 月 22 日）核心便围绕 Claude Skills（Claude 技能） 展开，核心是解释如何创建可复用的 AI 工作流。

![skills-1](https://img1.kiosk007.top/static/images/blog/20260108010913-skills-1.png)

<!--more-->

Anthropic 工程师、Skills功能的创造者Barry Zhang 和 Mahesh Murag做客《AI Engineer》节目，分享了他们最新观点：**停止构建 Agent ，开始构建 Skills 吧！**

**他们呼吁开发者停止重复构建独立的Agent，而应该开始为通用的Agent构建和分享技能。**

# Skills 是什么？

Skills是什么？官方定义：Skill = 一个带有指令、脚本与资源的文件夹，Claude 在需要时自动加载，让 Claude 能调用特定“技能”完成专业任务。

有些人会认为，AI 圈又开始造科技名词了，这不就是 Agent + MCP 吗？又是 Function Call 能力的二次封装。

要理解 Claude 的 **Agent Skills** 与 **Agent**、**MCP** 的关系，需先明确三者的核心定位，再通过 “功能分工” 和 “协同逻辑” 拆解 —— 本质上，三者是为解决 AI 从 “通用智能” 到 “专业落地” 的不同痛点而设计的互补体系，而非替代关系。

## **Agent Skills 与 Agent 的关系：从 “通才” 到 “专家” 的升级包**

如果把 **Agent** 比作一个 “聪明但无经验的新员工”，那么 **Agent Skills** 就是给这个员工的 “岗位说明书 + 培训手册 + 工具包”—— 核心是让 Agent 从 “什么都懂一点但不精”，变成 “在特定领域能稳定输出的专家”。

Barry打了一个有趣的比方：



![skills-2](https://img1.kiosk007.top/static/images/blog/20260108010942-skills-2.png)

> “你希望谁来帮你处理税务？是拥有300智商的数学天才Mahesh，还是经验丰富的税务专家Barry？”

现有的 Agent 就像是智商 300 的Mahesh，它们很聪明，但在缺乏重要上下文时，无法很好地吸收专业知识，也无法随时间学习，难以持续、一致地执行特定领域的任务。我们需要的是Barry那样的领域专家，能够基于现有知识提供**一致性执行**。

但不幸的是，现在的Agent 很像是 Mahesh，为了解决这个问题，Claude 团队推出了 **Agent Skills**。

当前的通用Agent 存在两大核心缺陷：

- **缺乏领域专业知识**：Agent 能调用工具（如 Python、API），但不懂 “专业套路”。比如它能处理数据，却不知道 “财务报表需按 GAAP 准则格式输出”“品牌文案需规避某类敏感词”—— 这些需要行业经验的规则，无法通过 Agent 自身的通用智能推导。
- **上下文窗口有限且效率低**：若把所有专业规则都写在 Prompt 里，会占用大量上下文空间（导致成本上升），且每次任务都要重复输入，无法复用。

**Skills 如何赋能 Agent？**

Skills 通过 “模块化封装专业知识”，让 Agent 实现 “按需加载、重复复用”，具体表现为：

- **形态上**：Skills 是一个文件夹，内含 `skill.md`（核心指令，如 “财务分析需计算毛利率、净利率并标注同比变化”）、脚本（如自动生成报表的 Python 代码）、资源（如品牌色卡、报表模板）。

- **使用逻辑上**：Agent 接到任务后，先扫描所有 Skills 的 “元数据”（如 “财务报表生成技能”“品牌文案撰写技能”），判断需要哪些技能后，再加载该技能的完整内容到上下文，按规则执行任务。

- **最终效果**：Agent 从 “靠自己猜怎么做事”，变成 “按专业手册精准做事”。例如：

  - 无 Skills 时：Agent 生成的财务报表格式混乱，需人工调整；
  - 有 Skills 时：Agent 调用 “财务报表技能”，自动按公司模板生成带公式的 Excel，无需修改。

  <br/>

## Skills：为 Agent 提供可组合的专业能力

Skills 本质上是**有组织的文件夹**，用来打包程序化知识。

- 内容可以是脚本、代码、文件，也可以是工具或资源。
- 任何人或 Agent 都可以创建和使用。
- 支持版本管理，方便团队共享和迭代。

![skills-3](https://img1.kiosk007.top/static/images/blog/20260108011044-skills-3.png)

Claude 的 **Agent Skills** 并非随意的文件集合，而是遵循 Anthropic 定义的标准化规范设计，涵盖 **文件结构、内容格式、元数据规则、加载逻辑** 四大核心维度，确保技能可复用、可跨平台运行、可被 Claude 自动识别。结合官方文档与开发指南，其具体规范可拆解为以下 6 个关键模块：

**Skills 解决上下文爆炸的核心机制：从 “全量加载” 到 “按需调用”**

传统 Agent 扩展能力时，常将所有工具指令、领域知识、历史规则一次性塞进上下文窗口（例如把 5000 Token 的 “财务报表规范” 直接写入 Prompt），导致对话轮次增加后迅速触发 Token 上限。而 Skills 的设计逻辑完全相反：**通过 “元数据轻量加载 + 核心指令按需披露”，将上下文占用从 “静态大体积” 转为 “动态小颗粒”**，具体通过三个层级实现：

**第一层：元数据预加载（仅占 50-100 Token）**

Skills 的本质是 “带说明书的能力模块”，每个 Skill 文件夹的根目录都包含 `SKILL.md`，其中用 YAML 前置元数据定义核心信息（如技能名称、适用场景、触发关键词），例如：

```c
# SKILL.md 元数据示例（仅占 ~80 Token）
name: 财务报表自动生成
description: 按 GAAP 准则处理 Excel 数据，生成含毛利率/净利率的标准化报表
triggers: ["财务报表", "利润分析", "GAAP 格式"]
version: 1.0
```

Claude 启动时，仅加载所有已安装 Skill 的元数据（而非完整指令），相当于给 Agent 一个 “技能目录”—— 例如同时安装 20 个 Skill，总元数据占用也仅 20×80=1600 Token，远低于传统方案单技能的 Token 消耗。

**第二层：核心指令动态披露（仅在触发时加载）**

只有当用户需求匹配某个 Skill 的触发关键词（如用户说 “生成 10 月财务报表”），Claude 才会加载该 Skill 的完整指令（如报表字段映射规则、公式计算逻辑），且仅加载当前任务必需的部分。例如：

- 若用户只需 “生成基础报表”，则仅加载 `SKILL.md` 中 “基础字段配置” 章节（~300 Token）；
- 若用户需要 “含异常数据标注”，则额外加载 “异常值检测规则” 章节（~200 Token）；
- 无需加载与当前任务无关的内容（如 “报表导出 PDF 步骤”）。

这种 “用多少加载多少” 的机制，避免了 “无关信息占用上下文” 的问题。例如处理财务报表任务时，传统方案需加载 5000 Token 全量规范，而 Skills 仅加载 500-800 Token 核心指令，节省 80% 以上的 Token 占用。

**第三层：脚本 / 模板外部存储（不占 Prompt Token）**

Skills 支持将重量级资源（如 Python 处理脚本、Excel 模板、品牌色卡）以文件形式存储在外部容器（如 Claude 代码执行环境、企业私有存储），仅在执行时通过 “文件 ID 引用” 调用，而非将脚本代码或模板内容写入上下文。例如：

- 生成财务报表时，Skills 仅向 Claude 传递 “调用 `generate_report.py` 脚本，传入数据文件 ID: abc123” 的指令（~50 Token）；
- 脚本的完整代码（~2000 Token）和报表模板（~1500 Token）均存储在外部，不占用 Prompt 上下文。

这种 “指令与资源分离” 的设计，彻底解决了 “工具脚本挤占上下文” 的行业痛点 —— 传统 Agent 处理「自定义 / 轻量化工具执行」时，因无外部资源存储能力，只能将脚本 / 模板塞进上下文，导致 Token 浪费。而 Skills 仅需传递调用指令，Token 消耗降低 95% 以上。

在 [anthropics 的 skills 官方库](https://github.com/anthropics/skills) 里，可以看到很多 skills 的案例集合。

## Skills 实战

Skills开放标准：https://agentskills.io/specification

- 元数据（MetaData） 必定加载
- 指令（Instruction） 按需加载
- 资源（Resource） 按需加载

AI 在使用 Skills 时只将元数据加载进 上下文提示词，如果有需要后续才会紧接着加载 指令和资源。

### 安装 Claude code

[Claude Code - AI coding agent for terminal & IDE | Claude](https://claude.com/product/claude-code)

执行命令安装

```bash
curl -fsSL <https://claude.ai/install.sh> | bash
```

这里我使用 deepseek 模型，来代替 claude opus 模型

```bash
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export ANTHROPIC_AUTH_TOKEN=${DEEPSEEK_API_KEY}
export API_TIMEOUT_MS=600000
export ANTHROPIC_MODEL=deepseek-chat
export ANTHROPIC_SMALL_FAST_MODEL=deepseek-chat
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

<br/>

### 使用 skills



在 claude code 中，skills 是以文件夹的形式存放到项目目录 `./claude` 中的

```bash
项目目录/.claude/skills/skill-name1/
├── SKILL.md          # 元数据 + 指令
├── scripts/          # 资源层 （可选） 
│   └── main.py
├── references/       # 资源层 （可选）
│   └── doc.md
└── assets/           # 资源层 （可选）
    └── pic.png
```

下面是一个最简单的 skills 元数据, 和 hugo markdown 的 page 是一个语法。

```bash
---
name: skill-name
description: A description of what this skill does and when to use it.
---
```

这里我使用 https://github.com/ComposioHQ/awesome-claude-skills/blob/master/domain-name-brainstormer/SKILL.md

这是一个帮人起域名的头脑风暴 skills

输入 `/skills` 可以查看有哪些 skills 技能

我问

```bash
> 我想搞一个AI网站，一个家庭私人管家的相关功能的，用什么域名好？ 
```

![skills-4](https://img1.kiosk007.top/static/images/blog/20260108011248-skills-4.png)



这里会自动加载执行 skills ，AI 机器人执行到了 skills 并让我确认。

以下是输出：

```bash
● /domain-name-brainstormer

● Web Search("AI home personal assistant domain name ideas 2026")
  ⎿  Did 0 searches in 256ms                                                                                                                                                             

● 基于你的AI家庭私人管家网站需求，我为你生成了一些创意域名建议：

  核心概念分析

  - AI/智能: 体现人工智能技术
  - 家庭/家: 定位家庭使用场景
  - 管家/助手: 突出服务性质
  - 私人/专属: 强调个性化服务

  创意域名建议

  中文概念域名 (.com)

  1. homeai.com - 直接结合"家庭"和"AI"，国际通用
  2. familyai.com - 强调家庭场景的AI服务
  3. housebutler.com - "房屋管家"，形象生动
  4. homeassistant.com - 家庭助理，功能明确
  5. smartfamily.com - 智能家庭，概念清晰

  短域名/品牌域名 (.com/.ai)

  6. hearth.ai - "hearth"指家庭/壁炉，温暖感 + .ai扩展名
  7. butler.ai - 直接"管家" + .ai，专业AI感
  8. aide.ai - "aide"是助手，简短易记
  9. nurture.ai - "培育/滋养"，体现家庭关怀
  10. haven.ai - "避风港"，温馨家庭概念
  
  创意组合域名

  11. homely.ai - "homely"有家庭般的舒适感
  12. kinetic.home - "kinetic"动力 + .home扩展名
  13. caretaker.app - "看管人" + .app适合应用
  14. nestmind.com - "巢穴思维"，智能家庭大脑
  15. hearthub.com - "家庭中心枢纽"

  国际范域名 (.io/.co)

  16. homeio.io - 家庭物联网概念
  17. fam.co - 简短，"fam"是family的缩写
  18. assist.io - 助手服务 + 科技感.io
  19. butlr.co - butler的变体，时尚简短
  20. aidev.co - AI + dev(开发)，技术导向
```







参考：

[mp.weixin.qq.com](https://mp.weixin.qq.com/s/tga9kibRyKhnx7ymJVJoiA)