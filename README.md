# Lightweight Agent

`Lightweight Agent` 是一个极简的 LLM 代理实现：它可以用代码编写“动作”来完成任务。重点在于简洁与教学。`Lightweight Agent` 展示了：LLM 代理背后的核心思想其实很简单。在当今先进的 LLM 支持下，从零实现一个基础代理并不是什么“火箭科学”。

当然，这种侧重也意味着取舍。例如，相比人们希望在一个完整的、可用于生产的库中看到的情况，这里减少了校验步骤，错误处理也没有那么“周到”，对边界情况的覆盖也更少。但考虑到上面提到的目标，这完全没问题——而且确实带来收益。核心的 `agent.py` 模块（不包含大量注释）只有大约 100 行代码。

代理本身从零构建，只使用了少量库，并配有大量注释。不过，它受到了 [Hugging Face 的 Smolagents 库](https://github.com/huggingface/smolagents/tree/main) 的启发。

代码代理需要一个受控且隔离的环境来运行它生成的代码。然而，创建这种环境并不算真正的核心 AI 难题；如果在这里从零实现，会带来很小的价值。因此，本仓库依赖 Smolagents 的 [本地 Python 执行器](https://github.com/huggingface/smolagents/blob/main/src/smolagents/local_python_executor.py) 来进行本地代码执行。

## 用法

### 克隆与安装

1. 安装 [`uv`](https://docs.astral.sh/uv/getting-started/installation/)，用于 Python 打包与环境创建。

2. 克隆仓库：
   ```bash
   git clone https://github.com/kyy338/Lightweight-Agent.git
   cd minimal-agent
   ```

3. 配置你的模型与 API 密钥：
   
   本仓库使用 [LiteLLM](https://docs.litellm.ai/) 来为所有模型提供商提供统一接口。请在 `Lightweight Agent` 文件夹中创建一个 `.env` 文件，并加入 `MODEL` 环境变量（其值使用 LiteLLM 中对应模型的名称），同时添加对应提供商所需的凭据。
   
   AWS Bedrock 示例（*）：
   ```bash
   # AWS Bedrock：https://docs.litellm.ai/docs/providers/bedrock
   AWS_ACCESS_KEY_ID=<YOUR-AWS-ACCESS-KEY-ID>
   AWS_SECRET_ACCESS_KEY=<YOUR-AWS-SECRET-ACCESS-KEY>
   AWS_REGION_NAME=<YOUR-AWS-REGION-NAME>

   MODEL="bedrock/anthropic.claude-3-7-sonnet-20250219-v1:0"
   ```

   Google Gemini 示例：
   ```bash
   # Google Gemini：https://docs.litellm.ai/docs/providers/gemini
   GEMINI_API_KEY=<YOUR-API-KEY>

   MODEL="gemini/gemini-2.0-flash"
   ```

   其他模型提供商请参考 [LiteLLM 文档](https://docs.litellm.ai/docs/providers)。你可能需要为特定提供商额外安装一些依赖（这些在 LiteLLM 文档中有说明）。你可以使用 `uv add <your-python-package>` 来添加。例如，Bedrock 模型需要 `boto3`（该依赖已被添加）。如果你不使用 AWS 提供的模型，也可以移除 `boto3`。
   
   注意：代码类代理需要性能更强的 LLM，例如 Claude 3.7 Sonnet、Amazon Nova Pro 或 Gemini 2.0 Flash。虽然你也可以用较弱的模型运行这段代码，但结果可能不会很理想。

请注意，`run_agent.py` 中实现的代理默认使用 DuckDuckGo 作为网页搜索工具。DuckDuckGo 的好处是不需要 API 密钥，这意味着你可以在不做额外配置的情况下直接运行代理。缺点是你可能会很快遇到速率限制。为了避免这些限制，你也可以把网页搜索从 DuckDuckGo 替换为 [Tavily](https://www.tavily.com/)。具体做法是：替换 `run_agent.py` 里的搜索工具、创建一个 [Tavily](https://www.tavily.com/) 账号、获取 API 密钥并把它添加到你的 `.env` 文件中：
   ```bash
   TAVILY_API_KEY=<YOUR-TAVILY-API-KEY>
   ```

4. 运行示例：
   ```bash
   uv run run_agent.py
   ```

(*) 注意：与通过 boto3（LiteLLM 也支持）使用访问密钥相比，AWS 还有更好的 [认证方式](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html#credentials)。之所以在上面给出这种方式，是因为它是 LiteLLM 文档中提到的第一种方案，并且能在不同提供商之间保持一致。

### 用法示例

默认示例会查询“2024 年最热的一天以及该天的道琼斯指数数值”，并让代理具备联网搜索以及访问网页来查找答案的能力。

你可以在 `run_agent.py` 中修改任务：
```python
res = agent.run("<在这里用自然语言描述你的任务>")
```

## 代理架构

该架构遵循 [ReAct 框架](https://arxiv.org/abs/2210.03629)。代理会用一系列步骤来完成任务。在每一步中，它都可以进行“Reason（推理）”与“Act（行动）”（也就是 ReAct）。代理会执行尽可能多的步骤来完成任务。

因此，核心想法既简单又优雅，而 `Lightweight Agent` 就是按这个思路实现的。下面是架构图：
![Uploading image.png…]()


查看 `src/minimal_agent/agents.py` 以了解对应的源码实现。


如果想更详细地了解代理，并理解不同层级的 AI 代理，Hugging Face 的 [AI 代理课程](https://huggingface.co/learn/agents-course/en/unit1/what-are-agents) 是一个很好的起点。

请注意，许多 AI 代理框架（会允许）对 ReAct 框架进行抽象。然而，实际的调用图通常要复杂得多，因为它们要么增加了额外能力，要么不得不引入组件以适配更通用的场景（例如更完善的错误处理等）。

这当然很有用。正因为缺少这些内容，本仓库中的代码可能会在某些任务上失败。

不过，对于教学目的，甚至是针对任务范围非常窄的更具体使用场景，从更简单的代码开始并根据需求进行定制，通常比去改造一个更复杂的框架更有效。
