# Seeed Studio XIAO ESP32-S3 Agora Voice Conversation Project (AI Agent Demo)

*[简体中文](README_cn.md) | English*

This repository demonstrates how to use **Seeed Studio reSpeaker XVF3800 (XIAO ESP32-S3)** as an edge voice device, build a real-time voice link via **Agora**, and connect to an **AI Agent backend service** to complete a full voice conversation loop.

> The key content is in `ai_agents/`:
> - **Edge (ESP32)**: [`ai_agents/esp32-client`](./ai_agents/esp32-client)
> - **Backend (AI Agent Server)**: [`ai_agents/server`](./ai_agents/server)

---

## Directory Guide (You only need to read these two)

```text
ai_agents/
  esp32-client/   # XIAO ESP32-S3 edge side: audio capture/playback + Agora connection + conversation interaction
  server/         # Backend: AI Agent orchestration / LLM / ASR / TTS, etc. (works together with the edge side)
```

---

## System Architecture (Brief)

1. The XIAO ESP32-S3 connects to the network and joins an Agora room
2. The edge side captures microphone audio and publishes it (or uploads data)
3. `ai_agents/server` receives audio/events and performs ASR → LLM → TTS (or other Agent flows)
4. The backend sends the response audio/commands back, and the edge side plays it, enabling real-time voice conversation

---

## Quick Start (Recommended Workflow)

### 0. Prerequisites
- Hardware: Seeed Studio XIAO ESP32-S3 (plus mic/speaker or the corresponding expansion board)
- Network: Able to access Agora services
- Software: Install according to the requirements in the subdirectories (see the two links below)

### 1) Start the backend first (run server first)
Go to: [`ai_agents/server`](./ai_agents/server)

Follow the README/docs in that directory:
- Configure environment variables (Agora / LLM / ASR / TTS, etc.)
- Start the service

> After that, you should see the backend service start successfully and wait for edge connections or room events.

#### Windows (Docker Desktop recommended) Steps (including a simple Docker installation/config)

> Applies to: Windows 10/11 (WSL2 is recommended). The following commands are recommended to run in **PowerShell** or **Windows Terminal**.

**A. Install / Configure Docker Desktop (one-time)**
1. Download and install Docker Desktop:
   https://www.docker.com/products/docker-desktop/
2. During installation, select/enable **Use WSL 2 instead of Hyper-V** (if available).
3. After installation, open Docker Desktop and wait until the tray shows **Docker is running**.
4. (Optional but recommended) In Docker Desktop: `Settings -> Resources -> WSL Integration`
   Enable your commonly used WSL distribution (e.g., Ubuntu).

**B. Clone the repo and prepare environment variables**
```bash
git clone https://github.com/zhannn668/seeed-xiao-agora-client.git
cd seeed-xiao-agora-client
cd ai_agents
```

Copy the example environment variables to `.env` (choose one):

- PowerShell:
```powershell
Copy-Item .env.example .env
```

- Or CMD:
```cmd
copy .env.example .env
```

Then open `.env` in an editor and fill in your keys/config (Agora / LLM / ASR / TTS, etc.):

**How to get API Keys:**

**Agora:**
1. Visit https://console.agora.io/
2. Register a free account
3. Create a new project (Project)
4. Copy the App ID and App Certificate

**Deepgram:**
1. Visit https://console.deepgram.com/
2. Register a free account
3. Go to the API Keys page
4. Create a new API Key

**OpenAI:**
1. Visit https://platform.openai.com/
2. Register and add a payment method
3. Go to the API Keys page
4. Create a new Secret Key

**ElevenLabs:**
1. Visit https://elevenlabs.io/
2. Register a free account
3. Go to Profile → API Key
4. Copy the API Key
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
WEATHERAPI_API_KEY=your_weatherapi_api_key_here
```

Open `ai_agents/agents/examples/voice-assistant/tenapp/property.json` in an editor, and modify it according to the model you choose. You can refer to: https://docs.agora.io/en/conversational-ai/models/asr/overview

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

**C. Start the service (Docker Compose)**
```bash
docker compose up -d
```

Check container status (optional):
```bash
docker compose ps
```

**D. Enter the container and install the sample (Voice Assistant)**
> Note: The container name may vary depending on the compose configuration. The example below uses `ten_agent_dev`. If yours differs, use the output of `docker compose ps`.

```bash
docker exec -it ten_agent_dev bash
```

After entering the container, run:
```bash
cd agents/examples/voice-assistant
task install
task run
```

**E. Verify the backend is running**
- You can see the relevant containers are Running in Docker Desktop
- Or view logs (optional):
```bash
docker compose logs -f
```

To stop the service:
```bash
docker compose down
```

### 2) Flash and run the ESP32 edge side (then run esp32-client)
Go to: [`ai_agents/esp32-client`](./ai_agents/esp32-client)

Follow the README/docs in that directory:
- Configure Wi-Fi / Agora AppID/Token/Channel (or obtain them from the backend)
- Build and flash to the XIAO ESP32-S3
- Power on and observe the serial logs

### 3) Validation
- Serial logs show the device successfully joined the channel / connected
- After you speak to the device, the backend receives audio / recognized text
- The device can play back the AI response audio (or execute commands)

---

## FAQ
- **Q: Which one should I read first?**
  A: Read `ai_agents/server` first (get the whole pipeline running), then read `ai_agents/esp32-client` (connect the edge device).
- **Q: What if I only want to modify the edge side?**
  A: Go directly to `ai_agents/esp32-client`. The backend can be started with the default sample.

---

## References & More
- Directories outside `ai_agents/` are upstream frameworks/toolchains/sample collections. This demo mainly focuses on the edge + backend linkage.
