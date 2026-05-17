# xiaozhi-esp32-server 项目结构全解析

> 项目地址: https://github.com/xinnan-tech/xiaozhi-esp32-server  
> 技术栈: Python + Java + Vue  
> 简介: xiaozhi-esp32 配套后端服务，实现 ASR → LLM → TTS 全链路语音对话，集成 MCP/声纹识别/知识库/记忆系统

---

## 整体架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│  ESP32 设备 (xiaozhi-esp32)                                          │
│  WebSocket / MQTT+UDP                                                │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    xiaozhi-esp32-server                               │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  xiaozhi-server (Python)  — 核心语音对话引擎                     │ │
│  │                                                                  │ │
│  │  WebSocket Server (8000)    HTTP Server (8003)                  │ │
│  │       │                          │                               │ │
│  │       ▼                          ▼                               │ │
│  │  ConnectionHandler           OTA 固件下发                        │ │
│  │       │                     视觉分析 API                         │ │
│  │       ▼                                                         │ │
│  │  ┌─────────────────────────────────────────┐                    │ │
│  │  │         语音处理流水线                     │                    │ │
│  │  │  VAD检测 → ASR识别 → 意图识别             │                    │ │
│  │  │                ↓                          │                    │ │
│  │  │           LLM对话 (流式)                   │                    │ │
│  │  │                ↓                          │                    │ │
│  │  │           TTS合成 → Opus音频帧             │                    │ │
│  │  │                                          │                    │ │
│  │  │  辅助模块:                                │                    │ │
│  │  │  声纹识别 | 记忆系统 | MCP工具 | RAG知识库  │                    │ │
│  │  └─────────────────────────────────────────┘                    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────┐  ┌──────────────────────────────────┐ │
│  │  manager-api (Java)      │  │  manager-web (Vue.js)            │ │
│  │  Spring Boot REST API    │  │  管理后台 UI                      │ │
│  │  用户/设备/智能体管理      │  │  多语言 (中/英/繁)                │ │
│  │  MySQL + Redis           │  └──────────────────────────────────┘ │
│  └──────────────────────────┘                                       │
│                                                                      │
│  ┌──────────────────────────┐                                       │
│  │  digital-human (测试前端) │                                       │
│  │  Live2D 虚拟形象 + 测试页 │                                       │
│  └──────────────────────────┘                                       │
└──────────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐         ┌──────────┐
   │ ASR 服务  │        │ LLM 服务 │         │ TTS 服务 │
   │ 13种     │        │ 13种     │         │ 18种     │
   └──────────┘        └──────────┘         └──────────┘
```

---

## 一、目录树总览

```
xiaozhi-esp32-server/
├── README.md                          # 中文 README
├── LICENSE                            # MIT 许可证
│
├── Dockerfile-server                  # Python 服务端生产镜像
├── Dockerfile-server-base             # 基础镜像 (系统依赖 + Python 包)
├── Dockerfile-web                     # 完整 Web 管理后台镜像 (Java + Nginx)
├── docker-setup.sh                    # 一键安装脚本 (Ubuntu/Debian)
│
├── docs/                              # 文档 (~30篇)
│   ├── Deployment.md                  # 精简部署指南
│   ├── Deployment_all.md              # 完整模块部署指南
│   ├── FAQ.md                         # 常见问题
│   ├── firmware-build.md              # ESP32 固件编译指南
│   ├── ota-upgrade-guide.md           # OTA 升级指南
│   ├── voiceprint-integration.md      # 声纹识别集成
│   ├── ragflow-integration.md         # RAGFlow 知识库集成
│   ├── mqtt-gateway-integration.md    # MQTT 网关集成
│   ├── mcp-endpoint-integration.md    # MCP 端点集成
│   ├── homeassistant-integration.md   # Home Assistant 集成
│   ├── fish-speech-integration.md     # FishSpeech TTS 集成
│   ├── powermem-integration.md        # PowerMem 记忆系统
│   ├── context-provider-integration.md # 上下文提供者
│   ├── all-in-one-digital-human-setup.md # 数字人一体机
│   ├── contributor_open_letter.md     # 致开发者的公开信
│   └── readme/                        # 多语言 README (英/德/越/葡)
│
└── main/                              # 所有源代码
    │
    ├── xiaozhi-server/                # 核心语音服务器 (Python)
    │   ├── app.py                     # 入口点
    │   ├── config.yaml                # 主配置文件 (~1170行)
    │   ├── config_from_api.yaml       # 智控台模式配置模板
    │   ├── agent-base-prompt.txt      # Jinja2 系统提示词模板
    │   ├── mcp_server_settings.json   # MCP 服务配置
    │   ├── requirements.txt           # Python 依赖
    │   ├── docker-compose.yml         # 精简 Docker Compose
    │   ├── docker-compose_all.yml     # 完整 Docker Compose
    │   │
    │   ├── config/                    # 配置基础设施
    │   │   ├── config_loader.py       # 配置加载/合并/缓存
    │   │   ├── logger.py              # Loguru 日志系统
    │   │   ├── settings.py            # 配置校验
    │   │   └── manage_api_client.py   # manager-api REST 客户端
    │   │
    │   ├── core/                      # 引擎核心
    │   │   ├── websocket_server.py    # WebSocket 服务器 (端口 8000)
    │   │   ├── connection.py          # 连接处理器 (核心文件, ~1656行)
    │   │   ├── http_server.py         # HTTP 服务器 (端口 8003)
    │   │   │
    │   │   ├── handle/                # 消息处理器
    │   │   │   ├── helloHandler.py    # 客户端握手
    │   │   │   ├── abortHandler.py    # 打断处理
    │   │   │   ├── listenHandler.py   # 聆听模式控制
    │   │   │   ├── iotHandler.py      # IoT 指令
    │   │   │   ├── mcpHandler.py      # MCP 工具调用
    │   │   │   ├── serverHandler.py   # 服务器管理
    │   │   │   ├── pingHandler.py     # 心跳保活
    │   │   │   ├── receiveAudioHandle.py # 音频接收流程
    │   │   │   ├── sendAudioHandle.py # 音频发送流程
    │   │   │   ├── intentHandler.py   # 意图识别路由
    │   │   │   ├── reportHandle.py    # 数据上报
    │   │   │   └── abortHandle.py     # 打断逻辑
    │   │   │
    │   │   ├── providers/             # 可插拔 AI 服务抽象
    │   │   │   ├── asr/               # ASR 语音识别 (13种)
    │   │   │   ├── llm/               # LLM 大模型 (13种)
    │   │   │   ├── tts/               # TTS 语音合成 (18种)
    │   │   │   ├── vad/               # VAD 语音活动检测
    │   │   │   ├── vllm/              # VLLM 视觉模型
    │   │   │   ├── memory/            # 记忆系统 (6种)
    │   │   │   ├── intent/            # 意图识别 (3种)
    │   │   │   ├── afe/               # 音频前端处理
    │   │   │   └── tools/             # 工具系统 (5类)
    │   │   │
    │   │   └── utils/                 # 工具函数
    │   │       ├── asr.py / llm.py / tts.py / memory.py / vad.py  # 工厂方法
    │   │       ├── modules_initialize.py  # 集中式模块初始化
    │   │       ├── prompt_manager.py      # Jinja2 提示词构建
    │   │       ├── context_provider.py    # 动态上下文 (天气/时间/位置)
    │   │       ├── voiceprint_provider.py # 声纹识别客户端
    │   │       ├── dialogue.py            # 对话上下文管理
    │   │       ├── auth.py                # JWT 认证
    │   │       ├── cache/                 # 内存缓存 (TTL策略)
    │   │       ├── audioRateController.py # 音频速率控制
    │   │       ├── output_counter.py      # 每日输出限制
    │   │       ├── gc_manager.py          # 垃圾回收管理器
    │   │       └── opus_encoder_utils.py  # Opus 编解码工具
    │   │
    │   ├── plugins_func/functions/     # 功能插件
    │   │   ├── get_weather.py          # 天气查询
    │   │   ├── get_news_from_chinanews.py # 新闻获取
    │   │   ├── play_music.py           # 音乐播放
    │   │   ├── change_role.py          # 角色切换
    │   │   ├── web_search.py           # 网络搜索
    │   │   ├── search_from_ragflow.py  # RAGFlow 知识库
    │   │   ├── handle_exit_intent.py   # 退出对话
    │   │   └── hass_*.py               # Home Assistant 集成
    │   │
    │   ├── models/                     # 本地模型
    │   │   ├── SenseVoiceSmall/        # ASR 模型
    │   │   └── snakers4_silero-vad/    # VAD 模型
    │   │
    │   ├── performance_tester/         # 性能测试工具
    │   ├── config/assets/              # 音频资源
    │   │   ├── wakeup_words.wav        # 唤醒提示音
    │   │   ├── tts_notify.mp3          # 停止提示音
    │   │   └── bind_code/              # 绑定验证码播报
    │   └── music/                      # 示例音乐文件
    │
    ├── digital-human/                  # 数字人前端 (测试用)
    │   ├── start.py                    # 启动脚本 (端口 8006)
    │   ├── index.html                  # 测试页面
    │   ├── wakeword_runtime/           # 本地唤醒词运行时
    │   ├── js/live2d/                  # Live2D Cubism 4 虚拟形象
    │   └── js/opus/                    # Opus 音频处理
    │
    ├── manager-api/                    # Java 管理后台 API
    │   ├── pom.xml                     # Maven 构建 (Java 21, Spring Boot)
    │   └── src/main/java/xiaozhi/
    │       ├── AdminApplication.java   # 入口
    │       ├── common/                 # 通用工具
    │       └── modules/                # 业务模块 (用户/设备/智能体/日志)
    │
    ├── manager-web/                    # Vue 管理界面
    │   └── src/                        # Vue 2 + Element UI
    │
    └── manager-mobile/                 # Uni-app 移动管理端
        └── src/                        # 跨平台移动应用
```

---

## 二、启动流程

### 入口 app.py

```
app.py 启动
  ├── 1. 检查 ffmpeg 是否安装
  ├── 2. 加载配置
  │     ├── config.yaml (精简模式)
  │     ├── data/.config.yaml (用户配置覆盖)
  │     └── manager-api (智控台模式, 远程拉取)
  ├── 3. 解析 auth_key → JWT 认证管理器
  ├── 4. 验证 MCP 端点 URL
  ├── 5. 启动三个并发任务:
  │     ├── WebSocketServer   → 端口 8000, 设备连接
  │     ├── SimpleHttpServer  → 端口 8003, OTA + 视觉分析
  │     └── GCManager         → 每 5 分钟垃圾回收
  └── 6. 进入异步事件循环 (asyncio)
```

### 设备连接流程

```
ESP32 WebSocket 连接 ws://host:8000/xiaozhi/v1/
  │
  ├── 1. JWT 认证 (可选) + 设备白名单校验
  │
  ├── 2. 接收 hello 消息
  │     └── 协商参数: 音频格式=opus, 采样率=24000Hz, 帧长=60ms
  │
  ├── 3. 后台初始化 (_background_initialize)
  │     └── 异步拉取私有配置 (LLM/TTS选择, 提示词, 声纹等)
  │
  └── 4. 进入消息循环 (_route_message)
        ├── 文本消息 → TextMessageHandlerRegistry (7种类型)
        └── 二进制消息 → ASR 音频管道
```

---

## 三、核心子系统详解

### 3.1 语音处理流水线 (最核心)

```
ESP32 设备
  │
  ▼ WebSocket 二进制帧 (Opus 编码, 24000Hz, 60ms)
┌──────────────────────────────────────────────────────┐
│  receiveAudioHandle.py                               │
│                                                      │
│  1. Opus 解码 → PCM                                  │
│       │                                              │
│  2. VAD 检测 (SileroVAD)                             │
│     ├── 无声: 丢弃                                   │
│     └── 有声: 送入 ASR                               │
│       │                                              │
│  3. ASR 识别 (13种可选)                              │
│     ├── 流式: 实时返回中间结果                        │
│     └── 非流式: 等语音结束返回完整文本                │
│       │                                              │
│  4. 意图识别 (3种模式)                                │
│     ├── nointent: 跳过                                │
│     ├── intent_llm: 独立 LLM 分类                     │
│     └── function_call: LLM 原生工具调用               │
│       │                                              │
│  5. LLM 对话 (13种可选)                               │
│     ├── Jinja2 提示词 + 上下文注入                    │
│     ├── 声纹发言人信息注入                            │
│     ├── 记忆检索注入                                  │
│     ├── 知识库检索注入                                │
│     ├── 流式输出 → TTS 队列                           │
│     └── function_call → UnifiedToolHandler            │
│       │                                              │
│  6. TTS 合成 (18种可选)                               │
│     ├── 流式: 边合成边发送                            │
│     ├── 句子分段: FIRST / MIDDLE / LAST               │
│     └── Opus 编码 → WebSocket 发回设备                │
│                                                      │
│  7. 打断处理                                         │
│     └── 收到 abort 消息 → 清空 TTS 队列 → 停止播放    │
└──────────────────────────────────────────────────────┘
  │
  ▼ WebSocket 二进制帧 (Opus 编码)
ESP32 设备 → 扬声器播放
```

### 3.2 AI 服务提供者矩阵

#### ASR 语音识别 (13种)

| 类型 | 提供者 | 特点 |
|------|--------|------|
| 本地 | FunASR (SenseVoice) | 离线可用, 多语言, 2核4G |
| 本地 | Sherpa-ONNX | 轻量离线 |
| 本地 | Vosk | 轻量离线 |
| 流式 | 讯飞流式 | 低延迟 |
| 流式 | 豆包流式 | 火山引擎 |
| 流式 | 阿里百炼流式 | 阿里云 |
| 非流式 | 阿里云 / 百度 / 腾讯 / OpenAI / Qwen3-Flash | 云端 API |

#### LLM 大模型 (13种)

| 提供者 | 接口 |
|--------|------|
| 阿里百炼 (AliBL) | OpenAI 兼容 |
| DeepSeek | OpenAI 兼容 |
| 智谱 GLM | OpenAI 兼容 |
| 豆包 (火山引擎) | OpenAI 兼容 |
| 科大讯飞 | 自定义适配 |
| Gemini | OpenAI 兼容 |
| Coze / Dify / FastGPT | OpenAI 兼容 |
| Ollama / Xinference | OpenAI 兼容 (本地部署) |
| HomeAssistant | 自定义适配 |

#### TTS 语音合成 (18种)

| 类型 | 提供者 | 说明 |
|------|--------|------|
| 流式 | 火山引擎流式 | 音色克隆, 边合成边播放 |
| 流式 | 阿里云流式 / 阿里百炼流式 | 低延迟 |
| 流式 | 讯飞流式 / Minimax / Index-TTS | 低延迟 |
| 免费 | EdgeTTS (微软) | 免费可用 |
| 免费 | 灵犀流式 | 免费 |
| 本地 | FishSpeech / GPT-SoVITS v2/v3 | 声音克隆 |
| 云端 | 腾讯云 / OpenAI / PaddleSpeech | 自定义 |

#### 记忆系统 (6种)

| 方案 | 特点 |
|------|------|
| nomem | 无记忆 |
| mem_local_short | 本地短期记忆, LLM 自动总结 |
| mem0ai | 云端记忆服务 (每月1000次免费) |
| PowerMem | OceanBase/SQLite/SeekDB 持久记忆 |
| mem_report_only | 仅报告 |

#### 意图识别 (3种)

| 方案 | 说明 |
|------|------|
| nointent | 跳过意图识别 |
| intent_llm | 独立 LLM 做意图分类 |
| function_call | 利用 LLM 原生 function call 能力 |

### 3.3 消息处理系统

7 种已注册消息类型处理器:

| 消息类型 | 处理器 | 功能 |
|----------|--------|------|
| `hello` | helloHandler | 握手: 设备 ID, 协议版本, 音频参数协商 |
| `abort` | abortHandler | 打断: 停止 TTS, 清空队列 |
| `listen` | listenHandler | 模式切换: auto 模式 (VAD自动) / manual 模式 (按下按钮) |
| `iot` | iotHandler | IoT 指令: 音量调节, 灯光控制等 |
| `mcp` | mcpHandler | MCP 工具: 客户端 MCP 工具调用 |
| `server` | serverHandler | 管理: 重启服务等 |
| `ping` | pingHandler | 心跳: WebSocket 保活 |

### 3.4 MCP 工具系统 (5 类)

```
UnifiedToolHandler (统一工具处理器)
  │
  ├── 1. SERVER_PLUGIN (服务端插件)
  │     └── plugins_func/functions/*.py
  │         天气 / 新闻 / 音乐 / 搜索 / 角色切换 / RAGFlow
  │
  ├── 2. SERVER_MCP (服务端 MCP)
  │     └── 连接外部 MCP 服务器 (stdio/SSE/HTTP)
  │         文件系统 / Playwright 浏览器 / 任意 MCP 服务
  │
  ├── 3. DEVICE_IOT (设备端 IoT)
  │     └── 通过客户端会话下发 IoT 指令
  │         固件升级 / 重启 / 音量调节
  │
  ├── 4. DEVICE_MCP (设备端 MCP)
  │     └── ESP32 设备暴露为 MCP 工具
  │         截图 / 拍照 / 灯光控制
  │
  └── 5. MCP_ENDPOINT (MCP 端点)
        └── 外部 MCP 客户端经 WebSocket 连接
            智能家居 PC桌面操作 知识搜索 邮件
```

### 3.5 声纹识别

```
音频流 ──→ ASR 识别 (文本)
   │
   └──→ 声纹服务器 (3D-Speaker) ──→ 说话人 ID
                                      │
                                      ▼
                    LLM 提示词注入:
                    {"speaker": "张三", "content": "今天天气怎么样"}
                                      │
                                      ▼
                    个性化回应: "张三你好, 今天北京晴, 25°C..."
```

配置方式 (`config.yaml`):
```yaml
voiceprint:
  enabled: true
  server_url: "http://voiceprint-server:8001"
  speakers:
    - "speaker_001,张三,男主人"
    - "speaker_002,李四,女主人"
```

### 3.6 上下文注入系统

LLM 提示词通过 Jinja2 模板动态注入实时上下文:

| 变量 | 来源 | 示例 |
|------|------|------|
| `current_time` | 系统时间 | 2026-05-17 15:30:00 |
| `lunar_date` | 农历计算 | 四月二十 |
| `weather` | 和风天气 API | 晴, 25°C |
| `location` | IP 定位 | 北京 |
| `speaker_info` | 声纹识别 | 张三 (男主人) |
| `memory` | 记忆系统 | 上次聊过的话题 |
| `knowledge` | RAGFlow | 相关文档片段 |

### 3.7 OTA 固件更新

```
HTTP GET/POST /xiaozhi/ota/
  │
  ├── 1. JWT 令牌验证
  ├── 2. 返回最新固件信息:
  │     {
  │       "version": "2.2.6",
  │       "url": "http://host:8003/xiaozhi/ota/download/firmware.bin",
  │       "protocol": "websocket",  // 或 "mqtt"
  │       "activation_code": null   // 如需激活
  │     }
  └── 3. 设备下载固件 → 写入 OTA 分区 → 校验 → 重启
```

---

## 四、部署架构

### 三种部署模式

| 模式 | 组件 | 资源需求 | 适用场景 |
|------|------|----------|----------|
| **精简版** | xiaozhi-server 单容器 | FunASR: 2核4G / 全API: 2核2G | 个人/家庭 |
| **全模块** | server + manager-api + manager-web + MySQL + Redis | FunASR: 4核8G / 全API: 2核4G | 多用户/演示 |
| **一体机** | 全部 + digital-human + Live2D | 更高 | 展厅/商业 |

### Docker 部署 (精简版)

```bash
cd xiaozhi-esp32-server

# 构建基础镜像
docker build -f Dockerfile-server-base -t xiaozhi-server-base .

# 构建运行镜像
docker build -f Dockerfile-server -t xiaozhi-server .

# 启动
docker run -d \
  -p 8000:8000 \
  -p 8003:8003 \
  -v $(pwd)/main/xiaozhi-server/config.yaml:/opt/xiaozhi-esp32-server/config.yaml \
  xiaozhi-server
```

### Docker 部署 (全模块)

```bash
# docker-compose_all.yml 包含:
# - xiaozhi-server (Python)
# - manager-api (Java)
# - manager-web (Nginx + Vue)
# - MySQL 8.0
# - Redis 7

docker compose -f main/xiaozhi-server/docker-compose_all.yml up -d
```

### 源码部署

```bash
cd main/xiaozhi-server
pip install -r requirements.txt
python app.py
```

---

## 五、配置体系

### 主配置文件 config.yaml (~1170行)

```yaml
server:
  websocket:
    host: "0.0.0.0"
    port: 8000
  http:
    port: 8003
  auth_key: ""           # JWT 密钥
  mqtt_gateway: {}       # MQTT 网关配置
  udp_gateway: {}        # UDP 网关配置

asr:                     # 语音识别提供者选择
  type: "fun_local"      # 或 xunfei_stream / aliyun / doubao ...
  fun_local:
    model_dir: "models/SenseVoiceSmall"

llm:                     # 大模型提供者选择
  type: "openai"
  openai:
    api_key: ""
    base_url: "https://api.openai.com/v1"
    model: "gpt-4"

tts:                     # 语音合成提供者选择
  type: "edge"
  edge:
    voice: "zh-CN-XiaoxiaoNeural"

vad:                     # 语音活动检测
  type: "silero"
  silero:
    threshold: 0.5

voiceprint:              # 声纹识别
  enabled: false

memory:                  # 记忆系统
  type: "nomem"

intent:                  # 意图识别
  type: "function_call"

ragflow:                 # RAGFlow 知识库
  enabled: false

plugins:                 # 功能插件列表
  - weather
  - web_search
  - play_music
```

### 配置加载优先级

```
智控台模式: config_from_api.yaml → manager-api (远程拉取)
精简模式:   config.yaml → data/.config.yaml (用户覆盖)
```

---

## 六、数字人模块 (digital-human)

```
digital-human/
├── start.py              → 启动 HTTP 服务 (端口 8006)
├── index.html            → 浏览器测试页面
│     ├── 3D 虚拟形象 (Live2D Cubism 4)
│     ├── 实时对话文本显示
│     ├── 音频录制/播放
│     └── MCP 工具测试面板
├── wakeword_runtime/     → 本地唤醒词引擎
│     └── 检测到唤醒词 → WebSocket 事件 → xiaozhi-server
├── js/
│   ├── live2d/           → Hiyori 免费模型 + Cubism SDK
│   ├── opus/             → 浏览器端 Opus 编解码
│   └── audio/            → 音频流处理
└── assets/               → UI 资源
```

### 数字人工作流程

```
浏览器麦克风 → Opus 编码 → WebSocket → xiaozhi-server
                                                ↓
                                      ASR → LLM → TTS
                                                ↓
浏览器扬声器 ← Opus 解码 ← WebSocket ←─── Opus 音频帧
       +
Live2D 虚拟形象 (根据情感改变表情和动作)
```

---

## 七、智控台 (管理后台)

### 架构

```
manager-web (Vue 2 + Element UI)
     │
     ▼ REST API
manager-api (Java 21, Spring Boot + MyBatis-Plus)
     │
     ├── MySQL (业务数据: 用户/设备/智能体/对话记录)
     └── Redis (缓存/会话)
```

### 功能模块

| 模块 | 功能 |
|------|------|
| 用户管理 | 注册/登录/权限/设备绑定 |
| 智能体管理 | 角色提示词/模型选择/个性配置 |
| 设备管理 | OTA 固件/配置同步/在线状态 |
| 对话历史 | 存储和检索 |
| 资源配置 | 字体/唤醒词/Emoji 在线生成 |
| 声纹管理 | 注册/管理/识别 |
| 多语言 | 简体中文/繁体中文/英文 |

---

## 八、通信协议汇总

| 协议 | 端口 | 用途 |
|------|------|------|
| **WebSocket** | 8000 | 设备对话主通道 (信令 + 音频) |
| **HTTP** | 8003 | OTA 固件 / 视觉分析 / 文件下载 |
| **WebSocket** | 8006 | 数字人事件桥 (唤醒词/音频) |
| **MQTT** | 1883 (可配) | 信令中继 + OTA 下发 |
| **UDP** | 可配 | 低延迟音频传输 |

### WebSocket 消息格式

```
文本消息:
{
  "type": "hello",
  "version": 3,
  "transport": "websocket",
  "audio_params": {"format": "opus", "sample_rate": 24000, "channels": 1}
}

二进制消息:
[Opus 编码音频帧, 60ms, 24000Hz, 单声道]
```

---

## 九、支持的 AI 服务总结

| 类别 | 数量 | 免费方案 |
|------|------|----------|
| ASR 语音识别 | 13 种 | FunASR (本地), Sherpa-ONNX (本地) |
| LLM 大模型 | 13 种 | 智谱 GLM-4-Flash, Gemini |
| TTS 语音合成 | 18 种 | EdgeTTS, 灵犀流式, CosyVoice |
| VLLM 视觉 | 2 种 | 智谱 GLM-4V-Flash |
| 记忆 | 6 种 | 本地短期记忆, mem0ai (每月1000次) |
| 意图 | 3 种 | function_call (免费) |
| VAD | 1 种 | SileroVAD (免费) |

---

## 十、两张表看清两个项目的关系

| | xiaozhi-esp32 (设备端) | xiaozhi-esp32-server (服务端) |
|------|------|------|
| 语言 | C++ | Python + Java + Vue |
| 运行位置 | ESP32 单片机 | PC / 服务器 / Docker |
| 功能 | 录音/播放/显示/LED/按键 | ASR/LLM/TTS 调度/设备管理 |
| 音频 | 采集 + Opus 编码 + 播放 | Opus 解码 + ASR + TTS + Opus 编码 |
| 唤醒词 | 本地离线运行 | - |
| VAD | 可本地/可服务端 | SileroVAD |
| MCP | 设备端执行 (音量/灯光等) | 云端执行 (天气/搜索/智能家居) |
| 配置 | sdkconfig + Kconfig | config.yaml |
| 构建 | ESP-IDF + CMake | pip / Maven / npm |
