# 百聆 (Bailing) 项目架构与二次开发指南

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 目录结构](#2-目录结构)
- [3. 系统架构](#3-系统架构)
  - [3.1 整体架构](#31-整体架构)
  - [3.2 核心流程](#32-核心流程)
  - [3.3 模块依赖关系](#33-模块依赖关系)
- [4. 核心模块详解](#4-核心模块详解)
  - [4.1 Robot (机器人核心)](#41-robot-机器人核心)
  - [4.2 Recorder (录音模块)](#42-recorder-录音模块)
  - [4.3 VAD (语音活动检测)](#43-vad-语音活动检测)
  - [4.4 ASR (语音识别)](#44-asr-语音识别)
  - [4.5 LLM (大语言模型)](#45-llm-大语言模型)
  - [4.6 TTS (语音合成)](#46-tts-语音合成)
  - [4.7 Player (播放器)](#47-player-播放器)
  - [4.8 Dialogue (对话管理)](#48-dialogue-对话管理)
  - [4.9 Memory (记忆模块)](#49-memory-记忆模块)
  - [4.10 RAG (检索增强生成)](#410-rag-检索增强生成)
- [5. 插件系统详解](#5-插件系统详解)
  - [5.1 插件架构](#51-插件架构)
  - [5.2 注册机制](#52-注册机制)
  - [5.3 工具类型与动作类型](#53-工具类型与动作类型)
  - [5.4 TaskManager (任务管理器)](#54-taskmanager-任务管理器)
  - [5.5 内置插件一览](#55-内置插件一览)
- [6. 服务层详解](#6-服务层详解)
  - [6.1 WebSocket 服务 (server.py)](#61-websocket-服务-serverpy)
  - [6.2 Flask 服务 (server/server.py)](#62-flask-服务-serverserverpy)
  - [6.3 前端页面 (static/index.html)](#63-前端页面-staticindexhtml)
- [7. 配置系统](#7-配置系统)
- [8. 第三方依赖](#8-第三方依赖)
- [9. 二次开发指南](#9-二次开发指南)
  - [9.1 添加新的 ASR 引擎](#91-添加新的-asr-引擎)
  - [9.2 添加新的 LLM 后端](#92-添加新的-llm-后端)
  - [9.3 添加新的 TTS 引擎](#93-添加新的-tts-引擎)
  - [9.4 添加新的 VAD 引擎](#94-添加新的-vad-引擎)
  - [9.5 添加新的 Recorder](#95-添加新的-recorder)
  - [9.6 添加新的 Player](#96-添加新的-player)
  - [9.7 添加新的插件/工具函数](#97-添加新的插件工具函数)
  - [9.8 自定义 Prompt](#98-自定义-prompt)
  - [9.9 自定义打断策略](#99-自定义打断策略)
- [10. 常见问题与排错](#10-常见问题与排错)

---

## 1. 项目概述

百聆 (Bailing) 是一个开源的语音对话助手，通过 **ASR + LLM + TTS** 的管线架构实现类 GPT-4o 的语音对话体验。项目设计目标：

- **低延迟**：端到端时延约 800ms
- **轻量化**：无需 GPU，可部署在边缘设备
- **模块化**：每个组件（ASR、VAD、LLM、TTS、Player、Recorder）均可独立替换
- **可扩展**：通过插件系统支持工具调用、任务管理、记忆等高级功能

**技术栈**：Python 3.12+，FastAPI (WebSocket)，Flask (可选展示服务)，PyTorch，OpenAI SDK

**开源协议**：MIT License

---

## 2. 目录结构

```
bailing/
├── main.py                     # 本地运行入口（命令行方式）
├── server.py                   # WebSocket 服务运行入口（推荐）
├── requirements.txt            # Python 依赖
├── LICENSE                     # MIT 开源协议
├── README.md                   # 项目说明（中文）
├── README_en.md                # 项目说明（英文）
│
├── bailing/                    # 核心模块目录
│   ├── robot.py                # Robot 核心编排器 —— 全局调度中心
│   ├── recorder.py             # 录音模块（麦克风 / WebSocket）
│   ├── vad.py                  # 语音活动检测（Silero VAD）
│   ├── asr.py                  # 语音识别（FunASR / SenseVoice）
│   ├── llm.py                  # 大语言模型（OpenAI 兼容接口 / Ollama）
│   ├── tts.py                  # 语音合成（EdgeTTS / Kokoro / ChatTTS / macOS say / gTTS）
│   ├── player.py               # 音频播放器（Pygame / PyAudio / sounddevice / WebSocket）
│   ├── dialogue.py             # 对话历史管理
│   ├── memory.py               # 长期记忆（对话摘要 + 持久化）
│   ├── rag.py                  # 检索增强生成（已注释，基于 LangChain）
│   ├── prompt.py               # 系统 Prompt 定义
│   └── utils.py                # 工具函数（配置读取、分句、打断检测等）
│
├── plugins/                    # 插件系统目录
│   ├── registry.py             # 插件注册中心（装饰器 + 枚举定义）
│   ├── task_manager.py         # 任务管理器（调度插件执行）
│   ├── function_manager.py     # 函数管理器（早期版本，已被 task_manager 替代）
│   ├── function_calls_config.json  # LLM Function Calling 的 JSON Schema 配置
│   └── functions/              # 具体插件实现
│       ├── get_weather.py          # 天气查询
│       ├── get_day_of_week.py      # 日期/星期查询
│       ├── schedule_task.py        # 定时任务
│       ├── web_search.py           # 网页搜索
│       ├── open_application.py     # 打开 macOS 应用
│       ├── ielts_speaking_practice.py  # 雅思口语练习
│       ├── aigc_manus.py           # 通用 AIGC（集成 OpenManus Agent）
│       └── search_local_documents.py   # 本地文档搜索（已注释）
│
├── config/                     # 配置目录
│   └── config.yaml             # 主配置文件
│
├── static/                     # 前端静态文件（WebSocket 模式）
│   └── index.html              # 前端语音对话页面
│
├── server/                     # Flask 展示服务（本地模式可选）
│   ├── server.py               # Flask + SocketIO 服务
│   └── templates/
│       └── index.html          # Flask 模板页面
│
├── third_party/                # 第三方项目
│   └── OpenManus/              # OpenManus Agent 框架（用于 AIGC 插件）
│
├── models/                     # 模型文件存放目录（需手动下载）
│   └── (SenseVoiceSmall等)
│
├── documents/                  # RAG 文档目录
│   └── README.md
│
├── assets/                     # 静态资源（Logo、流程图等）
└── tmp/                        # 临时文件目录（音频文件、日志、对话历史等）
```

---

## 3. 系统架构

### 3.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        用户层 (User Layer)                        │
│   ┌──────────────┐     ┌───────────────────────────────────┐     │
│   │ 本地麦克风    │     │ 浏览器 WebSocket 客户端            │     │
│   │ (PyAudio)    │     │ (static/index.html)              │     │
│   └──────┬───────┘     └────────────────┬──────────────────┘     │
│          │ 音频流                        │ PCM 二进制流             │
└──────────┼──────────────────────────────┼────────────────────────┘
           │                              │
┌──────────┼──────────────────────────────┼────────────────────────┐
│          ▼                              ▼                        │
│   ┌──────────────┐              ┌──────────────────┐            │
│   │ Recorder     │              │ WebSocket Server │            │
│   │ (PyAudio)    │              │ (FastAPI)        │            │
│   └──────┬───────┘              └────────┬─────────┘            │
│          │                               │                      │
│          └───────────┬───────────────────┘                      │
│                      ▼                                          │
│             ┌────────────────┐                                  │
│             │  audio_queue   │  (原始 PCM 音频队列)              │
│             └───────┬────────┘                                  │
│                     ▼                                           │
│             ┌────────────────┐                                  │
│             │   VAD 模块     │  语音活动检测 (Silero VAD)        │
│             │                │  输出: {voice, vad_status}       │
│             └───────┬────────┘                                  │
│                     ▼                                           │
│             ┌────────────────┐                                  │
│             │  vad_queue     │  (VAD 处理后的队列)               │
│             └───────┬────────┘                                  │
│                     ▼                                           │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              Robot._duplex() 双工处理循环              │       │
│  │                                                      │       │
│  │  vad_start ──► 收集语音帧到 self.speech              │       │
│  │  vad_end   ──► ASR识别 ──► LLM对话 ──► TTS合成       │       │
│  │  播放中+说话 ──► 打断逻辑                             │       │
│  │  空闲      ──► 取后台任务播放                         │       │
│  └──────┬───────────────────────────────────┬───────────┘       │
│         │                                   │                   │
│         ▼                                   ▼                   │
│  ┌──────────────┐                   ┌──────────────┐            │
│  │  ASR 模块    │                   │  LLM 模块    │            │
│  │ (FunASR)    │                   │ (OpenAI/     │            │
│  │              │                   │  Ollama)     │            │
│  └──────┬───────┘                   └──────┬───────┘            │
│         │ text                              │ stream chunks     │
│         │                                   ▼                   │
│         │                           ┌──────────────┐            │
│         │                           │ 分句逻辑     │            │
│         │                           │ is_segment   │            │
│         │                           └──────┬───────┘            │
│         │                                  │ 句子片段            │
│         │                                  ▼                    │
│         │                           ┌──────────────┐            │
│         │                           │  TTS 模块    │            │
│         │                           │ (Kokoro/     │            │
│         │                           │  EdgeTTS/..) │            │
│         │                           └──────┬───────┘            │
│         │                                  │ 音频文件            │
│         │                                  ▼                    │
│         │                           ┌──────────────┐            │
│         │                           │  tts_queue   │            │
│         │                           │ (顺序播放队列)│            │
│         │                           └──────┬───────┘            │
│         │                                  ▼                    │
│         │                           ┌──────────────┐            │
│         │                           │ Player 模块  │            │
│         │                           │ (Pygame/     │            │
│         │                           │  WebSocket)  │            │
│         │                           └──────────────┘            │
│         │                                                       │
│  ┌──────┴────────────────────────────────────────────────┐      │
│  │                   Plugin System                        │      │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │      │
│  │  │Registry │  │TaskManager│  │ functions/         │   │      │
│  │  │(装饰器注册)│  │(调度执行) │  │ (具体工具函数实现) │   │      │
│  │  └──────────┘  └──────────┘  └───────────────────┘   │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                  │
│  ┌────────────────────────┐  ┌──────────────────────────┐       │
│  │    Memory 模块         │  │    Dialogue 模块          │       │
│  │  (对话摘要 + 持久化)   │  │  (对话历史管理 + 序列化)  │       │
│  └────────────────────────┘  └──────────────────────────┘       │
│                        服务层 (Service Layer)                     │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 核心流程

百聆的核心工作流程如下：

1. **音频采集**：`Recorder` 持续从麦克风或 WebSocket 接收音频流，放入 `audio_queue`
2. **VAD 检测**：独立线程从 `audio_queue` 取数据，经 `VAD` 判断是否有人说话，结果放入 `vad_queue`
3. **双工处理**（`Robot._duplex()`）：
   - 检测到 VAD `start` → 开始收集语音帧
   - 检测到 VAD `end` → 将收集的语音帧送 `ASR` 识别为文本
   - 识别结果 → 送入 `LLM` 生成流式回复
   - LLM 流式输出 → **分句逻辑**切分为句子片段
   - 每个句子片段 → 异步提交给 `TTS` 转为音频文件
   - TTS 音频文件 → 放入 `tts_queue` 保证顺序播放
   - `Player` 按顺序播放音频
4. **打断处理**：如果用户在播放过程中说话（VAD 检测到 `start`），可触发打断，清空播放队列
5. **工具调用**：如果 LLM 返回 function call，交给 `TaskManager` 执行相应插件

### 3.3 模块依赖关系

```
Robot（核心编排器）
  ├── Recorder (音频输入)
  ├── VAD (语音活动检测)
  ├── ASR (语音识别)
  ├── LLM (大语言模型)
  ├── TTS (语音合成)
  ├── Player (音频播放)
  ├── Dialogue (对话管理)
  ├── Memory (长期记忆)
  ├── TaskManager (插件/工具管理)
  └── utils (工具函数)
```

每个模块通过**工厂模式** (`create_instance`) 实例化，由 `config.yaml` 中的 `selected_module` 配置决定使用哪个具体实现类。

---

## 4. 核心模块详解

### 4.1 Robot (机器人核心)

**文件**：`bailing/robot.py`

Robot 是整个系统的核心编排器，负责：

- 读取配置，初始化所有子模块
- 管理音频采集、VAD、TTS 优先级等多个后台线程
- 实现 `_duplex()` 双工处理循环
- 协调 ASR → LLM → TTS → Player 的管线流程
- 处理打断逻辑
- 管理工具调用（通过 `chat_tool` 方法）

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `audio_queue` | `queue.Queue` | 原始音频数据队列 |
| `vad_queue` | `queue.Queue` | VAD 处理后的数据队列 |
| `tts_queue` | `queue.Queue` | TTS 任务的 Future 队列（保证播放顺序） |
| `dialogue` | `Dialogue` | 对话历史管理 |
| `memory` | `Memory` | 长期记忆 |
| `task_manager` | `TaskManager` | 插件任务管理器 |
| `chat_lock` | `bool` | 对话锁，防止并发对话冲突 |
| `stop_event` | `threading.Event` | 退出信号 |
| `INTERRUPT` | `bool` | 是否允许打断 |
| `executor` | `ThreadPoolExecutor` | 线程池（最大10线程） |

**关键方法**：

| 方法 | 说明 |
|------|------|
| `run()` | 主循环，启动录音 + VAD + TTS 线程，循环调用 `_duplex()` |
| `_duplex()` | 双工处理：从 vad_queue 取数据，处理 VAD 事件 |
| `chat(query)` | 非工具模式的对话处理：LLM 流式回复 → 分句 → TTS |
| `chat_tool(query)` | 工具模式的对话处理：支持 function calling |
| `speak_and_play(text)` | 将文本转为 TTS 音频文件 |
| `interrupt_playback()` | 打断当前播放 |
| `start_recording_and_vad()` | 启动录音 + VAD + TTS 优先级线程 |
| `shutdown()` | 安全关闭所有资源 |

### 4.2 Recorder (录音模块)

**文件**：`bailing/recorder.py`

**抽象基类**：`AbstractRecorder`

| 方法 | 说明 |
|------|------|
| `start_recording(audio_queue)` | 开始录音，将音频数据放入队列 |
| `stop_recording()` | 停止录音 |

**实现类**：

| 类名 | 说明 | 适用场景 |
|------|------|---------|
| `RecorderPyAudio` | 通过 PyAudio 从本地麦克风采集 | 本地运行 |
| `WebSocketRecorder` | 通过 WebSocket 接收前端音频 | 服务器运行（推荐） |

`WebSocketRecorder` 的 `put_audio(data)` 方法实现了音频分帧：接收前端传来的任意长度 PCM 数据，按 512 样本 (1024 字节) 分帧后放入队列。

### 4.3 VAD (语音活动检测)

**文件**：`bailing/vad.py`

**抽象基类**：`VAD`

| 方法 | 说明 |
|------|------|
| `is_vad(data)` | 判断音频帧是否有语音活动，返回 `None`（静音）或包含 `"start"`/`"end"` 的字典 |
| `reset_states()` | 重置 VAD 状态 |

**实现类**：

| 类名 | 说明 |
|------|------|
| `SileroVAD` | 基于 Silero VAD 模型，支持配置 `threshold`、`sampling_rate`、`min_silence_duration_ms` |

### 4.4 ASR (语音识别)

**文件**：`bailing/asr.py`

**抽象基类**：`ASR`

| 方法 | 说明 |
|------|------|
| `recognizer(stream_in_audio)` | 接收音频帧列表，返回 `(text, tmpfile)` |

**实现类**：

| 类名 | 模型 | 说明 |
|------|------|------|
| `FunASR` | SenseVoiceSmall | 支持中英日韩语自动识别，需下载模型到本地 |

### 4.5 LLM (大语言模型)

**文件**：`bailing/llm.py`

**抽象基类**：`LLM`

| 方法 | 说明 |
|------|------|
| `response(dialogue)` | 流式生成回复（yield text chunks） |
| `response_call(dialogue, functions_call)` | 支持 function calling 的流式生成（yield (content, tool_calls)） |

**实现类**：

| 类名 | 说明 |
|------|------|
| `OpenAILLM` | 兼容 OpenAI API 接口（支持 DeepSeek、Qwen、Gemini、01Yi 等任何 OpenAI 兼容接口） |
| `OllamaLLM` | 通过 Ollama 本地部署模型（如 Qwen2.5） |

### 4.6 TTS (语音合成)

**文件**：`bailing/tts.py`

**抽象基类**：`AbstractTTS`

| 方法 | 说明 |
|------|------|
| `to_tts(text)` | 将文本转为音频文件，返回文件路径 |

**实现类**：

| 类名 | 技术 | 说明 |
|------|------|------|
| `KOKOROTTS` | Kokoro-82M | 高质量中文 TTS，支持 GPU 加速，推荐使用 |
| `EdgeTTS` | 微软 Edge TTS | 在线 TTS，免费，质量好 |
| `CHATTTS` | ChatTTS | 开源 TTS，支持情感标记 |
| `MacTTS` | macOS say | macOS 系统自带，零依赖 |
| `GTTS` | Google TTS | 谷歌在线 TTS |

### 4.7 Player (播放器)

**文件**：`bailing/player.py`

**基类**：`AbstractPlayer`

所有播放器都实现了一个**消费者线程**模式：内部维护 `play_queue`，后台线程不断取出音频文件播放。

| 方法 | 说明 |
|------|------|
| `play(data)` | 将音频文件加入播放队列 |
| `stop()` | 清空播放队列（打断） |
| `get_playing_status()` | 返回是否正在播放 |
| `shutdown()` | 关闭播放器 |

**实现类**：

| 类名 | 说明 | 适用场景 |
|------|------|---------|
| `WebSocketPlayer` | 通过 WebSocket 将音频发送到前端播放 | 服务器模式（推荐） |
| `PygameSoundPlayer` | 使用 Pygame Sound 播放（支持预加载） | 本地运行 |
| `PygamePlayer` | 使用 Pygame Music 播放 | 本地运行 |
| `PyaudioPlayer` | 使用 PyAudio 播放 | 本地运行 |
| `SoundDevicePlayer` | 使用 sounddevice 播放 | 本地运行 |
| `CmdPlayer` | 通过系统命令（afplay/play）播放 | 本地运行 |

### 4.8 Dialogue (对话管理)

**文件**：`bailing/dialogue.py`

管理对话历史，包含两个核心类：

- **`Message`**：消息对象，包含 `role`、`content`、`tool_calls`、`tool_call_id` 等字段
- **`Dialogue`**：对话历史列表，提供：
  - `put(message)` — 添加消息
  - `get_llm_dialogue()` — 将对话历史转为 LLM 接受的格式（处理 tool_calls 等特殊消息）
  - `dump_dialogue()` — 将对话历史持久化为 JSON 文件

### 4.9 Memory (记忆模块)

**文件**：`bailing/memory.py`

通过 LLM 对历史对话进行摘要，实现长期记忆：

- 启动时读取 `tmp/` 目录下的对话历史文件
- 对未处理的对话历史调用 LLM 生成摘要
- 将摘要嵌入系统 Prompt 中，使 AI 能"记住"用户偏好
- 记忆持久化到 `tmp/memory.json`

### 4.10 RAG (检索增强生成)

**文件**：`bailing/rag.py`

目前已注释，原实现基于 LangChain：
- 使用 `HuggingFaceBgeEmbeddings` 做文档向量化
- 使用 `Chroma` 做向量存储
- 支持 PDF/TXT/MD/WORD 文档

---

## 5. 插件系统详解

### 5.1 插件架构

```
plugins/
├── registry.py                 # 核心：装饰器注册 + 枚举定义
├── task_manager.py             # 核心：任务调度器
├── function_calls_config.json  # LLM Function Calling 的 JSON Schema
└── functions/                  # 具体插件实现
    ├── get_weather.py
    ├── get_day_of_week.py
    ├── schedule_task.py
    ├── web_search.py
    ├── open_application.py
    ├── ielts_speaking_practice.py
    ├── aigc_manus.py
    └── search_local_documents.py
```

### 5.2 注册机制

插件注册使用**装饰器模式**，通过 `@register_function(name, action)` 将函数注册到全局字典 `function_registry`：

```python
from plugins.registry import register_function, ToolType, ActionResponse, Action

@register_function('my_function', ToolType.WAIT)
def my_function(param1: str):
    # 实现逻辑
    return ActionResponse(Action.REQLLM, result, None)
```

`task_manager.py` 在导入时会自动扫描 `plugins/functions/` 目录下的所有模块，触发装饰器注册。

### 5.3 工具类型与动作类型

**ToolType (工具执行策略)**：

| 枚举 | Code | 说明 |
|------|------|------|
| `NONE` | 1 | 调用完工具后，啥也不用管（后台异步） |
| `WAIT` | 2 | 调用工具，同步等待函数返回 |
| `SCHEDULER` | 3 | 定时任务，时间到了之后直接回复 |
| `TIME_CONSUMING` | 4 | 耗时任务，后台运行，有结果后再回复 |
| `ADD_SYS_PROMPT` | 5 | 增加系统 Prompt 到对话历史中去 |

**Action (动作响应类型)**：

| 枚举 | Code | 说明 |
|------|------|------|
| `NOTFOUND` | 0 | 没有找到函数 |
| `NONE` | 1 | 啥也不干 |
| `RESPONSE` | 2 | 直接回复用户 |
| `REQLLM` | 3 | 调用函数后再请求 LLM 生成回复 |
| `ADDSYSTEM` | 4 | 添加系统 Prompt 到对话中去 |
| `ADDSYSTEMSPEAK` | 5 | 添加系统 Prompt 到对话中去并主动说话 |

### 5.4 TaskManager (任务管理器)

**文件**：`plugins/task_manager.py`

TaskManager 是插件系统的调度中心：

1. 从 `function_calls_config.json` 读取 LLM Function Calling 的 Schema
2. 提供 `get_functions()` 方法供 LLM 调用
3. `tool_call(func_name, func_args)` 根据函数的 `ToolType` 决定执行策略：
   - **同步等待**：直接调用并返回结果
   - **异步后台**：提交到线程池，结果放入 `result_queue`
   - **定时任务**：使用 `schedule` 库执行
   - **添加 Prompt**：修改对话上下文

### 5.5 内置插件一览

| 插件名 | ToolType | Action | 功能 |
|--------|----------|--------|------|
| `get_weather` | WAIT | REQLLM | 爬取墨迹天气获取天气信息 |
| `get_day_of_week` | WAIT | REQLLM | 获取当前日期和星期几 |
| `schedule_task` | SCHEDULER | RESPONSE | 创建定时提醒任务 |
| `web_search` | TIME_CONSUMING | REQLLM | DuckDuckGo / 百度网页搜索 |
| `open_application` | NONE | RESPONSE | 打开 macOS 应用程序 |
| `ielts_speaking_practice` | ADD_SYS_PROMPT | ADDSYSTEMSPEAK | 雅思口语练习模式 |
| `aigc_manus` | TIME_CONSUMING | REQLLM | 集成 OpenManus 执行复杂任务 |

---

## 6. 服务层详解

### 6.1 WebSocket 服务 (server.py)

**文件**：`server.py`（项目根目录）

**推荐的运行方式**。基于 FastAPI + WebSocket：

- 启动 HTTPS + WSS 服务（需要 SSL 证书）
- 每个用户通过 `user_id` 参数连接，为每个用户创建独立的 `Robot` 实例
- 支持多用户并发
- 自动清理超时 (600秒) 的连接

**WebSocket 消息协议**：
- 客户端发送：
  - **二进制数据**：PCM 音频帧
  - **JSON 文本**：`{"type": "playback_status", "status": "playing|completed|interrupted", "queue_size": N}`
- 服务端发送：
  - **二进制数据**：TTS 生成的 WAV 音频
  - **JSON 文本**：`{"type": "interrupt"}` 或 `{"type": "update_dialogue", "data": [...]}`

### 6.2 Flask 服务 (server/server.py)

**文件**：`server/server.py`

可选的本地展示服务，基于 Flask + SocketIO：
- 提供一个简单的 Web 页面展示对话内容
- 通过 HTTP POST `/add_message` 接收对话消息
- 通过 SocketIO 实时推送对话更新到前端

### 6.3 前端页面 (static/index.html)

**文件**：`static/index.html`

WebSocket 模式下的前端页面，功能包括：

- **WebRecorder**：通过浏览器 `getUserMedia` API 采集麦克风音频，降采样到 16kHz 并转为 Int16 PCM 发送
- **AudioPlayer**：使用 Web Audio API 解码和播放服务端返回的 WAV 音频，支持队列播放和打断
- **对话展示**：实时展示用户和 AI 的对话内容
- **系统日志**：可折叠的日志面板
- **控制按钮**：开始对话、停止、打断

---

## 7. 配置系统

**文件**：`config/config.yaml`

配置文件使用 YAML 格式，包含以下主要部分：

```yaml
# 基本信息
name: 百聆（bailing）
version: 1.0

# 全局开关
interrupt: true          # 是否允许打断
StartTaskMode: true      # 是否启用工具调用模式

# 模块选择 —— 通过类名指定使用哪个实现
selected_module:
  Recorder: WebSocketRecorder    # 或 RecorderPyAudio
  ASR: FunASR
  VAD: SileroVAD
  LLM: OpenAILLM                 # 或 OllamaLLM
  TTS: KOKOROTTS                 # 或 EdgeTTS, CHATTTS, MacTTS, GTTS
  Player: WebSocketPlayer        # 或 PygameSoundPlayer, PyaudioPlayer 等

# 各模块的配置参数
Recorder:
  RecorderPyAudio: ...
  WebSocketRecorder: ...

ASR:
  FunASR:
    model_dir: FunAudioLLM/SenseVoiceSmall
    output_file: tmp/

VAD:
  SileroVAD:
    sampling_rate: 16000
    threshold: 0.5
    min_silence_duration_ms: 200

LLM:
  OpenAILLM:
    model_name: deepseek-chat
    url: https://api.deepseek.com
    api_key: <your-api-key>
  OllamaLLM:
    model_name: qwen-chat
    url: http://localhost:11434/api/chat

TTS:
  KOKOROTTS:
    output_file: tmp/
    lang: z
    voice: zf_001
    repo_id: hexgrad/Kokoro-82M-v1.1-zh
  EdgeTTS:
    voice: zh-CN-XiaoxiaoNeural
    output_file: tmp/
  # ... 其他 TTS 配置

Player:
  WebSocketPlayer: null
  PygameSoundPlayer: null
  # ...

# 记忆配置
Memory:
  dialogue_history_path: tmp/
  memory_file: tmp/memory.json
  model_name: deepseek-chat
  url: https://api.deepseek.com
  api_key: <your-api-key>

# 插件配置
TaskManager:
  functions_call_name: plugins/function_calls_config.json
  aigc_manus_enabled: false
```

**核心设计**：`selected_module` 下的键值对应模块类型，值是**具体实现类的类名**。系统通过 `create_instance(class_name, config)` 工厂方法动态实例化。

---

## 8. 第三方依赖

### Python 核心依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `torch` / `torchaudio` | 2.4.1 | PyTorch（模型运行框架） |
| `funasr` | 1.1.6 | ASR 语音识别引擎 |
| `silero_vad` | 5.1 | VAD 语音活动检测 |
| `openai` | 1.97.0 | OpenAI 兼容 API 客户端 |
| `edge_tts` | 6.1.12 | 微软 Edge TTS |
| `chattts` | 0.1.1 | ChatTTS 语音合成 |
| `kokoro` / `misaki[zh]` | >=0.8.1 | Kokoro TTS 中文语音合成 |
| `transformers` | 4.41.1 | Hugging Face Transformers |
| `fastapi` | 0.116.1 | WebSocket 服务框架 |
| `uvicorn` | 0.35.0 | ASGI 服务器 |
| `flask` / `flask-socketio` | 3.0.3 / 5.3.7 | Flask Web 框架 |
| `pyaudio` | 0.2.14 | 本地音频采集/播放 |
| `pygame` | 2.6.1 | 本地音频播放 |
| `sounddevice` / `soundfile` | 0.5.0 / 0.12.1 | 音频设备/文件操作 |
| `pydub` | 0.25.1 | 音频格式转换 |
| `schedule` | 1.2.2 | 定时任务 |
| `beautifulsoup4` | 4.12.2 | 网页解析（天气/搜索插件） |
| `requests` / `httpx` | 2.32.2 / 0.28.1 | HTTP 请求 |
| `PyYAML` | 6.0.2 | YAML 配置读取 |

### 第三方项目

- **OpenManus**：位于 `third_party/OpenManus/`，提供通用 AI Agent 能力，用于 `aigc_manus` 插件

---

## 9. 二次开发指南

所有核心模块都遵循**抽象基类 + 工厂方法**的设计模式，二次开发流程统一为以下三步：

1. **实现子类**：继承对应的抽象基类，实现必要方法
2. **更新配置**：在 `config.yaml` 中添加新类的配置
3. **选择使用**：在 `selected_module` 中指定新类名

### 9.1 添加新的 ASR 引擎

**步骤**：

1. 在 `bailing/asr.py` 中继承 `ASR` 基类：

```python
class MyASR(ASR):
    def __init__(self, config):
        # 读取配置，初始化模型
        self.model = ...

    def recognizer(self, stream_in_audio):
        """
        Args:
            stream_in_audio: list[bytes]，原始 PCM 音频帧列表

        Returns:
            (text: str, tmpfile: str)  识别文本和临时音频文件路径
        """
        # 1. 将音频帧保存为 WAV 文件
        tmpfile = "tmp/asr-xxx.wav"
        self._save_audio_to_file(stream_in_audio, tmpfile)

        # 2. 调用你的 ASR 模型
        text = self.model.transcribe(tmpfile)

        return text, tmpfile
```

2. 在 `config.yaml` 中添加配置：

```yaml
ASR:
  MyASR:
    model_path: /path/to/model
    # 其他配置

selected_module:
  ASR: MyASR
```

### 9.2 添加新的 LLM 后端

**步骤**：

1. 在 `bailing/llm.py` 中继承 `LLM` 基类：

```python
class MyLLM(LLM):
    def __init__(self, config):
        self.api_key = config.get("api_key")
        # 初始化客户端

    def response(self, dialogue):
        """
        Args:
            dialogue: list[dict]，对话历史，格式 [{"role": "user", "content": "..."}]

        Yields:
            str: 文本片段（流式输出）
        """
        for chunk in self.client.stream(dialogue):
            yield chunk.text

    def response_call(self, dialogue, functions_call):
        """
        支持 function calling 的流式生成

        Args:
            dialogue: 对话历史
            functions_call: list[dict]，工具定义 JSON Schema

        Yields:
            (content: str | None, tool_calls: list | None)
        """
        for chunk in self.client.stream(dialogue, tools=functions_call):
            yield chunk.content, chunk.tool_calls
```

2. 在 `config.yaml` 中添加配置：

```yaml
LLM:
  MyLLM:
    api_key: xxx
    model_name: my-model

selected_module:
  LLM: MyLLM
```

### 9.3 添加新的 TTS 引擎

**步骤**：

1. 在 `bailing/tts.py` 中继承 `AbstractTTS` 基类：

```python
class MyTTS(AbstractTTS):
    def __init__(self, config):
        self.output_file = config.get("output_file", "tmp/")
        # 初始化 TTS 引擎

    def to_tts(self, text):
        """
        Args:
            text: str，待合成的文本

        Returns:
            str: 生成的音频文件路径（WAV 格式），失败返回 None
        """
        tmpfile = os.path.join(self.output_file, f"tts-{uuid.uuid4().hex}.wav")
        # 调用你的 TTS 引擎
        audio = self.engine.synthesize(text)
        # 保存为 WAV
        sf.write(tmpfile, audio, 24000)
        return tmpfile
```

2. 在 `config.yaml` 中添加配置：

```yaml
TTS:
  MyTTS:
    output_file: tmp/
    model_path: /path/to/model

selected_module:
  TTS: MyTTS
```

### 9.4 添加新的 VAD 引擎

**步骤**：

1. 在 `bailing/vad.py` 中继承 `VAD` 基类：

```python
class MyVAD(VAD):
    def __init__(self, config):
        # 初始化 VAD 模型
        pass

    def is_vad(self, data):
        """
        Args:
            data: bytes，512 样本的 16kHz Int16 PCM 音频帧

        Returns:
            None: 静音/无语音
            dict: 包含 "start" 键 → 语音开始；包含 "end" 键 → 语音结束
        """
        # 你的 VAD 逻辑
        if speech_started:
            return {"start": timestamp}
        elif speech_ended:
            return {"end": timestamp}
        return None

    def reset_states(self):
        # 重置模型状态
        pass
```

2. 更新配置文件。

### 9.5 添加新的 Recorder

**步骤**：

1. 在 `bailing/recorder.py` 中继承 `AbstractRecorder`：

```python
class MyRecorder(AbstractRecorder):
    def __init__(self, config):
        pass

    def start_recording(self, audio_queue: queue.Queue):
        """
        开始录音，将 PCM 音频数据 (512 samples, 16kHz, Int16) 放入 audio_queue
        """
        # 启动录音线程
        pass

    def stop_recording(self):
        """停止录音"""
        pass
```

2. 更新配置文件。

### 9.6 添加新的 Player

**步骤**：

1. 在 `bailing/player.py` 中继承 `AbstractPlayer`：

```python
class MyPlayer(AbstractPlayer):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 初始化播放器

    def do_playing(self, audio_file):
        """
        播放单个音频文件的具体实现

        Args:
            audio_file: str，WAV 格式的音频文件路径
        """
        # 你的播放逻辑
        pass
```

**注意**：`AbstractPlayer` 基类已实现了播放队列、消费者线程、停止和关闭等通用逻辑，子类只需实现 `do_playing` 方法。

2. 更新配置文件。

### 9.7 添加新的插件/工具函数

这是最常见的二次开发场景。步骤如下：

**第1步**：在 `plugins/functions/` 目录下创建新文件，例如 `my_tool.py`：

```python
from plugins.registry import register_function, ToolType, ActionResponse, Action

@register_function('my_tool', action=ToolType.WAIT)
def my_tool(param1: str, param2: int = 10):
    """
    我的自定义工具

    Args:
        param1: 参数1说明
        param2: 参数2说明（可选）
    """
    # 实现你的工具逻辑
    result = do_something(param1, param2)

    # 返回 ActionResponse
    # Action.REQLLM → 将 result 作为工具结果返回给 LLM 继续生成回复
    # Action.RESPONSE → 直接将 response 作为语音回复用户
    return ActionResponse(
        action=Action.REQLLM,
        result=str(result),  # 工具执行结果（发给 LLM）
        response=None        # 直接回复内容（当 action=RESPONSE 时使用）
    )
```

**第2步**：在 `plugins/function_calls_config.json` 中添加 LLM Function Calling Schema：

```json
{
    "type": "function",
    "function": {
        "name": "my_tool",
        "description": "我的自定义工具的描述，LLM 据此决定何时调用",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "参数1的描述"
                },
                "param2": {
                    "type": "integer",
                    "description": "参数2的描述"
                }
            },
            "required": ["param1"]
        }
    }
}
```

**第3步**：无需修改其他任何代码！`task_manager.py` 会自动扫描并加载新插件。

**ToolType 选择指南**：
- **WAIT**：适合快速返回的工具（如查天气、查时间）
- **TIME_CONSUMING**：适合耗时操作（如搜索、AIGC 任务），系统会先回复"正在处理"
- **NONE**：适合不需要返回结果的工具（如打开应用）
- **SCHEDULER**：适合定时触发的工具
- **ADD_SYS_PROMPT**：适合需要改变对话模式的工具（如雅思口语练习）

### 9.8 自定义 Prompt

**文件**：`bailing/prompt.py`

修改 `sys_prompt` 可以改变 AI 的人格和行为：

```python
sys_prompt = """
# 角色定义
你是百聆，由寒江雪开发。...

# 以下是历史对话摘要:
{memory}

# 回复要求
1. 你的回复应该简短、友好...
2. 如果需要调用工具...
"""
```

`{memory}` 占位符会在运行时被 Memory 模块生成的对话摘要替换。

修改 `memory_prompt_template` 可以改变记忆摘要的生成策略。

### 9.9 自定义打断策略

打断策略在 `bailing/robot.py` 的 `_duplex()` 方法中实现：

```python
if "start" in vad_status:
    if self.player.get_playing_status() or self.chat_lock is True:
        if self.INTERRUPT:
            self.chat_lock = False
            self.interrupt_playback()
            self.vad_start = True
            self.speech.append(data)
```

通过 `config.yaml` 的 `interrupt: true/false` 控制是否允许打断。

如需更精细的打断策略（如关键字打断），可参考 `bailing/utils.py` 中的 `is_interrupt()` 函数。

---

## 10. 常见问题与排错

### Q1: 如何在无 GPU 环境下运行？

项目默认支持 CPU 运行。VAD (Silero) 和 ASR (FunASR/SenseVoice) 都支持 CPU 推理。TTS 推荐使用 `EdgeTTS`（在线）或 `KOKOROTTS`（自动检测 CPU/GPU）。

### Q2: 如何更换 LLM 模型？

修改 `config.yaml` 中 `LLM.OpenAILLM` 的配置即可。任何兼容 OpenAI API 的模型都可以使用：
- DeepSeek: `url: https://api.deepseek.com`
- OpenAI: `url: https://api.openai.com/v1`
- 本地 Ollama: 使用 `OllamaLLM` 类

### Q3: WebSocket 模式如何部署？

```bash
# 1. 生成自签名证书
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# 2. 启动服务
python server.py

# 3. 浏览器访问 https://<your-ip>:8000
```

### Q4: 如何调整 VAD 灵敏度？

修改 `config.yaml` 中 VAD 配置：
- `threshold`: 值越低越灵敏（默认 0.5）
- `min_silence_duration_ms`: 静音判断时长，越大则对停顿越宽容（默认 200ms）

### Q5: 为什么 TTS 延迟较高？

项目使用了**分句 + 流式 TTS**策略来降低感知延迟：
- LLM 输出按标点符号分句（逗号、句号、问号等）
- 每个句子独立提交 TTS 任务
- TTS 结果按顺序排队播放

如果延迟仍较高，可尝试：
1. 使用 `EdgeTTS`（网络延迟低）
2. 降低 `min_silence_duration_ms`
3. 使用更快的 LLM 模型

### Q6: 模块之间的线程模型是怎样的？

```
主线程: Robot.run() → _duplex() 循环
线程1: Recorder 录音线程 → audio_queue
线程2: VAD 处理线程 (audio_queue → vad_queue)
线程3: TTS 优先级线程 (tts_queue → Player)
线程4: Player 消费者线程 (play_queue → 播放)
线程池: ThreadPoolExecutor (LLM/TTS/Tool 异步任务)
```

### Q7: 如何添加多语言支持？

- ASR: FunASR 已支持中英日韩语自动识别
- TTS: 更换 TTS 引擎或声音参数即可（如 EdgeTTS 支持多种语言声音）
- LLM: 大多数 LLM 天然支持多语言

---

> 本文档基于项目源码自动生成，如有疑问请参考源码或提交 Issue。
