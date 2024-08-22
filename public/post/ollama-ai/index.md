# ollama 本地AI大模型




自 OpenAI 公司于 2022 年 11 月 30 日发布 ChatGPT 以来，经过 23 年一整年的发展之后，大语言模型的概念已逐渐普及，各种基于大语言模型的周边产品，以及集成层出不穷，可以说已经玩出花来了。

在这个过程中，也有不少本地化的模型应用方案冒了出来，针对一些企业知识库问答的场景中，模型本地化是第一优先考虑的问题，因此如何在本地把模型调教的更加智能，就是一个非常重要的技能了。

<!--more-->



# 什么是 ollama?

Ollama 是一个用于构建大型语言模型应用的工具，它提供了一个简洁易用的命令行界面和服务器，让你能够轻松下载、运行和管理各种开源 LLM。



- 项目地址：[ollama(opens new window)](https://github.com/ollama/ollama)
- 官网地址： [https://ollama.com/(opens new window)](https://ollama.com/)
- 模型仓库： [https://ollama.com/library(opens new window)](https://ollama.com/library)
- 此文撰写时项目最新版本：[v0.3.6(opens new window)](https://github.com/ollama/ollama/releases/tag/v0.3.6)
- 官方 logo 是一只可爱的羊驼

<img src="https://img1.kiosk007.top/static/images/blog//20240821003113-ollama.png" alt="Ollama" style="display: block; margin: 0 auto; width: 10%; max-width: 10%; height: auto;">

<br/>



 `Ollama` 是一个基于 Go 语言开发的简单易用的本地大语言模型运行框架。可以将其类比为 docker（同基于 [cobra (opens new window)](https://github.com/spf13/cobra)包实现命令行交互中的 list,pull,push,run 等命令），事实上它也的确制定了类 docker 的一种模型应用标准。如 `ollama pull xxx`、`ollama run xxx ` 等等。

<br/>



在管理模型的同时，它还基于 Go 语言中的 Web 框架 [gin (opens new window)](https://github.com/gin-gonic/gin)提供了一些 Api 接口，让你能够像跟 OpenAI 提供的接口那样进行交互。

官方还为此专门发布了一篇官方博客： https://ollama.com/blog/openai-compatibility (opens new window)，并配了如下一个可爱的图。

<img src="https://img1.kiosk007.top/static/images/blog//20240821004719-1709995014383.webp" alt="Ollama" style="display: block; margin: 0 auto; width: 100%; max-width: 100%; height: auto;">









同时：官方还提供了类似 GitHub，DockerHub 一般的，可类比理解为 ModelHub，用于存放大语言模型的仓库(有 llama 2，mistral，qwen 等模型，同时你也可以自定义模型上传到仓库里来给别人使用)。



<br/>





```
curl -fsSL https://ollama.com/install.sh | sh

>>> Installing ollama to /usr/local
>>> Downloading Linux amd64 CLI
######################################################################## 100.0%#=#=-#  #                                                                  ######################################################################## 100.0%
>>> Making ollama accessible in the PATH in /usr/local/bin
>>> Creating ollama user...
>>> Adding ollama user to render group...
>>> Adding ollama user to video group...
>>> Adding current user to ollama group...
>>> Creating ollama systemd service...
>>> Enabling and starting ollama service...
Created symlink /etc/systemd/system/default.target.wants/ollama.service → /etc/systemd/system/ollama.service.
>>> The Ollama API is now available at 127.0.0.1:11434.
>>> Install complete. Run "ollama" from the command line.
WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode.
```



> ## AMD Radeon GPU 支持[¶](https://ollama.fan/getting-started/linux/#amd-radeon-gpu-support)
>
> 虽然 AMD 已将 `amdgpu` 驱动程序上游贡献给官方 Linux 内核源代码，但该版本较旧，可能不支持所有 ROCm 功能。我们建议您从 [AMD 官网](https://www.amd.com/en/support/linux-drivers) 安装最新驱动程序，以获得对您 Radeon GPU 的最佳支持。
>
> 不幸的是，我的GPU是 intel 的，所以只能是 CPU-only mode，所以有以下报错:
>
> WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode.





<br/>

<br/>



参考文章：

- [二丫讲梵](https://wiki.eryajf.net/pages/97047e/#%E5%89%8D%E8%A8%80)

- [ollama 中文网](https://ollama.fan/getting-started/)
