# ✈️ ATC Copilot — 模拟飞行陆空通话实时辅助

> 面向中文模拟飞友的本地 ATC 指令实时识别 / 解析 / 回读建议系统。
> **仅用于模拟飞行（VATSIM/IVAO/PilotEdge），不得用于任何真实航空运行。**

[![tests](https://img.shields.io/badge/tests-1700%2B%20passing-brightgreen)]() [![python](https://img.shields.io/badge/python-3.11-blue)]() [![Vue 3](https://img.shields.io/badge/Vue-3-42b883)]() [![License](https://img.shields.io/badge/license-personal--use-lightgrey)]()

---

## 1. 项目能力

实时监听 ATC 频道音频 → 自动识别英文管制指令 → 双轨解析 → 显示中英对照的「指令卡」与标准回读建议。

```
┌──────────┐  PCM ┌──────────┐ text ┌─────────────────────┐ JSON ┌──────────┐
│ 麦克风    │ ───▶ │ Whisper  │ ───▶│ DualTrackDispatcher │ ───▶│ Vue 前端  │
│ VB-Cable │      │  + VAD   │      │  ├ RuleEngine (5ms) │      │ 指令卡   │
└──────────┘      └──────────┘      │  └ DeepSeek (3s)    │      │ + 回读   │
                                    └─────────────────────┘      └──────────┘
```

| 能力 | 实现 |
|------|------|
| 英文 ATC 识别 | faster-whisper + ATC 微调模型（atc-small / atc-medium） |
| 语音活动检测 | Silero VAD（ONNX，CPU 实时） |
| 指令族解析 | 规则引擎覆盖 18+ 指令族（放行 / 滑行 / 起飞 / 高度 / 航向 / 速度 / 频率 / 推出 / 复飞 …） |
| LLM 兜底/校正 | DeepSeek V3 异步 HTTP，三种策略：`smart` / `always` / `off` |
| 呼号匹配 | 80+ 航司 ICAO 词典 + pypinyin 中文谐音 + Levenshtein 模糊数字 |
| 回读生成 | 模板引擎，中英双语 |
| 飞行员/管制员分类 | SpeakerClassifier（4 信号加权） |
| SimBrief 集成 | 一键拉取 OFP，注入到 LLM Prompt |
| 飞行阶段状态机 | FlightPhaseDisambiguator（地面 → 离场 → 巡航 → 进近 → 落地） |

---

## 2. 系统架构

```
ATC copilot/
├── server/              # FastAPI 后端
│   ├── app.py           # ASGI app、生命周期
│   ├── ws_handler.py    # WebSocket 协议（5 个 action）
│   ├── rest_api.py      # REST API（9 个端点）
│   └── session.py       # SessionManager（会话状态、热切换）
│
├── stt/                 # 语音识别
│   ├── whisper_engine.py   # PTT / 流式监听双模式
│   ├── vad.py              # Silero VAD 封装（含 RMS 回退）
│   └── common.py           # PyAudio 通用工具
│
├── parser/              # 指令解析
│   ├── pipeline.py         # Pipeline orchestrator
│   ├── rule_engine.py      # 规则引擎入口
│   ├── intent_families/    # 18+ 指令族子模块
│   ├── callsign_matcher.py # 呼号匹配
│   ├── normalizer.py       # 文本标准化（数字/字母/单位）
│   └── stt_corrections.py  # STT 谐音纠错词典
│
├── core/                # 业务核心
│   ├── dual_track.py       # 双轨调度器
│   ├── llm_authority.py    # DeepSeek 客户端（async）
│   ├── speaker_classifier.py
│   ├── utterance_assembler.py
│   ├── flight_phase.py     # 飞行阶段状态机
│   └── prompts.py          # LLM Prompt 模板
│
├── auth/                # API Key 管理（本地 JSON）
├── navdata/             # SimBrief / 导航数据
│
├── frontend/            # Vue 3 + Vite 前端
│   └── src/
│       ├── views/          # Dashboard / SettingsView / SetupWizard
│       ├── components/     # InstructionCard / PttButton / FlightInfo …
│       ├── composables/    # useATCWebSocket / usePTT …
│       └── stores/         # Pinia: session / atcStore
│
├── tests/               # pytest 单元 + 黑盒回归（1700+ 用例）
├── training/            # 模型微调脚本（可选）
├── models/              # ATC Whisper 模型权重
├── scripts/diag/        # 诊断/一次性脚本（不参与运行时）
└── docs/                # 设计文档
```

### 双轨解析

```
                      ┌─► RuleEngine.parse()  ─► 即时结果（pending）
ATC text ─► Dispatcher┤
                      └─► LLMAuthority.parse() ─► 最终结果（confirmed）
                                                  覆盖 / 修正 / 拒绝
```

LLM 策略：
- `smart`：仅在规则引擎置信度低或留空时才调 DeepSeek（默认）
- `always`：每条都送 DeepSeek 二次确认
- `off`：完全离线，仅用规则引擎

---

## 3. 安装运行

### 3.1 环境要求
- Windows 10 / 11（Linux/macOS 未测试）
- Python **3.11**（推荐 venv）
- Node.js 18+
- 麦克风 或 VB-Cable（用于把模拟器/VATSIM 客户端音频路由到 ATC Copilot）

### 3.2 一键安装

```powershell
# 仓库根
.\setup.bat            # 创建 .venv + 装 Python 依赖 + 装 npm 依赖
```

或手动：

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

cd frontend
npm install
```

### 3.3 GPU 加速（可选）

Whisper 默认在 CPU 上推理，若有 NVIDIA 显卡：

```powershell
pip install torch --index-url https://download.pytorch.org/whl/cu121
```

`stt/whisper_engine.py` 会自动探测 CUDA 并切换到 `float16`。

### 3.4 启动

开发模式（前后端分离 + 热重载）：

```powershell
.\start_dev.ps1
# 后端 http://127.0.0.1:8731
# 前端 http://127.0.0.1:5173
```

或快捷启动：双击 [启动.bat](启动.bat)。

生产模式（前端打包后由后端托管）：

```powershell
cd frontend; npx vite build
cd ..; .venv\Scripts\python.exe -m uvicorn server.app:app --host 127.0.0.1 --port 8731
# 浏览器打开 http://127.0.0.1:8731
```

### 3.5 首次配置

启动后首次进入会自动跳到「设置向导」（SetupWizard）：

1. **硬件检测** — 自动选择推荐 Whisper 模型
2. **DeepSeek API Key** — 用于 LLM 兜底；不填则强制 `llm_strategy=off`
3. **音频设备** — 选择麦克风（PTT）+ ATC 监听设备（VB-Cable 等）
4. **呼号 + SimBrief 用户名** — 用于呼号匹配 / 飞行计划上下文
5. 完成 → 进入 Dashboard

向导任何一步都可以跳过，后续在「设置」页修改。

---

## 4. 配置文件

| 文件 | 内容 | 说明 |
|------|------|------|
| `config/api_keys.json` | DeepSeek API Key | **明文 JSON 本地存储**，已加入 `.gitignore`；如需上传仓库请保留 `.example` |
| `localStorage`（前端） | callsign / whisperModel / micDeviceIndex / atcDeviceIndex / pttKeyCode / llm_strategy | 浏览器本地，不出本机 |
| `models/atc-*` | Whisper 模型权重 | 启动后自动加载，建议放 SSD |

> ⚠️ API Key 当前为明文存储。如打算分享设备或长期运行，建议使用系统钥匙串或操作系统级权限隔离。

---

## 5. 前后端通信

### REST API（详见 [server/rest_api.py](server/rest_api.py)）

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/session` | 当前会话快照 |
| POST | `/api/session/configure` | 更新会话（callsign / lang / whisper_model / mic_device_index / atc_device_index / simbrief_username / llm_strategy） |
| POST | `/api/session/load-flight-plan` | 拉取 SimBrief 计划 |
| GET / POST | `/api/ai/config` | DeepSeek API key 读写（写后强制重建 dispatcher） |
| POST | `/api/ai/test` | DeepSeek 连通性测试 |
| GET | `/api/devices` | 列出 PyAudio 设备 |
| GET | `/api/hardware-detect` | CPU / GPU / 内存 + 推荐模型 |

### WebSocket（详见 [docs/websocket_protocol.md](docs/websocket_protocol.md)）

前端 → 后端 actions：`configure` / `start_listen` / `stop_listen` / `start_atc_listen` / `stop_atc_listen` / `ping`

后端 → 前端 events：`session_updated` / `stt_event` / `atc_stt_event` / `pending` / `confirmed` / `llm_error` / `my_readback` / `flight_plan` / `phase_updated` / `audio_level` / `error`

---

## 6. 测试

```powershell
.venv\Scripts\python.exe -m pytest tests/ -q
```

完整套件 1700+ 用例，平均跑时 ~30s。
单独跑回归：

```powershell
.venv\Scripts\python.exe -m pytest tests/test_rule_engine.py tests/test_callsign_matcher.py -q
```

诊断/一次性脚本统一放在 [scripts/diag/](scripts/diag/)，不参与 CI。

---

## 7. 目录约定

| 路径 | 含义 |
|------|------|
| `_*.py` | 历史诊断脚本（已迁入 `scripts/diag/`） |
| `temp_*.vue` / `tmp_*.txt` | 一次性产物，**禁止提交** |
| `backend.log` | 运行时日志，**禁止提交** |
| `models/` | 大文件，使用 `.gitignore` 忽略，独立分发 |

---

## 8. 已知限制

- **仅模拟飞行用途**：识别率 ~85%，且不保证零漏识，不能替代飞行员对真实指令的判断。
- **仅英文 ATC**：中文管制识别已停用（讯飞依赖移除）。
- **API Key 明文存储**：见 §4 注意事项。
- **首次启动 Whisper 模型加载** ~5–15s（CPU），建议保持后台预热。

---

## 9. 致谢与协议

- 基于 [faster-whisper](https://github.com/SYSTRAN/faster-whisper)、[silero-vad](https://github.com/snakers4/silero-vad)、[FastAPI](https://fastapi.tiangolo.com/)、[Vue 3](https://vuejs.org/) 构建
- ATC Whisper 模型微调数据来自 [ATCO2 / ATCOSIM](https://www.atco2.org/) 公开语料
- 本项目仅供个人学习与模拟飞行使用，禁止任何商业用途与真实航空运行用途














