# SoulCovenant - 轻量级 Claude CLI

> 直连 Anthropic API 的终端 AI 助手，无 Gateway、无 WebSocket，一个文件搞定一切。

---

## 目录

- [为什么做这个](#为什么做这个)
- [架构总览](#架构总览)
- [系统架构图](#系统架构图)
- [核心流程图](#核心流程图)
- [模块详解](#模块详解)
- [调用链分析](#调用链分析)
- [文件结构](#文件结构)
- [配置系统](#配置系统)
- [工具系统](#工具系统)
- [记忆系统](#记忆系统)
- [会话管理](#会话管理)
- [命令参考](#命令参考)
- [使用示例](#使用示例)

---

## 为什么做这个

openclaw 的 Gateway/WebSocket 架构太重，经常卡住不回复。SoulCovenant 的设计哲学：

```
openclaw:   用户 → Gateway → WebSocket → API → WebSocket → Gateway → 用户
                    ↑ 三层中间件，任何一层挂了都完蛋

SoulCovenant: 用户 → anthropic SDK → API → 流式输出
                    ↑ 零中间层，直连 API
```

- 单文件 Python 脚本，约 400 行
- 零中间层，anthropic SDK 直连 API
- 流式逐字输出，不卡顿
- JSONL 持久化，不丢消息

---

## 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      SoulCovenant CLI                           │
│                    /root/bin/soulcovenant                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Config   │  │  Memory  │  │  Tools   │  │   Session     │  │
│  │          │  │  (Soul)  │  │          │  │   (Scroll)    │  │
│  │ env vars │  │ soul/*.md│  │ bash     │  │ scrolls/*.jsonl│ │
│  │ json cfg │  │ genesis  │  │ read     │  │ append-only   │  │
│  │ CLI args │  │          │  │ write    │  │               │  │
│  │          │  │          │  │ edit     │  │               │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬───────┘  │
│       │              │             │                │           │
│  ┌────▼──────────────▼─────────────▼────────────────▼────────┐ │
│  │                    SoulCovenant Engine                     │ │
│  │                                                           │ │
│  │  client.messages.stream()  ←→  Tool Loop  ←→  Scroll I/O │ │
│  └──────────────────────┬────────────────────────────────────┘ │
│                         │                                      │
│  ┌──────────────────────▼────────────────────────────────────┐ │
│  │                     TUI Layer                             │ │
│  │                                                           │ │
│  │  Rich Live Markdown  │  Tool Feedback  │  User Prompt     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              │ HTTPS (streaming SSE)
                              ▼
                 ┌────────────────────────┐
                 │   Anthropic API        │
                 │   api.anthropic.com    │
                 │   (or custom base_url) │
                 └────────────────────────┘
```

---

## 核心流程图

### 启动流程

```
main()
 │
 ├─ argparse 解析 CLI 参数 (--model, --awaken, --scroll)
 │
 ├─ load_config()
 │   ├─ 读取默认值 (model=claude-opus-4-6-20260205, max_tokens=16384)
 │   ├─ 读取 ~/.soulcovenant/covenant.json (如果存在)
 │   └─ 环境变量覆盖 (ANTHROPIC_AUTH_TOKEN > ANTHROPIC_API_KEY, ANTHROPIC_BASE_URL)
 │
 ├─ 会话恢复
 │   ├─ --scroll <path> → 加载指定 JSONL 文件
 │   ├─ --awaken       → 找到 scrolls/ 下最新的 JSONL 文件
 │   └─ (default)      → 创建新 Scroll，文件名为当前时间戳
 │
 ├─ SoulCovenant.__init__()
 │   ├─ build_client(cfg)     → 创建 anthropic.Anthropic 实例
 │   ├─ build_system_prompt() → 拼接 genesis.md + soul/*.md
 │   └─ 恢复历史消息到 self.messages
 │
 ├─ print_banner()  → 打印 ASCII art + model 信息
 │
 ├─ signal.signal(SIGINT, handler)  → Ctrl+C 优雅中断
 │
 └─ 进入主循环 (while True)
     │
     ├─ console.input("you> ")  → 等待用户输入
     │
     ├─ 以 "/" 开头？ → 解析并执行命令
     │   ├─ /ascend, /quit   → break 退出
     │   ├─ /rebirth, /new   → 新建 Scroll，清空 messages
     │   ├─ /model [name]    → 切换模型
     │   ├─ /scrolls         → 列出历史会话
     │   ├─ /soul            → 列出记忆文件
     │   └─ /help            → 显示帮助
     │
     └─ 普通消息 → append 到 messages + scroll → stream_response()
```

### 流式对话 + 工具循环（核心）

```
stream_response()
 │
 └─ while True:  ◄──────────────────────────────────────────┐
     │                                                       │
     ├─ client.messages.stream(                              │
     │      model, system, messages, tools                   │
     │  )                                                    │
     │                                                       │
     ├─ Rich Live 开始实时渲染                                │
     │                                                       │
     ├─ for event in stream:                                 │
     │   │                                                   │
     │   ├─ content_block_start (tool_use)                   │
     │   │   └─ 记录 tool_use {id, name}                     │
     │   │                                                   │
     │   ├─ content_block_delta                              │
     │   │   ├─ text_delta → 追加文本，更新 Live 渲染         │
     │   │   └─ input_json_delta → 累积工具参数 JSON          │
     │   │                                                   │
     │   ├─ message_start → 记录 input_tokens                │
     │   │                                                   │
     │   └─ message_delta → 记录 stop_reason, output_tokens  │
     │                                                       │
     ├─ Rich Live 停止                                       │
     │                                                       │
     ├─ 解析工具参数 JSON                                     │
     │                                                       │
     ├─ 构建 assistant message → append 到 messages + scroll  │
     │                                                       │
     ├─ 显示 token 用量                                       │
     │                                                       │
     ├─ stop_reason == "tool_use"?                           │
     │   │                                                   │
     │   ├─ NO → return (对话结束)                            │
     │   │                                                   │
     │   └─ YES → 执行工具:                                   │
     │       │                                               │
     │       ├─ for each tool_call:                          │
     │       │   ├─ 终端显示: ⚒ bash: ls -la                 │
     │       │   ├─ execute_tool(name, input)                │
     │       │   ├─ 终端显示: ✓/✗ + 简要结果                  │
     │       │   └─ 收集 tool_result                         │
     │       │                                               │
     │       ├─ tool_results → append 到 messages + scroll   │
     │       │                                               │
     │       └─ continue ─────────────────────────────────────┘
```

### 工具执行流程

```
execute_tool(name, input_data)
 │
 ├─ "bash"
 │   └─ subprocess.run(command, shell=True, timeout=120)
 │       ├─ 合并 stdout + stderr
 │       ├─ 空输出 → "(no output)"
 │       ├─ 非零退出码 → 追加 "[exit code: N]"
 │       └─ 超时 → "Command timed out after 120 seconds"
 │
 ├─ "read_file"
 │   └─ Path(path).expanduser().read_text()
 │       ├─ 文件不存在 → "File not found"
 │       └─ 成功 → 返回文件全部内容
 │
 ├─ "write_file"
 │   └─ Path(path).expanduser().write_text(content)
 │       ├─ 自动创建父目录 (mkdir -p)
 │       └─ 成功 → "Wrote N bytes to path"
 │
 └─ "edit_file"
     └─ 读取文件 → 查找 old_string
         ├─ 0 次匹配 → "old_string not found"
         ├─ >1 次匹配 → "must be unique, provide more context"
         └─ 1 次匹配 → replace + write → "Edited path"
```

---

## 模块详解

### 1. Config 模块 (L33-67)

配置加载优先级（后者覆盖前者）：

```
  优先级（低 → 高）
  ┌─────────────────────────┐
  │ 硬编码默认值             │  model=claude-opus-4-6-20260205
  │                         │  max_tokens=16384
  ├─────────────────────────┤
  │ covenant.json            │  ~/.soulcovenant/covenant.json
  │                         │  任意 key 都会覆盖默认值
  ├─────────────────────────┤
  │ 环境变量                 │  ANTHROPIC_AUTH_TOKEN (优先)
  │                         │  ANTHROPIC_API_KEY
  │                         │  ANTHROPIC_BASE_URL
  ├─────────────────────────┤
  │ CLI 参数                 │  --model 覆盖 model
  └─────────────────────────┘
```

**covenant.json 示例：**

```json
{
  "api_key": "sk-ant-...",
  "base_url": "https://your-proxy.example.com",
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 8192
}
```

### 2. Memory 模块 (L70-110)

System prompt 拼接流程：

```
build_system_prompt()
 │
 ├─ genesis.md 存在？
 │   └─ YES → 读取内容作为第一段
 │
 ├─ soul/ 目录有 .md 文件？
 │   └─ YES → 按文件名排序，逐个读取
 │       └─ 格式: "## {filename}\n\n{content}"
 │       └─ 用 "---" 分隔多个文件
 │
 ├─ 都为空？
 │   └─ 使用内置默认 prompt
 │
 └─ 输出: genesis + "---" + "# Soul Memories\n" + 所有记忆
```

**最终 system prompt 结构：**

```
{genesis.md 内容}

---

# Soul Memories

## preferences
{soul/preferences.md 内容}

---

## coding_style
{soul/coding_style.md 内容}
```

### 3. Tools 模块 (L113-267)

四个核心工具的 API Schema 和执行实现：

| 工具 | 参数 | 返回 | 超时 |
|------|------|------|------|
| `bash` | `command: str` | stdout+stderr, exit code | 120s |
| `read_file` | `path: str` | 文件内容 | - |
| `write_file` | `path: str, content: str` | 写入字节数 | - |
| `edit_file` | `path: str, old_string: str, new_string: str` | 替换结果 | - |

**工具安全机制：**
- `edit_file` 要求 `old_string` 唯一匹配（避免误改）
- `bash` 有 120 秒超时保护
- 工具结果超过 50,000 字符自动截断
- 所有工具错误通过 `is_error: true` 回传给模型

### 4. Session 模块 (L270-307)

**Scroll 类** — JSONL 持久化：

```
Scroll
 │
 ├─ __init__(path=None)
 │   ├─ path 指定 → 加载该文件
 │   └─ path=None → 创建新文件: scrolls/{YYYYMMDD_HHMMSS}.jsonl
 │
 ├─ _load()
 │   └─ 逐行读取 JSONL，每行 json.loads() → self.messages[]
 │
 ├─ append(message)
 │   ├─ self.messages.append(message)
 │   └─ 追加写入文件 (mode="a")  ← 不会丢失已有消息
 │
 └─ latest() [静态方法]
     └─ 找到 scrolls/ 下按名称排序最新的 .jsonl 文件
```

**JSONL 文件格式（每行一个 JSON 对象）：**

```jsonl
{"role": "user", "content": "帮我写个 hello world"}
{"role": "assistant", "content": [{"type": "text", "text": "好的..."}, {"type": "tool_use", "id": "toolu_xxx", "name": "write_file", "input": {"path": "hello.py", "content": "print('hello')"}}]}
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_xxx", "content": "Wrote 15 bytes to hello.py", "is_error": false}]}
{"role": "assistant", "content": [{"type": "text", "text": "已经创建了 hello.py 文件。"}]}
```

### 5. Chat Engine (L310-491)

**SoulCovenant 类** — 核心引擎：

```
SoulCovenant
 │
 ├─ 属性
 │   ├─ client     → anthropic.Anthropic 实例
 │   ├─ model      → 当前模型名称
 │   ├─ messages    → 当前会话消息列表 (in-memory)
 │   ├─ scroll     → Scroll 实例 (持久化)
 │   ├─ system_prompt → 拼接好的 system prompt
 │   └─ _interrupted  → Ctrl+C 中断标志
 │
 └─ 方法
     ├─ set_model(model)      → 运行时切换模型
     └─ stream_response()     → 流式 API 调用 + 工具循环
```

### 6. TUI 层 (L494-626)

终端交互层，负责：

- ASCII banner 显示
- 用户输入循环 (`console.input`)
- 斜杠命令解析和执行
- 将普通消息传递给 Chat Engine
- SIGINT 信号处理

---

## 调用链分析

### 调用链 1: 普通文本对话

```
用户输入 "你好"
    │
    ▼
main() while loop
    │
    ├─ user_msg = {"role": "user", "content": "你好"}
    ├─ cov.messages.append(user_msg)
    ├─ cov.scroll.append(user_msg)           → 写入 JSONL 文件
    │
    └─ cov.stream_response()
        │
        ├─ client.messages.stream(
        │      model="claude-opus-4-6-20260205",
        │      system="You are a powerful AI...",
        │      messages=[{"role":"user","content":"你好"}],
        │      tools=[bash, read_file, write_file, edit_file]
        │  )
        │
        ├─ Live(Markdown("")) 开始
        │
        ├─ event loop:
        │   ├─ text_delta: "你" → Live 更新
        │   ├─ text_delta: "好" → Live 更新
        │   ├─ text_delta: "！" → Live 更新
        │   └─ message_delta: stop_reason="end_turn"
        │
        ├─ Live 停止
        │
        ├─ assistant_msg = {"role":"assistant","content":[{"type":"text","text":"你好！"}]}
        ├─ messages.append(assistant_msg)
        ├─ scroll.append(assistant_msg)      → 写入 JSONL 文件
        │
        ├─ 显示: tokens: 42 in / 8 out
        │
        └─ stop_reason != "tool_use" → return
```

### 调用链 2: 带工具调用的对话

```
用户输入 "列出 /root 目录"
    │
    ▼
main() while loop
    │
    ├─ messages.append + scroll.append (user msg)
    │
    └─ cov.stream_response()
        │
        ├─ ┌─ API 请求 #1 ────────────────────────────────────┐
        │   │ client.messages.stream(messages=[user_msg])      │
        │   │                                                   │
        │   │ 流式事件:                                         │
        │   │   text_delta: "好的，让我..."   → Live 渲染       │
        │   │   content_block_start: tool_use (bash)            │
        │   │   input_json_delta: {"command":"ls -la /root"}    │
        │   │   message_delta: stop_reason="tool_use"           │
        │   └───────────────────────────────────────────────────┘
        │
        ├─ assistant_msg = {text + tool_use} → messages + scroll
        │
        ├─ 终端显示: ⚒ bash: ls -la /root
        │
        ├─ execute_tool("bash", {"command": "ls -la /root"})
        │   └─ subprocess.run("ls -la /root")
        │       └─ (True, "total 48\ndrwx...\n...")
        │
        ├─ 终端显示:
        │   ✓ total 48
        │   ✓ drwx------  1 root root ...
        │   ✓ -rwxr-xr-x  1 root root ...
        │
        ├─ tool_result_msg → messages + scroll
        │
        ├─ continue (回到 while True 顶部)
        │
        ├─ ┌─ API 请求 #2 ────────────────────────────────────┐
        │   │ client.messages.stream(                           │
        │   │   messages=[user, assistant, tool_result]         │
        │   │ )                                                 │
        │   │                                                   │
        │   │ 流式事件:                                         │
        │   │   text_delta: "/root 目录包含..."  → Live 渲染    │
        │   │   message_delta: stop_reason="end_turn"           │
        │   └───────────────────────────────────────────────────┘
        │
        ├─ assistant_msg → messages + scroll
        │
        └─ stop_reason != "tool_use" → return
```

### 调用链 3: 多步工具链

```
用户: "创建 hello.py 并运行它"
    │
    ▼
stream_response() while True:
    │
    ├── API #1 → 模型返回 tool_use: write_file
    │   execute_tool("write_file", {path:"hello.py", content:"print('hello')"})
    │   → ✓ Wrote 15 bytes
    │   → tool_result 回传 → continue
    │
    ├── API #2 → 模型返回 tool_use: bash
    │   execute_tool("bash", {command:"python3 hello.py"})
    │   → ✓ hello
    │   → tool_result 回传 → continue
    │
    └── API #3 → 模型返回 text: "已创建并运行..."
        → stop_reason="end_turn" → return

整个过程: 3 次 API 调用，2 次工具执行，用户只输入了一句话
```

### 调用链 4: 会话恢复 (--awaken)

```
soulcovenant --awaken
    │
    ▼
main()
    │
    ├─ Scroll.latest()
    │   └─ sorted(scrolls/*.jsonl)[-1]
    │       └─ "20260303_143025.jsonl"
    │
    ├─ Scroll("20260303_143025.jsonl")
    │   └─ _load()
    │       └─ 逐行读 JSONL → self.messages = [msg1, msg2, ...]
    │
    ├─ SoulCovenant(cfg, scroll)
    │   └─ self.messages = list(scroll.messages)  ← 恢复历史
    │
    └─ 用户输入新消息
        └─ stream_response()
            └─ client.messages.stream(
                   messages=[...历史消息..., 新消息]
               )
            → 模型能看到所有历史上下文
```

---

## 文件结构

```
/root/bin/soulcovenant              ← 主程序（单文件，~400 行）
 │
 └─ 运行时创建/读取:

~/.soulcovenant/                     ← 工作根目录
 │
 ├── covenant.json                   ← 契约 — API 配置
 │   {
 │     "api_key": "sk-ant-...",       可选，也可用环境变量
 │     "base_url": "https://...",     可选，默认官方 API
 │     "model": "claude-opus-4-6-20260205",
 │     "max_tokens": 16384
 │   }
 │
 ├── genesis.md                      ← 创世 — 自定义 system prompt
 │   你是一个专注于 Python 开发的助手...  （可选文件）
 │
 ├── soul/                           ← 灵魂 — 记忆目录
 │   ├── preferences.md               AI 和用户都可读写
 │   ├── project_notes.md             启动时自动加载到 system prompt
 │   └── coding_style.md              按文件名排序拼接
 │
 ├── scrolls/                        ← 卷轴 — 会话历史
 │   ├── 20260303_100000.jsonl        每次新会话一个文件
 │   ├── 20260303_143025.jsonl        JSONL 格式，每行一条消息
 │   └── 20260303_220000.jsonl        追加写入，不会丢失
 │
 └── relics/                         ← 遗物 — 临时文件（预留）
```

---

## 配置系统

### 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `ANTHROPIC_AUTH_TOKEN` | API 密钥（最高优先级） | `sk-ant-api03-...` |
| `ANTHROPIC_API_KEY` | API 密钥（标准） | `sk-ant-api03-...` |
| `ANTHROPIC_BASE_URL` | 自定义 API 地址 | `https://proxy.example.com` |

### CLI 参数

```bash
soulcovenant                       # 新会话，默认模型
soulcovenant --model claude-sonnet-4-20250514  # 指定模型
soulcovenant --awaken              # 恢复最近会话
soulcovenant --scroll path/to.jsonl # 恢复指定会话
```

---

## 工具系统

### 工具定义 → API → 执行 → 回传 完整链路

```
┌─ TOOL_DEFINITIONS ──────────────────────────────────┐
│                                                      │
│  [                                                   │
│    {name: "bash",      input_schema: {command}},     │  ← 发送给 API
│    {name: "read_file", input_schema: {path}},        │    模型知道有哪些工具
│    {name: "write_file", input_schema: {path,content}},│
│    {name: "edit_file", input_schema: {path,old,new}},│
│  ]                                                   │
│                                                      │
└──────────────────────────┬───────────────────────────┘
                           │
                           ▼
┌─ API Response ───────────────────────────────────────┐
│                                                      │
│  stop_reason: "tool_use"                             │
│  content: [                                          │
│    {type: "text", text: "让我看看..."},               │
│    {type: "tool_use", id: "toolu_xxx",               │  ← 模型决定调用工具
│     name: "bash", input: {command: "ls"}}            │
│  ]                                                   │
│                                                      │
└──────────────────────────┬───────────────────────────┘
                           │
                           ▼
┌─ execute_tool() ─────────────────────────────────────┐
│                                                      │
│  name="bash" → _tool_bash("ls")                      │
│    → subprocess.run("ls", shell=True, timeout=120)   │  ← 本地执行
│    → (True, "file1.py\nfile2.py\n")                  │
│                                                      │
└──────────────────────────┬───────────────────────────┘
                           │
                           ▼
┌─ Tool Result Message ────────────────────────────────┐
│                                                      │
│  {role: "user", content: [                           │
│    {type: "tool_result",                             │
│     tool_use_id: "toolu_xxx",                        │  ← 回传给 API
│     content: "file1.py\nfile2.py\n",                 │    模型继续处理
│     is_error: false}                                 │
│  ]}                                                  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 各工具详细行为

#### bash

```
输入: {command: "git status"}
  │
  ├─ subprocess.run(command, shell=True, capture_output=True, timeout=120)
  │
  ├─ 正常执行:
  │   stdout: "On branch main\n..."
  │   stderr: ""
  │   returncode: 0
  │   → (True, "On branch main\n...")
  │
  ├─ 命令失败:
  │   stdout: ""
  │   stderr: "fatal: not a git repository"
  │   returncode: 128
  │   → (False, "fatal: not a git repository\n[exit code: 128]")
  │
  └─ 超时:
      → (False, "Command timed out after 120 seconds")
```

#### read_file

```
输入: {path: "~/project/main.py"}
  │
  ├─ Path("~/project/main.py").expanduser()
  │   → /root/project/main.py
  │
  ├─ 文件存在 → (True, "import sys\n...")
  └─ 文件不存在 → (False, "File not found: ~/project/main.py")
```

#### write_file

```
输入: {path: "/tmp/test/hello.py", content: "print('hi')"}
  │
  ├─ Path("/tmp/test").mkdir(parents=True, exist_ok=True)  ← 自动建目录
  ├─ Path("/tmp/test/hello.py").write_text("print('hi')")
  └─ (True, "Wrote 11 bytes to /tmp/test/hello.py")
```

#### edit_file

```
输入: {path: "main.py", old_string: "def foo():", new_string: "def bar():"}
  │
  ├─ 读取文件内容
  ├─ 搜索 "def foo():" 出现次数
  │   ├─ 0 次 → (False, "old_string not found in file")
  │   ├─ >1 次 → (False, "old_string found N times — must be unique...")
  │   └─ 1 次 → 替换 → 写回 → (True, "Edited main.py (replaced 1 occurrence)")
  │
  └─ 安全保证: 只有唯一匹配时才执行替换
```

---

## 记忆系统

AI 可以通过 `write_file` / `edit_file` 工具读写 `~/.soulcovenant/soul/` 目录下的文件，实现跨会话记忆。

```
会话 1:
  用户: "我喜欢用 4 空格缩进"
  AI: write_file("~/.soulcovenant/soul/preferences.md", "- 缩进: 4 空格")

                    ↓ 记忆持久化

会话 2 (新会话):
  启动 → build_system_prompt()
       → 读取 soul/preferences.md
       → system prompt 包含 "- 缩进: 4 空格"
  AI 自动使用 4 空格缩进，无需重复说明
```

---

## 命令参考

### 交互命令（会话内）

| 命令 | 别名 | 说明 |
|------|------|------|
| `/help` | - | 显示帮助信息 |
| `/rebirth` | `/new` | 新建会话（清空消息，新建 Scroll 文件） |
| `/ascend` | `/quit`, `/exit` | 退出程序 |
| `/model` | - | 显示当前模型 |
| `/model <name>` | - | 切换模型（如 `/model claude-sonnet-4-20250514`） |
| `/scrolls` | - | 列出最近 10 个会话文件 |
| `/soul` | - | 列出所有灵魂记忆文件 |

### 快捷键

| 按键 | 说明 |
|------|------|
| `Ctrl+C` | 中断当前 API 请求（不退出程序） |
| `Ctrl+C` x2 / `Ctrl+D` | 退出程序 |

---

## 使用示例

### 基本使用

```bash
$ soulcovenant

  ╔═══════════════════════════════════════╗
  ║         S O U L C O V E N A N T      ║
  ╚═══════════════════════════════════════╝
  model: claude-opus-4-6-20260205
  type /help for commands

you> 帮我写一个 Python 快排
```

### 恢复会话

```bash
$ soulcovenant --awaken
Awakened latest scroll: 20260303_143025.jsonl (12 messages)
```

### 自定义 System Prompt

```bash
$ cat > ~/.soulcovenant/genesis.md << 'EOF'
你是 SoulCovenant，一个专注于系统编程的 AI 助手。
- 优先使用 Rust 和 C
- 代码注释用中文
- 回答简洁直接
EOF
```

### 使用代理/自定义端点

```bash
export ANTHROPIC_BASE_URL="https://your-proxy.example.com"
export ANTHROPIC_AUTH_TOKEN="sk-ant-..."
soulcovenant
```

---

## 数据流全景图

```
┌──────────┐     ┌──────────────────────────────────────────────────┐
│          │     │              SoulCovenant Process                │
│  用户    │     │                                                  │
│          │     │  ┌─────────┐    ┌────────────┐   ┌──────────┐  │
│  键盘 ───┼────►│  │  TUI    │───►│   Engine   │──►│  Tools   │  │
│          │     │  │         │    │            │   │          │  │
│  屏幕 ◄──┼─────│  │ Rich    │◄───│ messages[] │◄──│ execute  │  │
│          │     │  │ Markdown│    │            │   │ _tool_*  │  │
│          │     │  └─────────┘    └─────┬──────┘   └────┬─────┘  │
│          │     │                       │               │         │
│          │     │              ┌────────▼──────┐   ┌────▼─────┐  │
│          │     │              │  Scroll       │   │ 本地文件  │  │
│          │     │              │  (JSONL 持久化)│   │ 系统     │  │
│          │     │              └───────────────┘   └──────────┘  │
│          │     │                       │                         │
│          │     └───────────────────────┼─────────────────────────┘
│          │                             │
└──────────┘          ┌──────────────────┼──────────────────┐
                      │                  │                  │
                      ▼                  ▼                  ▼
               ~/.soulcovenant/   ~/.soulcovenant/   ~/.soulcovenant/
               scrolls/*.jsonl    soul/*.md          genesis.md
               (会话持久化)        (记忆持久化)       (人格配置)
                                         │
                      ┌──────────────────┘
                      │
                      ▼
            ┌───────────────────┐        ┌────────────────────┐
            │ build_system_     │        │                    │
            │ prompt()          │───────►│  Anthropic API     │
            │                   │        │  (streaming SSE)   │
            │ genesis + soul    │        │                    │
            └───────────────────┘        └────────────────────┘
```

---

## 与 openclaw 对比

| 特性 | openclaw | SoulCovenant |
|------|----------|-------------|
| 架构 | Gateway + WebSocket + API | SDK 直连 API |
| 中间层 | 3 层 | 0 层 |
| 代码量 | 多文件，数千行 | 单文件，~400 行 |
| 依赖 | Gateway 服务 + WebSocket | anthropic SDK + rich |
| 稳定性 | 经常卡住 | 直连，无中间故障点 |
| 部署 | 需启动多个服务 | 一个文件，chmod +x 即用 |
| 会话恢复 | 依赖 Gateway 状态 | JSONL 文件，可靠持久 |
| 流式输出 | WebSocket 转发 | SSE 直出 |
