# mini-cc Python 版学习指南

本指南将带你从零开始，逐步深入理解一个 AI Agent 的完整工作原理。你不需要任何 AI Agent 开发经验，只需要有基本的 Python 基础。

---

## 项目全景概览

### mini-cc 是什么？

mini-cc 是 Anthropic 官方 Claude Code CLI 的**教学精简版**。它用最少的代码实现了一个能自主编写代码的 AI Agent。Python 版大约 **1,700 行代码**，20 个源文件，你可以在一个下午读完所有代码。

### 它能做什么？

```
你：帮我写一个 Python 的快排算法
Agent：好的，我来帮你创建文件... [调用 FileWriteTool 写入文件]
Agent：我再运行一下测试... [调用 BashTool 执行 python quicksort.py]
Agent：测试通过了！代码已经写到 ../test_file/quicksort.py
```

这就是 Agent 的核心能力：**自主思考 → 调用工具 → 观察结果 → 继续思考**，直到任务完成。

### 核心概念一图流

```
┌──────────────────────────────────────────────────────────┐
│                     用户终端 (REPL)                        │
│                   mini-cc> 帮我写个快排                    │
└──────────────────────┬───────────────────────────────────┘
                       │ 用户输入
                       ▼
┌──────────────────────────────────────────────────────────┐
│                     Agent (智能体)                         │
│                                                          │
│   ┌─────────┐    ┌───────────┐    ┌─────────────────┐   │
│   │ 用户消息  │───▶│  大模型API  │───▶│  解析响应/工具调用 │   │
│   └─────────┘    └───────────┘    └────────┬────────┘   │
│                       ▲                         │         │
│                       │                         ▼         │
│   ┌─────────────┐    │              ┌──────────────┐     │
│   │ 工具执行结果  │────┘              │  执行工具函数  │     │
│   └─────────────┘    (循环直到       └──────────────┘     │
│                       模型不再                         │
│                       调用工具)                          │
└──────────────────────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌───────────┐
    │ BashTool │ │FileRead  │ │FileWrite  │
    │ 执行命令  │ │读取文件   │ │写入文件    │
    └──────────┘ └──────────┘ └───────────┘
```

---

## 学习路线图

整个学习过程分为 **8 个阶段**，每个阶段聚焦一个核心概念，建议按顺序进行。

| 阶段 | 主题 | 核心文件 | 你将学到 |
|------|------|---------|---------|
| 1 | 项目启动与配置 | `main.py`, `src/config.py` | CLI 入口、配置管理、环境变量 |
| 2 | Provider 抽象层 | `src/core/providers/base.py` | 接口设计、依赖倒置原则 |
| 3 | OpenAI 流式调用 | `src/core/providers/openai_provider.py` | 流式输出、Tool Call 解析 |
| 4 | Anthropic 流式调用 | `src/core/providers/anthropic_provider.py` | 另一种 API 的适配方式 |
| 5 | 工具系统 | `src/tools/` 全部文件 | Tool Use 机制、安全沙箱 |
| 6 | Agent 核心循环 | `src/core/agent.py` | ReAct 循环——Agent 的心脏 |
| 7 | REPL 交互 | `src/main.py`, `src/agent/loop.py` | 完整的 Agent 运行方式对比 |
| 8 | 测试与动手 | `tests/` | 单元测试、动手扩展 |

---

## 阶段 1：项目是怎么启动的？

### 阅读文件
- `python/main.py` — 程序入口（77 行）
- `python/src/config.py` — 配置管理（97 行）

### 关键知识点

#### 1.1 两个 main.py？

项目里有两个 `main.py`：
- **根目录的 `main.py`**：使用 `argparse` 处理 CLI 子命令（如 `config set`），负责配置管理，然后调用 `main_loop()`
- **`src/main.py`**：直接从环境变量读取配置，手动构建 Provider 和 Agent，用原生 `input()` 做 REPL

它们是**两种不同的启动方式**，都能跑起来。根目录的版本更完善（有首次引导、命令行参数），`src/main.py` 更直观（能看到整个启动流程的每一步）。

**建议**：先读懂 `src/main.py`（逻辑更清晰），再看根目录的 `main.py`（工程化更好）。

#### 1.2 配置优先级

```python
# src/config.py 第 42-54 行
def get_config_value(key: str) -> str:
    # 优先级：环境变量 (含 .env) > 全局 config.json
    env_val = os.environ.get(key)
    if env_val:
        return env_val
    config_data = read_config()
    return config_data.get(key, "")
```

这是配置管理的经典模式：环境变量 > 配置文件 > 默认值。`python-dotenv` 库会自动把 `.env` 文件加载到 `os.environ`。

#### 1.3 异步入口

```python
# src/main.py 最后一行
asyncio.run(main())
```

整个 Agent 是基于 `asyncio` 构建的，因为：
- 网络请求（调用大模型 API）是 I/O 密集型
- 工具执行（如 Bash 命令）可能很耗时
- 异步可以避免"等一个工具执行完才能做其他事"

### 动手练习

1. 运行 `python python/main.py`，观察首次引导流程
2. 尝试 `python python/main.py config set OPENAI_API_KEY sk-xxx`
3. 检查 `~/.mini-cc/config.json` 文件的内容

---

## 阶段 2：Provider 抽象层——让 Agent 不绑定任何一家大模型

### 阅读文件
- `python/src/core/providers/base.py` — 抽象基类（37 行）

### 关键知识点

#### 2.1 为什么需要抽象层？

OpenAI、Anthropic、国内的通义千问、DeepSeek……每家的 API 格式都不同。如果 Agent 直接调用 OpenAI 的 SDK，想换成 Claude 就得大改代码。

**解决方法**：定义一个统一的接口（`LLMProvider`），让 Agent 只依赖这个接口，不依赖具体的 SDK。

```python
class LLMProvider:
    async def send_message(self, user_message, on_text_response):
        """发送用户消息，获取模型回复"""
        raise NotImplementedError

    async def send_tool_results(self, results, on_text_response):
        """把工具执行结果发回给模型"""
        raise NotImplementedError
```

这就是**依赖倒置原则 (DIP)**：高层模块（Agent）不依赖低层模块（具体 SDK），两者都依赖抽象。

#### 2.2 两个方法的设计意图

| 方法 | 什么时候调用 | 返回什么 |
|------|------------|---------|
| `send_message` | 用户发了一条新消息 | `{"text": "...", "toolCalls": [...]}` |
| `send_tool_results` | 工具执行完了，把结果发回模型 | 同上 |

注意返回值的统一格式——无论底层是 OpenAI 还是 Anthropic，Agent 拿到的都是同样的结构。

### 动手练习

1. 画出 `Agent → LLMProvider → OpenAIProvider / AnthropicProvider` 的依赖关系
2. 思考：如果要接入 DeepSeek，你需要做什么？（答案：写一个 `DeepSeekProvider` 继承 `LLMProvider`）

---

## 阶段 3：OpenAI 流式调用——Agent 的"嘴巴"和"耳朵"

### 阅读文件
- `python/src/core/providers/openai_provider.py` — OpenAI 兼容 Provider（171 行）

**这是整个项目中最核心、最复杂的文件之一**，建议仔细读。

### 关键知识点

#### 3.1 系统提示词 (System Prompt)

```python
system_prompt = '你是一个名为 mini-cc 的高级 AI 编程助手...'
self.messages.append({"role": "system", "content": system_prompt})
```

System Prompt 定义了 AI 的"人设"。对于 OpenAI 接口，它混在 `messages` 数组里；对于 Anthropic，它是一个单独的参数（后面会看到）。

#### 3.2 工具描述的转换

```python
def get_tools(self):
    return [{
        "type": "function",
        "function": {
            "name": t["name"],
            "description": t["description"],
            "parameters": t["inputSchema"]
        }
    } for t in tools]
```

Agent 内部用 `inputSchema` 描述工具参数，OpenAI API 用 `parameters`。这个方法做了格式转换。

#### 3.3 流式输出 (Streaming)——最精华的部分

```python
stream = await self.client.chat.completions.create(
    stream=True,  # 关键：开启流式
    ...
)

async for chunk in stream:  # 逐块接收
    delta = chunk.choices[0].delta
```

流式输出就是"边生成边返回"，而不是等全部生成完再一次性返回。这样用户能实时看到 AI 的输出，体感好很多。

每个 `chunk` 是一个小片段，可能包含三种内容：

| 内容类型 | 字段 | 说明 |
|---------|------|------|
| 思考过程 | `delta.reasoning_content` | Qwen/DeepSeek 的思维链 |
| 正常文本 | `delta.content` | AI 的回复文本 |
| 工具调用 | `delta.tool_calls` | AI 想要调用的工具 |

#### 3.4 流式工具调用的拼接

这是最容易出错的地方。工具调用的参数在流式传输中被切成很多片段：

```
chunk 1: {"name": "BashTool", "arguments": ""}
chunk 2: {"name": "", "arguments": "{\"co"}
chunk 3: {"name": "", "arguments": "mmand"}
chunk 4: {"name": "", "arguments": "\": \"ls\"}"}
```

代码用 `tool_calls_map` 字典按 `index` 拼接：

```python
if tc.function and tc.function.arguments:
    tool_calls_map[idx]["function"]["arguments"] += tc.function.arguments
```

最后用 `json.loads()` 解析拼接好的完整 JSON。

#### 3.5 JSON 解析失败的自愈机制

大模型有时会生成格式错误的 JSON。代码做了两层防护：

```python
try:
    args = json.loads(raw_args)           # 直接解析
except json.JSONDecodeError:
    raw_args = self.fix_json_string(raw_args)  # 修复换行符
    args = json.loads(raw_args)
except Exception:
    args = {"_parse_error": True, ...}    # 打上错误标记
```

打上 `_parse_error` 标记后，Agent 层会把错误信息反馈给大模型，让它自我纠正。

### 动手练习

1. 在 `create_message` 方法中，加一个 `print` 打印每个 chunk 的类型，观察流式输出的真实结构
2. 思考：为什么 `tool_calls_map` 用 `index` 而不是 `id` 作为 key？（答案：流式传输中 `id` 可能延迟到达）

---

## 阶段 4：Anthropic 流式调用——同一种思想，不同的实现

### 阅读文件
- `python/src/core/providers/anthropic_provider.py` — Anthropic Provider（134 行）

### 关键知识点

#### 4.1 与 OpenAI 的关键差异

| 差异点 | OpenAI | Anthropic |
|--------|--------|-----------|
| System Prompt | 放在 `messages` 数组里 | 作为独立参数 `system=` |
| 工具调用结果 | `role: "tool"` 消息 | 包装在 `role: "user"` 消息里 |
| 工具调用解析 | 需要手动拼接流式片段 | SDK 自动拼接 |
| content 结构 | 纯字符串 | 数组（可同时包含文本和工具调用） |

#### 4.2 SDK 自动拼接的优势

```python
async with self.client.messages.stream(...) as stream:
    async for event in stream:
        if event.type == "text_delta":
            on_text_response(event.delta.text, False)
        elif event.type == "tool_use":
            pass  # 不需要手动拼接！

    final_message = await stream.get_final_message()  # SDK 帮你拼好了
```

Anthropic SDK 的 `stream.get_final_message()` 自动完成了工具调用的拼接，不需要像 OpenAI 那样手动处理。

#### 4.3 工具结果的特殊包装

```python
# Anthropic 要求把工具结果包在 role="user" 的消息中
self.messages.append({
    "role": "user",
    "content": tool_result_content  # 数组，每个元素是一个 tool_result
})
```

这是 Anthropic API 的特殊要求——工具执行结果必须以 `user` 角色发送，每个结果包含 `tool_use_id` 来关联之前的工具调用。

### 动手练习

1. 对比两个 Provider 的 `send_tool_results` 方法，列出所有结构差异
2. 思考：如果要支持 Google Gemini，Provider 需要适配哪些差异？

---

## 阶段 5：工具系统——Agent 的"手和脚"

### 阅读文件
- `python/src/tools/base.py` — 工具基类（42 行）
- `python/src/tools/bash_tool.py` — Bash 工具（101 行）
- `python/src/tools/file_read_tool.py` — 文件读取工具（58 行）
- `python/src/tools/file_write_tool.py` — 文件写入工具（61 行）
- `python/src/tools/__init__.py` — 工具注册（9 行）
- `python/src/tools/registry.py` — 工具注册表（47 行，另一种实现方式）

### 关键知识点

#### 5.1 工具的两套实现

Python 版中存在两套工具实现：

**方案 A：字典式（`bash_tool.py` 等带 `_tool` 后缀的文件）**
```python
bash_tool = {
    "name": "BashTool",
    "description": "...",
    "inputSchema": {...},
    "execute": execute_bash  # 函数引用
}
```

**方案 B：面向对象式（`bash.py` 等不带后缀的文件，继承 `BaseTool`）**
```python
class BashTool(BaseTool):
    name = "BashTool"
    description = "..."
    args_schema = BashArgs  # Pydantic 模型

    async def execute(self, command: str) -> str:
        ...
```

方案 A 被 `src/core/agent.py` 使用（核心路径），方案 B 被 `src/agent/loop.py` 的 `ToolRegistry` 使用。两套方案各有优劣：

| 维度 | 方案 A（字典） | 方案 B（OOP） |
|------|--------------|-------------|
| 简单程度 | 更简单直观 | 需要理解继承 |
| Schema 生成 | 手写 JSON Schema | Pydantic 自动生成 |
| 类型安全 | 弱（dict 参数） | 强（有类型注解） |
| 扩展性 | 一般 | 好（可以加通用逻辑到基类） |

#### 5.2 工具描述——大模型决定调用哪个工具的唯一依据

```python
"description": """
    在本地系统执行 Bash/Shell 命令。
    注意：
    - 命令是无交互式的，请避免运行 vim, nano
    - 始终使用绝对路径
    - 如果命令可能会产生大量输出，请使用 head 截断
"""
```

这段描述是大模型决定**什么时候用、怎么用**这个工具的唯一参考。写得越清楚，模型调用就越准确。这是 Agent 开发中最被低估的技能——**写好工具描述**。

#### 5.3 安全沙箱

`bash_tool.py` 包含了一个简易但有效的安全机制：

```python
DANGEROUS_PATTERNS = [
    re.compile(r'rm\s+-r[fF]?\s+/'),      # rm -rf /
    re.compile(r'mkfs\.'),                  # 格式化磁盘
    re.compile(r'dd\s+if=.*of=/dev/sda'),   # 覆写磁盘
]

COMMAND_SUBSTITUTION_PATTERNS = [
    re.compile(r'\$\([^)]+\)'),  # $(...)
    re.compile(r'`[^`]+`'),      # `...`
]
```

在执行任何命令前，先用正则匹配检查安全性。如果命中高危模式，直接拒绝并返回错误信息。

**局限性**：这只是正则匹配，不是真正的沙箱。一个高级的 Agent 会用 Docker 容器或 OS 级别的沙箱。

#### 5.4 上下文瘦身（Microcompact）

```python
# agent.py 第 62 行
if isinstance(result, str) and len(result) > 8000:
    result = result[:8000] + '\n\n...[已被截断]...'
```

如果工具返回的结果太长（比如 `ls` 一个几万文件的目录），会截断到 8000 字符。这是因为大模型有上下文窗口限制（如 128K tokens），必须控制每次返回的数据量。

### 动手练习

1. 写一个新工具 `ListDirTool`，功能是列出指定目录下的文件和文件夹
2. 给 BashTool 加一个新的危险命令模式：拦截 `curl | bash` 这种管道注入
3. 思考：如果把 8000 字符的截断阈值改成可配置的，应该怎么设计？

---

## 阶段 6：Agent 核心循环——Agent 的心脏

### 阅读文件
- `python/src/core/agent.py` — Agent 类（115 行）

**这是整个项目最核心的文件**。它只有 115 行，但实现了 Agent 的灵魂。

### 关键知识点

#### 6.1 ReAct 循环（Reasoning + Acting）

ReAct 是目前最主流的 Agent 架构模式。核心思想很简单：

```
1. 用户输入 → 发给大模型
2. 大模型回复文本 + 工具调用
3. 如果有工具调用 → 执行工具 → 把结果发回大模型 → 回到步骤 2
4. 如果没有工具调用 → 任务完成 → 等待下一次用户输入
```

```python
async def chat(self, user_message, on_text_response):
    # 第一步：发送用户消息
    response = await self.provider.send_message(user_message, on_text_response)

    # 循环：只要有工具调用就继续
    while response.get("toolCalls") and len(response["toolCalls"]) > 0:
        # 执行工具
        tool_results = await self.handle_tool_calls(response["toolCalls"])
        # 把结果发回模型
        response = await self.provider.send_tool_results(tool_results, on_text_response)
```

这就是全部！整个 AI Agent 的核心只有这么几行。

#### 6.2 安全熔断机制

```python
max_loops = 3  # 最多循环 3 次

while ...:
    loop_count += 1
    if loop_count > max_loops:
        print("工具调用循环次数过多，已强制终止。")
        break
```

防止模型陷入死循环（比如不断调用失败又不断重试）。生产环境中 Claude Code 的 max_loops 远大于 3，但教学版用 3 就够了。

#### 6.3 错误反馈而非崩溃

Agent 的设计哲学是：**永远不要因为工具错误而崩溃，而是把错误信息反馈给模型让它自己处理**。

```python
# 工具参数 JSON 解析失败
if args.get("_parse_error"):
    results.append({
        "result": f"你输出的工具参数 JSON 格式不合法...",
        "isError": True
    })

# 模型幻觉——调用了不存在的工具
if not tool:
    results.append({
        "result": f"未知的工具调用: {call['name']}",
        "isError": True
    })

# 工具执行异常
except Exception as e:
    results.append({
        "result": f"执行工具 {call['name']} 时出错: {str(e)}",
        "isError": True
    })
```

这三层错误处理确保了 Agent 的健壮性。

### 动手练习

1. 画出一次完整的对话流程：用户说"帮我写个 hello world"到最终回复的所有步骤
2. 把 `max_loops` 改成 1，观察 Agent 的行为变化
3. 思考：如果在循环中加入一个 `print(messages)` 来打印上下文，你能看到什么？

---

## 阶段 7：两种 REPL 实现

### 阅读文件
- `python/src/main.py` — 简单 REPL（150 行）
- `python/src/agent/loop.py` — 完善 REPL（143 行）
- `python/src/agent/llm.py` — LLM 客户端封装（74 行）

### 关键知识点

#### 7.1 两种实现的差异

| 维度 | `src/main.py` | `src/agent/loop.py` |
|------|-------------|-------------------|
| Provider | 核心层的 `LLMProvider` | 代理层的 `LLMClient` |
| 工具系统 | `src/tools/__init__.py` 的字典列表 | `src/tools/registry.py` 的 `ToolRegistry` |
| 上下文管理 | Provider 内部管理 `messages` | `loop.py` 自己管理 `messages` |
| 输入库 | 原生 `input()` | `prompt_toolkit` |
| 清空上下文 | 重新创建 Agent 实例 | 切片 `messages` 数组 |

#### 7.2 `loop.py` 的完整 Agent 循环

`loop.py` 中的循环没有经过 Agent 类封装，而是直接在 `while True` 中操作 `messages`：

```python
while True:
    response = await llm.chat(messages)

    # 构建 assistant 消息并存入 messages
    ai_msg = {"role": "assistant"}
    if response["content"]:
        ai_msg["content"] = response["content"]
    if response["tool_calls"]:
        ai_msg["tool_calls"] = [...]  # 必须原样保存工具调用信息
    messages.append(ai_msg)

    # 没有工具调用就结束
    if not response["tool_calls"]:
        break

    # 执行工具，把结果存入 messages
    for tc in response["tool_calls"]:
        tool_result = await registry.execute_tool(tc["name"], tc["arguments"])
        messages.append({
            "role": "tool",
            "tool_call_id": tc["id"],
            "content": tool_result
        })
```

这种写法让你能直接看到 `messages` 数组是如何构建的，非常适合理解 OpenAI 的对话 API 规范。

**关键细节**：`tool_calls` 信息必须原样存回 `messages`，这是 OpenAI API 的硬性要求。如果遗漏了这一步，API 会返回错误。

#### 7.3 `llm.py` —— OpenAI SDK 的简单封装

```python
class LLMClient:
    async def chat(self, messages: list) -> dict:
        response = await self.client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            tools=registry.get_all_schemas(),  # 告诉模型有哪些工具可用
            tool_choice="auto"  # 模型自行决定是否调用工具
        )
```

`tool_choice="auto"` 的意思是"模型自己决定是直接回复文本还是调用工具"。你也可以设成 `"none"`（禁止调用工具）或 `"required"`（必须调用工具）。

### 动手练习

1. 用两种方式分别启动 mini-cc，对比它们的差异
2. 在 `loop.py` 中加一个 `/history` 命令，打印当前的 `messages` 数组长度
3. 思考：`src/main.py` 把上下文管理交给 Provider，`loop.py` 自己管理——哪种更好？为什么？

---

## 阶段 8：测试与动手扩展

### 阅读文件
- `python/tests/test_agent.py` — Agent 测试（54 行）
- `python/tests/test_bash_tool.py` — Bash 工具测试（27 行）
- `python/tests/test_file_tools.py` — 文件工具测试（36 行）

### 关键知识点

#### 8.1 Mock Provider 测试模式

```python
class MockProvider(LLMProvider):
    """假的 Provider，不调用真实 API"""
    def __init__(self):
        self.loop_count = 0

    async def send_message(self, user_message, on_text_response):
        self.loop_count += 1
        return {
            "text": "Sure, I will execute.",
            "toolCalls": [{"id": "call_1", "name": "BashTool", ...}]
        }

    async def send_tool_results(self, results, on_text_response):
        self.loop_count += 1
        return {"text": "Done.", "toolCalls": []}  # 第二轮停止

async def test_agent_loop():
    provider = MockProvider()
    agent = Agent(provider)
    await agent.chat("Do something", on_text)
    assert provider.loop_count == 2  # 验证循环了 2 次
```

这是测试 Agent 的标准方法：用 Mock 替换掉真实的 API 调用，只测试 Agent 的循环逻辑是否正确。

#### 8.2 安全测试

```python
async def test_bash_tool_dangerous():
    dangerous_cmds = [
        "rm -rf /",
        "mkfs.ext4 /dev/sda1",
        "dd if=/dev/zero of=/dev/sda",
        "echo $(ls)",
        "echo `ls`"
    ]
    for cmd in dangerous_cmds:
        res = await execute_bash({"command": cmd})
        assert "安全沙盒拒绝" in res or "安全沙盒拦截" in res
```

安全测试用黑名单方式验证：给定危险命令，验证它们确实被拦截了。

### 运行测试

```bash
cd python
pip install pytest pytest-asyncio
pytest tests/ -v
```

---

## 附录 A：文件速查表

按阅读顺序排列所有 Python 源文件：

```
python/
├── main.py                          # [77行]  程序入口，argparse CLI
├── requirements.txt                 # 依赖列表
├── src/
│   ├── main.py                      # [150行] 简单 REPL，手动构建 Agent
│   ├── config.py                    # [97行]  配置管理（.env + config.json）
│   ├── utils/
│   │   └── console.py               # [23行]  Rich 终端输出
│   ├── core/
│   │   ├── agent.py                 # [115行] ★ 核心：ReAct 循环
│   │   └── providers/
│   │       ├── base.py              # [37行]  Provider 抽象基类
│   │       ├── openai_provider.py   # [171行] OpenAI 流式调用
│   │       └── anthropic_provider.py# [134行] Anthropic 流式调用
│   ├── tools/
│   │   ├── base.py                  # [42行]  工具基类（Pydantic 版）
│   │   ├── registry.py              # [47行]  工具注册表（Pydantic 版）
│   │   ├── __init__.py              # [9行]   字典版工具导出
│   │   ├── bash_tool.py             # [101行] Bash 工具 + 安全沙箱
│   │   ├── file_read_tool.py        # [58行]  文件读取 + 截断
│   │   ├── file_write_tool.py       # [61行]  文件写入 + 自动建目录
│   │   ├── bash.py                  # [51行]  BashTool 的 Pydantic 版
│   │   ├── file_read.py             # [69行]  FileReadTool 的 Pydantic 版
│   │   └── file_write.py            # [48行]  FileWriteTool 的 Pydantic 版
│   ├── agent/
│   │   ├── llm.py                   # [74行]  LLM 客户端封装
│   │   └── loop.py                  # [143行] 完善 REPL 循环
│   └── buddy/
│       └── companion.py             # [76行]  电子宠物彩蛋
└── tests/
    ├── test_agent.py                # [54行]  Agent 循环测试
    ├── test_bash_tool.py            # [27行]  安全沙箱测试
    └── test_file_tools.py           # [36行]  文件读写测试
```

## 附录 B：核心概念词汇表

| 术语 | 英文 | 含义 |
|------|------|------|
| Agent | Agent | 能自主决策、调用工具完成任务的 AI 程序 |
| ReAct 循环 | ReAct Loop | Reasoning + Acting，思考-行动交替进行 |
| Tool Use | Tool Use / Function Calling | 大模型调用外部工具的能力 |
| Provider | Provider | 大模型 API 的封装层 |
| 流式输出 | Streaming | 边生成边返回，不需要等待全部完成 |
| System Prompt | System Prompt | 定义 AI 行为规则的初始指令 |
| 上下文窗口 | Context Window | 大模型能处理的最大文本长度 |
| Microcompact | Microcompact | 截断过长的工具返回结果以节省上下文 |
| 安全沙箱 | Sandbox | 限制工具执行权限的安全机制 |
| REPL | Read-Eval-Print Loop | 交互式命令行循环 |

## 附录 C：推荐的进阶学习路径

学完 mini-cc Python 版后，建议按以下路径继续：

1. **对照 TypeScript 版**：用相同的方法读 `typescript/src/`，对比同一段逻辑在 TS 中怎么写
2. **读 docs/ 目录**：`docs/00-outline.md` 到 `docs/10-ultimate-agent-capabilities.md` 是对完整 Claude Code 的深度源码解析
3. **读 claw-code**：看 Rust 如何重写同一个 Agent，理解跨语言实现的异同
4. **动手扩展 mini-cc**：
   - 给 Python 版加一个 `GrepTool`（文件内容搜索）
   - 加一个 `WebSearchTool`（网络搜索）
   - 实现对话历史持久化（保存到文件）
   - 实现流式输出的思维链显示（支持 DeepSeek-R1）
   - 加一个 `/compact` 命令来压缩上下文

---

## 快速启动命令

```bash
# 1. 安装依赖
cd python
python -m venv venv
source venv/bin/activate   # Linux/Mac
# venv\Scripts\activate    # Windows
pip install -r requirements.txt

# 2. 配置 API Key
python main.py config set OPENAI_API_KEY sk-your-key-here

# 3. 启动
python main.py

# 4. 运行测试
pip install pytest pytest-asyncio
pytest tests/ -v
```
