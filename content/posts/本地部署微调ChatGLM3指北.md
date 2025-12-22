---
title: 本地部署微调 ChatGLM3-6b 指北
author: kiosk
tags: 
  - devops
categories:
  - Artificial Intelligence
date : 2025-11-05 08:50:00
---

<!--more-->

# **部署环境：**

Ubuntu 24.04

Mem: 32G

GPU: 5080 16G

磁盘：建议大于100G

最低要求参考：

https://zhipu-ai.feishu.cn/wiki/OwCDwJkKbidEL8kpfhPcTGHcnVc

**ChatGLM3** 是智谱AI和清华大学 KEG 实验室联合发布的对话预训练模型 原生支持[工具调用](https://github.com/zai-org/ChatGLM3/blob/main/tools_using_demo/README.md)（Function Call）、代码执行（Code Interpreter）和 Agent 任务等复杂场景。



![chat-glm3](https://img1.kiosk007.top/static/images/blog/20251223001842-chat-glm3.png)

## 代码克隆

```bash
git clone <https://github.com/THUDM/ChatGLM3>
```

这项目里面全是 `.py` 后缀的 Python 脚本、Demo 示例、安装环境的说明文档。告诉计算机**如何启动**模型、如何处理你的提问、如何把模型的回答显示在屏幕上。

## 配置运行环境

### **Anaconda 安装**

**Anaconda 是一个面向数据科学、机器学习的 Python/R 编程语言的开源发行版**，你可以把它理解成：

- 一个 “打包好的 Python 全家桶”：内置了 Python 解释器 + 几百个数据科学常用的库（比如 NumPy、Pandas、Matplotlib、Scikit-learn 等）；
- 一个强大的环境 / 包管理工具：核心是 `conda`（比 pip 功能更全的包管理器），能轻松解决不同项目的 Python 版本、库版本冲突问题。

https://repo.anaconda.com/archive/

或使用清华大学的加速镜像地址：

https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/

首先在终端中下载最新的 Anaconda 安装脚本

```bash
wget https://repo.anaconda.com/archive/Anaconda3-2025.12-1-Linux-x86_64.sh

## 或者使用下面的清华的加速地址 (清华的链接不带UA会403...挺无语的)
curl 'https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2025.12-1-Linux-x86_64.sh' \
  -H 'referer: https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/' \
  -H 'user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36' \
  -o Anaconda3-2025.12-1-Linux-x86_64.sh \
  --progress-bar
  
  
## 下载完成后，安装
bash Anaconda3-2025.12-1-Linux-x86_64.sh  

# 按 Enter 阅读许可协议，一直按 Space 翻页直到末尾。
# 输入 yes 同意协议。
# 选择安装路径（默认是 ~/anaconda3，直接按 Enter 即可，无需修改）。
# 关键步骤：当出现 Do you wish the installer to initialize Anaconda3 by running conda init? 时，输入 yes（这一步会自动配置环境变量，避免手动配置）。
```

安装完成后，可以验证一下是否安装成功，并且配置国内的加速镜像

https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/



```bash
➜  Downloads source ~/.zshrc
compinit:527: no such file or directory: /usr/share/zsh/vendor-completions/_docker
(base) ➜  Downloads conda --version
conda 25.11.0
(base) ➜  Downloads python --version
Python 3.13.9
(base) ➜  Downloads vim ${HOME}/.condarc

### 内容如下：
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

创建一个 ChatGM3 的运行环境

最好使用 python 3.10 - 3.11，环境过新可能有问题

```bash
conda create --name chatglm3-6b python=3.11 -y    # 创建环境
conda activate chatglm3-6b     # 激活环境
```

### 安装ChatGML依赖

```bash
cd ChatGLM3
pip install -r requirements.txt
```

### PyTorch 介绍

> 上个步骤会自动将 torch 也安装上，但是版本过低，需要处理，否则会报错 “NVIDIA GeForce RTX 5080 with CUDA capability sm_120 is not compatible with the current PyTorch installation.”

**PyTorch 是一套专门为深度学习设计的 Python 框架**，可以把它理解成模型的 “运行底座”：ChatGLM 的代码是基于 PyTorch 写的，没有它，模型的代码根本跑不起来；大模型本质上是成千上万个矩阵运算。PyTorch 提供了一个高效的环境，将这些复杂的数学公式转化为计算机（尤其是 GPU）能听懂的指令。

- **张量运算 (Tensor Operations)：** PyTorch 将数据包装成“张量”，利用算力进行大规模并行计算。
- **CUDA 加速：** PyTorch 是模型与显卡（NVIDIA GPU）之间的桥梁，它调用 CUDA 核心来极大地提高模型生成文字的速度。

需要根据 CUDA 版本选择 Pytorch 框架，比如执行 `nvidia-smi` 可以看到我使用的 CUDA 版本是 12.9 。需要安装与之匹配的 Pytorch

```bash
➜  ~ nvidia-smi
Sat Dec 20 23:25:27 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 575.64.01              Driver Version: 576.88         CUDA Version: 12.9     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 5080        On  |   00000000:02:00.0  On |                  N/A |
|  0%   57C    P3             46W /  360W |    2782MiB /  16303MiB |      2%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A             507      G   /Xwayland                             N/A      |
+-----------------------------------------------------------------------------------------+
```

需要安装支持最新显卡架构的 PyTorch 预览版或最新稳定版。我的 RTX 5080 需要 **CUDA 12.9 版本** 的支持。（默认安装的 CUDA 版本是 12.0）

```bash
# 先卸载刚才默认安装的老版本 CUDA 
pip uninstall torch torchvision torchaudio

# 重新安装新版 CUDA cu129
pip install torch torchvision torchaudio --index-url <https://download.pytorch.org/whl/cu129>
```

查看 Torch 和 CUDA 版本是否正确对应：

```bash
(chatglm3-6b) ➜  ChatGLM3 git:(main) ✗ python3 -c "import torch; print(torch.__version__)"
2.11.0.dev20251220+cu129
(chatglm3-6b) ➜  ChatGLM3 git:(main) ✗ python3 -c "import torch; print('CUDA 是否可用:', torch.cuda.is_available())"
CUDA 是否可用: True
```

# 下载模型

首先先强调一下 **下载模型 = 下载权重 + 下载分词器 + 下载配置。**

## 权重（Weights）

权重文件（Weights）里面包含的是几十个 GB 的二进制文件（如 `model-00001-of-00007.safetensors`）。

在 AI（神经网络）的世界里，权重就是一系列**数字**（通常是浮点数，比如 `0.0023` 或 `-0.145`）。

在 ChatGLM 这种 Transformer 模型中，每一层都在进行矩阵运算。权重就是矩阵中的每一个

**数值。**

**文件例子：** `model.safetensors`, `pytorch_model.bin`

## 分词器（Tokenizer）

分词器负责将用户的输入（字符串）切分成一个个 “符号” （Tokens），并转换成对应的 ID（整数数字）

**文件例子：** `tokenizer.json`, `tokenizer_config.json`

## 配置（Config）

定义模型的层数，每层有多少神经元，用哪些数学公式。

**文件例子：** `config.json`

## Hugging Face

Hugging Face 是 AI 资源的“大仓库” (The Hub)

这是它最出名的地方。在 Hugging Face 的官网上，你可以直接找到并下载：

- **Models（模型）：** 包含 ChatGLM、Llama、BERT、Stable Diffusion 等几十万个已经训练好的模型。你可以直接下载到本地运行。
- **Datasets（数据集）：** 训练 AI 离不开海量的数据（文字、图片、音频）。这里有各种现成的数据集供科研和开发使用。
- **Spaces（空间）：** 这是一个神奇的地方，很多人把 AI 模型做成了网页版的“在线试用 Demo”。你不需要写一行代码，点进去就能直接调戏各种有趣的 AI。

**huggingface-cli 是 Hugging Face 官方提供的命令行工具（CLI = Command Line Interface），**Hugging Face Hub 是全球最大的开源模型 / 数据集平台，ChatGLM、LLaMA、BERT 等主流模型都托管在这里，`huggingface-cli` 就是连接本地环境和 Hugging Face Hub 的 “桥梁”。

不过现在使用 `hf` 命令代替了。但推荐国内环境使用 `hfd` 。

**`hfd`**。它是专门为解决国内下载 Hugging Face 模型慢、易中断而设计的脚本，通过使用 `aria2c`（多线程下载工具）来代替原生的 `git clone`

以下是使用 `hfd.sh` 代替 `git clone` 下载 ChatGLM3-6B 的具体步骤：

如果网络状况很好，则 git clone 即可 `git clone <https://huggingface.co/THUDM/chatglm3-6b`>

（当然，我要使用 hfd 来实现下载）

首先 `hfd` 依赖 `aria2c` 来实现多线程加速。

- **Ubuntu/Debian:** `sudo apt-get install aria2`

下载并配置 hfd 脚本：

```bash
wget <https://hf-mirror.com/hfd/hfd.sh>
chmod a+x hfd.sh

# 设置加速镜像源
export HF_ENDPOINT=https://hf-mirror.com
```

[hf-mirror.com](http://hf-mirror.com) 是一个公益项目，致力于帮助国内AI开发者快速、稳定的下载模型、数据集。

### 使用 hfd 下载 ChatGLM3-6B

现在用它来代替 `git clone`：

```bash
./hfd.sh THUDM/chatglm3-6b --tool aria2c -x 10

...
# 最终输出
gid   |stat|avg speed  |path/URI
======+====+===========+=======================================================
d0e20c|OK  |     899B/s|./config.json
62f505|OK  |   0.9KiB/s|./.gitattributes
76bbe1|OK  |   1.0KiB/s|./configuration_chatglm.py
32ad2c|OK  |   1.9KiB/s|./MODEL_LICENSE
be32f8|OK  |   3.0KiB/s|./README.md
189d03|OK  |   2.0MiB/s|./model-00001-of-00007.safetensors
f65754|OK  |   2.0MiB/s|./model-00004-of-00007.safetensors
91c42f|OK  |   2.0MiB/s|./model-00005-of-00007.safetensors
98f196|OK  |   9.0KiB/s|./model.safetensors.index.json
023214|OK  |    44KiB/s|./modeling_chatglm.py
697e22|OK  |   1.9MiB/s|./model-00002-of-00007.safetensors
3a2c62|OK  |   1.8MiB/s|./model-00003-of-00007.safetensors
4f0c9d|OK  |   2.2MiB/s|./model-00007-of-00007.safetensors
f1d58a|OK  |   2.0MiB/s|./model-00006-of-00007.safetensors
62cd46|OK  |   2.1MiB/s|./pytorch_model-00001-of-00007.bin
3d32b6|OK  |   2.3MiB/s|./pytorch_model-00003-of-00007.bin
6f243c|OK  |   1.9MiB/s|./pytorch_model-00002-of-00007.bin
6d917d|OK  |   6.1KiB/s|./pytorch_model.bin.index.json
6eb050|OK  |    12KiB/s|./quantization.py
574e93|OK  |       7B/s|./special_tokens_map.json
0b4bc7|OK  |   4.0KiB/s|./tokenization_chatglm.py
51cfb2|OK  |   390KiB/s|./tokenizer.model
305191|OK  |   1.3KiB/s|./tokenizer_config.json
823b7a|OK  |   2.3MiB/s|./pytorch_model-00004-of-00007.bin
84d231|OK  |   3.3MiB/s|./pytorch_model-00007-of-00007.bin
ee36db|OK  |   3.7MiB/s|./pytorch_model-00005-of-00007.bin
e5e170|OK  |   3.6MiB/s|./pytorch_model-00006-of-00007.bin

Status Legend:
(OK):download completed.
```

将安装好的 `chatglm3-6b`  移动到之前的 `ChatGLM3` 项目中。

# 运行大模型

进入到最开始克隆的 `ChatGLM3` 项目中，请确保刚才下载的大模型被放到了这个目录中。

<aside> 💡

在实际运行过程中，我遇到了许多库不兼容的问题。不过将问题粘贴给 [Gemini](https://gemini.google.com/)，它都能出色地帮我解决。

</aside>

## 命令行终端对话

打开文件 basic_demo/cli_demo.py，修改模型加载路径。

或者像我一样，直接修改环境变量即可。

```bash
export MODEL_PATH="./chatglm3-6b"
python basic_demo/cli_demo.py  #运行大模型
```

![chatglm3-chat](https://img1.kiosk007.top/static/images/blog/20251223002157-chatglm3-chat.png)

## Web 对话

打开文件 basic_demo/web_demo_gradio.py，修改模型加载路径。或者和上述一样使用环境变量即可。

浏览器输入 `127.0.0.1:7870` , (或者根据自己的本地监听)

![chatglm3-web1](https://img1.kiosk007.top/static/images/blog/20251223002240-chatglm3-web1.png)

ChatGLM 还提供了一个全新的 web demo，支持 Chat、Tool、Code Interpreter，就在我们克隆下来的代码里，在文件夹 composite_demo 下。

```bash
cd composite_demo
pip install -r requirements.txt
export MODEL_PATH=../model
streamlit run main.py 或者 python -m streamlit run main.py
```

![chatglm3-web2](https://img1.kiosk007.top/static/images/blog/20251223002311-chatglm3-web2.png)

# 模型微调

想要增强大模型的能力，使其增强某一特定领域知识的能力。微调是一种方式

| 方法                             | 优点👍                                                        | 缺点👎                                                        |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 全量微调 FFT                     | • 在特定任务上可以达到最佳性能                        • 可以针对特定数据进行优化 | • 需要大量计算资源                                      • 容易过拟合，特别是在数据量较小的情况下                                                                   • 更新所有参数，导致模型大小和推理时间增加 |
| 参数高效微调 PEFT：Prompt Tuning | • 资源需求低，因为只修改少量参数                      • 可以保持预训练模型的大部分知识 | • 性能通常不如全量微调                              • 找到有效的提示可能很困难                     • 在某些任务上效果不如其他微调方法 |
| 参数高效微调 PEFT：Prefix Tuning | • 同样适用于较小的参数更新量                            • 可以为每个任务自定义前缀，保持预训练模型结构不变 | • 性能可能不如全量微调或其他方法          • 需要为每个任务设计和调整前缀              • 在复杂任务上可能不能够有效 |
| 参数高效微调 PEFT：LoRA          | • 参数数量相对较少，减少了计算和存储需求     • 保持了模型的可解释性和灵活性 | • 性能可能略逊于全量微调但通常优于其他参数效率方法                                             • 需要仔细选择要调整的参数和适应的低秩结构                                                               • 可能需要特定的初始化和调整策略 |
| 参数高效微调 PEFT：QLoRA         | • 在 LoRA 的基础上进一步减少参数数量，提高参数效率                                                                      • 旨在减小计算和存储需求，同时尽可能保持性能                                                                                  • 可以在不牺牲太多性能的情况下提高速度和效率 | • 相比原始 LoRA 可能有轻微性能下降       • 适用性和优化策略可能需要根据具体任务调整                                                               • 实施和调整可能比标准 LoRA 更复杂 |
| 知识库                           | • 知识准确，推理性能好                                         • 比微调更新频率更快，可以随时补充知识 | • 知识需要提前维护好                                         |
| API                              | • 和知识库类似，有较好的推理性能                      • 知识可以随时更新、调整 | • 知识需要提前维护好                                         |

## 法律助手模型微调 — 需求分析

法律小助手用来帮助员工解决日常生活中遇到的法律问题，以问答的方式进行，这种场景可以使用知识库模式，也可以使用微调模式。使用知识库模式的话，需要将数据集拆分成一条一条的知识，先放到向量库，然后通过 Agent 从向量库检索，再输入给大模型，这种方式的好处是万一我们发现数据集不足，可以随时补充，即时生效。还有一种方式就是进行微调，因为法律知识有的时候需要一定的逻辑能力，不是纯文本检索，而微调就是这样的，通过在一定量的数据集上的训练，增加大模型法律相关的常识及思维，从而进行推理。经过分析，我们确定下来，使用微调的方式进行。接下来就是准备数据了。

## 数据集格式

从 [GitHub](https://github.com/SophonPlus/ChineseNlpCorpus/blob/master/datasets/lawzhidao/intro.ipynb) 上面找了一个公开的数据集

官方给的微调数据格式是 CSV，格式如下：

```bash
{"conversations": [{"role": "user", "content": "类型#裙*裙长#半身裙"}, {"role": "assistant", "content": "这款百搭时尚的仙女半身裙，整体设计非常的飘逸随性，穿上之后每个女孩子都能瞬间变成小仙女啦。料子非常的轻盈，透气性也很好，穿到夏天也很舒适。"}]}
```

所以需要将下载的数据转换成官方给定的格式。用如下的代码做转换。

```bash
import csv
import json

# CSV文件的路径
csv_file_path = 'lawzhidao_filter.csv'
# 输出JSON文件的路径
json_file_path = 'output_json_file.json'

# 打开CSV文件，并进行处理
with open(csv_file_path, mode='r', encoding='utf-8') as csv_file, \\
     open(json_file_path, mode='w', encoding='utf-8') as json_file:

    csv_reader = csv.DictReader(csv_file)
    for row in csv_reader:
        # 根据CSV文件的列名获取数据
        title = row['title']
        reply = row['reply']
        # 构造单条对话的JSON结构
        conversation_entry = {
            "conversations": [
                {"role": "user", "content": title},
                {"role": "assistant", "content": reply}
            ]
        }
        # 将单条记录以JSON格式写入文件，每条记录一行
        json_line = json.dumps(conversation_entry, ensure_ascii=False)
        json_file.write(json_line + '\\n')
```

## 微调参数

**配置文件**

微调的参数统一放置在`finetune_demo/configs`下，将会有四个文件

```bash
.
├── deepspeed.json
├── lora.yaml
├── ptuning_v2.yaml
└── sft.yaml
```

**参数解释**

| 配置项                | 子参数                      | 描述                   | 值                  |
| --------------------- | --------------------------- | ---------------------- | ------------------- |
| **data_config**       | train_file                  | 训练数据文件路径       | train.json          |
|                       | val_file                    | 验证数据文件路径       | dev.json            |
|                       | test_file                   | 测试数据文件路径       | dev.json            |
|                       | num_proc                    | 数据处理的工作进程数   | 16                  |
| **max_input_length**  | --                          | 输入序列的最大长度     | 128                 |
| **max_output_length** | --                          | 输出序列的最大长度     | 256                 |
| **training_args**     | output_dir                  | 模型输出目录           | ./output            |
|                       | max_steps                   | 最大训练步骤           | 3000                |
|                       | per_device_train_batch_size | 每个设备的训练批次大小 | 1                   |
|                       | dataloader_num_workers      | 数据加载器的工作进程数 | 16                  |
|                       | remove_unused_columns       | 是否删除未使用的列     | false               |
|                       | save_strategy               | 保存检查点的策略       | steps               |
|                       | save_steps                  | 保存检查点的步骤间隔   | 500                 |
|                       | log_level                   | 日志级别               | info                |
|                       | logging_strategy            | 日志记录策略           | steps               |
|                       | logging_steps               | 日志记录的步骤间隔     | 10                  |
|                       | per_device_eval_batch_size  | 每个设备评估批次大小   | 16                  |
|                       | evaluation_strategy         | 评估策略               | steps               |
|                       | eval_steps                  | 评估的步骤间隔         | 500                 |
|                       | predict_with_generate       | 是否使用生成进行预测   | true                |
|                       | generation_config           | 生成配置               | max_new_tokens: 256 |
| **peft_config**       | peft_type                   | 微调类型               | LORA                |
|                       | task_type                   | 任务类型               | CAUSAL_LM           |
|                       | r                           | Lora矩阵的秩           | 8                   |
|                       | lora_alpha                  | Lora的alpha值          | 32                  |
|                       | lora_dropout                | Lora的dropout率        | 0.1                 |

由于本人的 5080 显存 16G 太小了，在微调过程中 显存总是爆掉。所以对 lora.yaml 做了一些修改，调小相关参数

```jsx
data_config:
  train_file: train.json
  val_file: dev.json
  test_file: dev.json
  num_proc: 16
max_input_length: 128  # 减少输入长度
max_output_length: 128  # 减少输出长度
training_args:
  # see `transformers.Seq2SeqTrainingArguments`
  output_dir: ./output
  max_steps: 3000
  learning_rate: 1e-5  # 保持较小的学习率
  per_device_train_batch_size: 1  # 减少到 1，进一步节省显存
  per_device_eval_batch_size: 1  # 同样减小 eval 的 batch size
  gradient_accumulation_steps: 16  # 增加梯度累积步数，弥补小 batch_size
  dataloader_num_workers: 8  # 减少 dataloader 的 worker 数量，节省显存
  remove_unused_columns: false
  save_strategy: steps
  save_steps: 500
  log_level: info
  logging_strategy: steps
  logging_steps: 10
  evaluation_strategy: steps
  eval_steps: 500
  predict_with_generate: true
  generation_config:
    max_new_tokens: 512
  use_cpu: false
  fp16: true  # 启用 FP16 混合精度训练
  gradient_checkpointing: true  # 启用梯度检查点
  bf16: false  # 禁用 bf16，因为某些硬件可能不支持

peft_config:
  peft_type: LORA
  task_type: CAUSAL_LM
  r: 8
  lora_alpha: 32
  lora_dropout: 0.1
```

## 开始微调

先安装2个依赖：

`nltk` 最常用的**自然语言处理（NLP）** 库 `libopenmpi-dev` 是 Linux 下的高性能并行计算库

```bash
pip3 install nltk           
sudo apt-get install libopenmpi-dev
```

准备训练数据，把 “准备数据” 环节格式化好的文件命名为 train.json，再准备一个相同格式的测试数据集 dev.json，在里面添加一些测试数据，得大于50条，不然会报错，否则修改源代码，将50 改小点。

```bash
eval_dataset=val_dataset.select(list(range(50))),
```

微调脚本用的是 `finetune_hf.py`。

第一个参数是训练数据集所在目录，此处值是 `data`。

第二个参数是模型所在目录，此处值是 `../chatglm-6b`。

第三个参数是微调配置，此处值是 `configs/lora.yaml`。执行微调命令，记得 Python 命令使用全路径。

```bash
/usr/bin/python3 finetune_hf.py data ../model configs/lora.yaml
```

![chatglm3-finetune](https://img1.kiosk007.top/static/images/blog/20251223002402-chatglm3-finetune.png)



## 验证

等待微调结束，就可以进行验证了，官方 demo 提供了验证脚本，执行如下命令：

```jsx
/usr/bin/python3 inference_hf.py output/checkpoint-3000/ --prompt "xxxxxxxxxxxx"
```

output/checkpoint-3000 是指新生成的权重，模型启动的时候会将原模型和新权重全部加载，然后进行推理。–prompt 是输入的提示。