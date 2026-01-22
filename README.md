# Seeed Studio XIAO ESP32-S3 Agora 语音对话工程（AI Agent Demo）

本仓库演示如何使用 **Seeed Studio XIAO ESP32-S3** 作为端侧语音设备，通过 **Agora** 实现实时语音链路，并对接 **AI Agent 服务端** 完成语音对话闭环。

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
