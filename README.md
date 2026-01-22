# ESP32-S3（XIAO）Agora 语音对话工程（main 组件）

本目录是一套可直接编译的 **ESP-IDF main 组件代码**，实现设备端与 **Agora RTC** 的音频上下行，并通过 **TEN AI Agent 服务**完成：
- 获取 AppID / Token（`/token/generate`）
- 启动/停止 Agent（`/start`、`/stop`）
- KeepAlive（`/ping`）

> 代码中包含音频链路（ESP-ADF pipeline）、Agora RTC 发送/接收、以及基于 XVF3800 的按键监控（SET/MUTE）。

---

## 目录文件一览（按实际代码）

- `llm_main.c`  
  程序入口 `app_main()`：WiFi → AIC3104 初始化 → `ai_agent_generate()` → 创建 RTC → 提示按键启动 Agent → 定时 ping + 打印内存。

- `app_config.h`  
  业务配置（TEN AI Agent 服务地址、Graph、Channel、UID、AppID 等）。

- `common.h`  
  全局状态 `g_app`、音频编解码宏选择（G711U/G722）以及采样率/帧长等参数。

- `ai_agent.c / ai_agent.h`  
  HTTP 客户端：请求 TEN AI Agent 服务（`token/generate`、`start`、`stop`、`ping`），解析返回并写入 `g_app.app_id / g_app.token`。

- `rtc_proc.c / rtc_proc.h`  
  Agora RTC 封装：join channel、事件回调、发送音频帧 `send_rtc_audio_frame()`、接收混音音频并播放 `playback_stream_write()`。

- `audio_proc.c / audio_proc.h`  
  ESP-ADF 音频 pipeline：I2S 录音 → AEC（`algo_stream`）→ raw → 发送到 RTC；下行音频写入 raw → I2S 播放。  
  备注：`audio_set_volume()` 目前是占位打印（未实现 AIC3104 音量寄存器控制）。

- `aic3104_ng.c / aic3104_ng.h`  
  AIC3104 Codec I2C 最小驱动（探测/写寄存器/默认初始化）。

- `xvf3800.c / xvf3800.h`  
  XVF3800（XMOS）I2C 控制与按键轮询任务：  
  - SET → `ai_agent_start()`  
  - MUTE → `ai_agent_stop()`

- `video_proc.c / video_proc.h`（已保留但默认不启用）  
  CMakeLists 里已注释 `video_proc.c`，且 `llm_main.c` 中 `start_video_proc()` 被注释。

---

## 依赖与工程放置方式（按本目录代码要求）

### 必需依赖（从 `CMakeLists.txt`/`idf_component.yml` 得出）
- ESP-IDF（`idf_component.yml` 标注 `>=4.1.0`，建议用你当前项目一致版本）
- ESP-ADF 相关组件（`audio_hal / audio_pipeline / audio_stream / audio_board / esp_peripherals / esp-adf-libs / input_key_service`）
- `components/agora_iot_sdk`（本目录通过 `idf_component.yml` 以相对路径引用）
- Managed component：`espressif/esp32-camera`（在 `idf_component.yml` 中声明）

### 推荐目录结构
确保工程根目录类似如下（重点是 `components/agora_iot_sdk` 存在）：

```text
<project_root>/
├─ main/                  # 放本目录所有 .c/.h/CMakeLists.txt/Kconfig/idf_component.yml
└─ components/
   └─ agora_iot_sdk/       # 必须存在（与 idf_component.yml 的 path 对应）
```

> ESP-ADF 组件可通过：
> - 复制 ADF 的 `components/` 到 `<project_root>/components/`  
> - 或在工程顶层 `CMakeLists.txt` 通过 `EXTRA_COMPONENT_DIRS` 指向 ADF `components/`（按你现有工程方式即可）

---

## 配置项

### 1）修改 `app_config.h`
你最常需要改这些：

- `TENAI_AGENT_URL`：TEN AI Agent 服务地址（默认 `http://192.168.9.26:8080`）
- `AI_AGENT_NAME`：`request_id`
- `AI_AGENT_CHANNEL_NAME`：频道名
- `AI_AGENT_USER_ID`：设备 UID（`user_uid/uid`）
- `CONFIG_GRAPH_OPENAI`：当前启用的 graph 选择（代码里默认打开）
- `AGORA_APP_ID`：写在宏里，但**实际运行会以 `/token/generate` 返回的 appId 覆盖 `g_app.app_id`**

### 2）Menuconfig 配 WiFi（来自 `Kconfig.projbuild`）
执行：

```bash
idf.py menuconfig
```

路径：

- `Agora Demo for ESP32`
  - `WiFi SSID`（生成宏 `CONFIG_WIFI_SSID`）
  - `WiFi Password`（生成宏 `CONFIG_WIFI_PASSWORD`）
  - `WiFi listen interval`（`CONFIG_EXAMPLE_WIFI_LISTEN_INTERVAL`）
  - CPU 频率选项（若开启 PM）

---

## 编译与烧录

```bash
idf.py set-target esp32s3
idf.py build
idf.py -p <PORT> flash monitor
```

---

## 运行流程（和代码一致）

上电后 `llm_main.c` 的关键流程：

1. 初始化 NVS
2. 连接 WiFi（使用 `CONFIG_WIFI_SSID / CONFIG_WIFI_PASSWORD`）
3. 初始化 AIC3104（I2C：`I2C_NUM_0`，SDA=`GPIO5`，SCL=`GPIO6`，100kHz）
4. 调用 `ai_agent_generate()` → 请求 `TENAI_AGENT_URL/token/generate`  
   - 成功后写入 `g_app.app_id`、`g_app.token` 并置位 `g_app.b_ai_agent_generated=true`
5. 创建并加入 Agora RTC：`agora_rtc_proc_create(NULL, AI_AGENT_USER_ID)`
6. 提示按键启动 Agent（由 `xvf3800` 监控任务触发）  
   - SET → `ai_agent_start()`  
   - MUTE → `ai_agent_stop()`
7. 主循环每 10s 打印内存；若 Agent 已启动则 `ai_agent_ping()` keepalive

---

## TEN AI Agent 服务接口（按 `ai_agent.c` 实际请求）

- `POST {TENAI_AGENT_URL}/token/generate`

```json
{
  "request_id": "tenai0125-11",
  "uid": 12345,
  "channel_name": "test_channel_12345"
}
```

- `POST {TENAI_AGENT_URL}/start`

```json
{
  "request_id": "tenai0125-11",
  "channel_name": "test_channel_12345",
  "user_uid": 12345,
  "graph_name": "voice_assistant"
}
```

- `POST {TENAI_AGENT_URL}/stop`

```json
{
  "request_id": "tenai0125-11",
  "channel_name": "test_channel_12345"
}
```

- `POST {TENAI_AGENT_URL}/ping`

```json
{
  "request_id": "tenai0125-11",
  "channel_name": "test_channel_12345"
}
```

---

## 备注（按当前代码事实）

- `audio_set_volume()` 目前未对 AIC3104 寄存器做实际控制，仅打印提示（见 `audio_proc.c`）。
- `video_proc.c` 已保留但默认不参与编译（`CMakeLists.txt` 注释、`llm_main.c` 调用也注释）。
- XVF3800 I2C 地址在头文件中定义为 `0x2C`（`xvf3800.h`），AIC3104 I2C 地址为 `0x18`（`aic3104_ng.h`）。
