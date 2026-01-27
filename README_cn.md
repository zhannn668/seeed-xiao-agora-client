# Seeed Studio XIAO ESP32-S3 Agora 语音对话工程（AI Agent Demo）

*简体中文| [English](README.md)*

本仓库演示如何使用 **Seeed Studio reSpeaker XVF3800 (XIAO ESP32-S3)** 作为端侧语音设备，通过 **Agora** 实现实时语音链路，并对接 **AI Agent 服务端** 完成语音对话闭环。

> 重点内容在 `ai_agents/`：
> - **端侧（ESP32）**：[`ai_agents/esp32-client`](./ai_agents/esp32-client)
> - **服务端（AI Agent Server）**：[`ai_agents/server`](./ai_agents/server)

---

## 目录导览（你只需要看这两个）

```text
ai_agents/
  esp32-client/   # XIAO ESP32-S3 端侧：采集/播放音频 + Agora 连接 + 对话交互
  server/         # 服务端：AI Agent 编排/LLM/ASR/TTS 等（与端侧联动）
```

---

## 系统架构（简述）

1. XIAO ESP32-S3 连接网络并加入 Agora 房间
2. 端侧采集麦克风音频并推流（或上行数据）
3. `ai_agents/server` 接收音频/事件，完成 ASR → LLM → TTS（或其它 Agent 流程）
4. 服务端将回复音频/指令回传，端侧播放，实现实时语音对话

---

## 快速开始（推荐流程）

### 0. 准备条件
- 硬件：Seeed Studio XIAO ESP32-S3（以及麦克风/喇叭或对应扩展板）
- 网络：可访问 Agora 服务
- 软件：根据子目录要求安装（见下方两个链接）

### 1) 启动服务端（先跑 server）
进入：[`ai_agents/server`](./ai_agents/server)

按照该目录 README/文档：
- 配置环境变量（Agora / LLM / ASR / TTS 等）
- 启动服务

> 完成后你应该能看到服务端启动成功，并等待端侧连接或房间事件。

#### Windows（推荐 Docker Desktop）步骤（包含 Docker 简单安装配置）

> 适用：Windows 10/11（建议开启 WSL2）。以下命令建议在 **PowerShell** 或 **Windows Terminal** 中执行。

**A. 安装 / 配置 Docker Desktop（一次性）**
1. 下载并安装 Docker Desktop：
   https://www.docker.com/products/docker-desktop/
2. 安装过程中选择/启用 **Use WSL 2 instead of Hyper-V**（若有该选项）。
3. 安装完成后打开 Docker Desktop，等待右下角提示 **Docker is running**。
4.（可选但推荐）在 Docker Desktop：`Settings -> Resources -> WSL Integration`
   勾选你常用的 WSL 发行版（如 Ubuntu）。

**B. 克隆代码并准备环境变量**
```bash
git clone https://github.com/zhannn668/seeed-xiao-agora-client.git
cd seeed-xiao-agora-client
cd ai_agents
```

将示例环境变量复制为 `.env`（二选一）：

- PowerShell：
```powershell
Copy-Item .env.example .env
```

- 或 CMD：
```cmd
copy .env.example .env
```

然后用编辑器打开 `.env`，填入你的密钥/配置（Agora / LLM / ASR / TTS 等）：
```
......
# ------------------------------
# RTC
# ------------------------------

# Agora App ID
# Agora App Certificate(only required if enabled in the Agora Console)
AGORA_APP_ID=
AGORA_APP_CERTIFICATE=

# ------------------------------
# LLM
# ------------------------------

# Extension: openai_chatgpt
# OpenAI API key
OPENAI_API_BASE=https://api.openai.com/v1
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o
OPENAI_PROXY_URL=
# ------------------------------
# STT
# ------------------------------

# Extension: deepgram_asr_python
# Deepgram ASR key
DEEPGRAM_API_KEY=

# Azure ASR
AZURE_ASR_API_KEY=
AZURE_ASR_REGION=

# ------------------------------
# TTS
# ------------------------------
# Extension: cartesia_tts
# Cartesia TTS key
CARTESIA_API_KEY=
......

```

**如何获取 API Key：**

**Agora：**
1. 访问 https://console.agora.io/
2. 注册免费账号
3. 创建一个新项目（Project）
4. 复制 App ID 与 App Certificate

**Deepgram：**
1. 访问 https://console.deepgram.com/
2. 注册免费账号
3. 进入 API Keys 页面
4. 创建新的 API Key

**OpenAI：**
1. 访问 https://platform.openai.com/
2. 注册并绑定支付方式
3. 进入 API Keys 页面
4. 创建新的 Secret Key

**ElevenLabs：**
1. 访问 https://elevenlabs.io/
2. 注册免费账号
3. 进入 Profile → API Key
4. 复制 API Key
```
# Agora RTC Configuration (Required)
AGORA_APP_ID=your_agora_app_id_here
AGORA_APP_CERTIFICATE=your_agora_certificate_here

# Deepgram ASR Configuration (Required)
DEEPGRAM_API_KEY=your_deepgram_api_key_here

# OpenAI LLM Configuration (Required)
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-4o
OPENAI_PROXY_URL=  # Optional: leave empty if not using proxy

# ElevenLabs TTS Configuration (Required)
ELEVENLABS_TTS_KEY=your_elevenlabs_api_key_here

# Optional: Weather API (for weather tool functionality)
WEATHERAPI_API_KEY=your_weather_api_key_here
```
用编辑器打开ai_agents/agents/examples/voice-assistant/tenapp/property.json，根据你选择模型修改,可参考 https://docs.agora.io/en/conversational-ai/models/asr/overview：

```
......

"llm": {
    "url": "https://api.openai.com/v1/chat/completions",
    "api_key": "<your_llm_key>",
    "system_messages": [
      {
        "role": "system",
        "content": "You are a helpful chatbot."
      }
    ],
    "max_history": 32,
    "greeting_message": "Hello, how can I assist you",
    "failure_message": "Please hold on a second.",
    "params": {
      "model": "gpt-4o-mini"
    },
  }

"tts": {
  "vendor": "cartesia",
  "params": {
    "api_key": "<your_cartesia_key>",
    "model_id": "sonic-2",
    "voice": {
        "mode": "id",
        "id": "<voice_id>"
    },
    "output_format": {
        "container": "raw",
        "sample_rate": 16000
    },
    "language": "en"
  }
}

......
```
**C. 启动服务（Docker Compose）**
```bash
docker compose up -d
```

查看容器状态（可选）：
```bash
docker compose ps
```

**D. 进入容器并安装示例（Voice Assistant）**
> 注意：容器名可能会随 compose 配置不同而变化。下面以 `ten_agent_dev` 为例；如不一致，请以 `docker compose ps` 输出为准。

```bash
docker exec -it ten_agent_dev bash
```

进入容器后执行：
```bash
cd agents/examples/voice-assistant
task install
task run
```

**E. 验证服务端已启动**
- Docker Desktop 能看到相关容器 Running
- 或用日志查看（可选）：
```bash
docker compose logs -f
```

如需停止服务：
```bash
docker compose down
```

### 2) 烧录并运行 ESP32 端侧（再跑 esp32-client）
进入：[`ai_agents/esp32-client`](./ai_agents/esp32-client)

按照该目录 README/文档：
- 配置 Wi-Fi / Agora AppID/Token/Channel（或从服务端获取）
- 编译并烧录到 XIAO ESP32-S3
- 上电运行，观察串口日志

### 3) 验证
- 串口日志显示成功入会/连接
- 对设备说话后，服务端能收到音频/识别文本
- 设备能播放 AI 回复音频（或执行指令）

---

## 常见问题（FAQ）
- **Q：我应该先看哪个？**
  A：先看 `ai_agents/server`（把链路跑起来），再看 `ai_agents/esp32-client`（端侧接入）。
- **Q：我只想改端侧怎么办？**
  A：直接进入 `ai_agents/esp32-client`，服务端按默认示例启动即可。

---

## 参考与更多
- `ai_agents/` 之外的目录为上游框架/工具链/示例集合，本 Demo 以端侧 + 服务端联动为主。
