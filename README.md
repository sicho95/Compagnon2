# Compagnon2 — Nestor v2 Firmware

> Basé sur [xiaoclaw](https://github.com/beancookie/xiaoclaw) (MIT) · Adapté pour Waveshare AMOLED 2.16" · Groq LLM · Agents auto-apprenants

## Architecture

```
[Wake Word ESP-SR] → [ASR streaming Groq Whisper]
       ↓ texte
[Bridge Layer]
       ↓
[Agent Router] ← SOUL.md (personnalité Nestor)
    ├── Agent Orchestrateur  (planification méta)
    ├── Agent Jardinier      (nettoyage / maintenance agents)
    ├── Agent Fabrique       (création agents à la volée)
    ├── Agent Web Analyst    (recherche + résumé)
    └── Agents customs       (auto-cristallisés)
       ↓
[ReAct Loop] → [Tool Calling] → [MCP local/distant]
       ↓
[Mémoire FATFS L0-L4] + [Auto-cristallisation skills]
       ↓
[Groq PlayAI TTS streaming] → [ES8311 Speaker]
       ↑
[LVGL UI 460×470 tactile CST9220] ← différenciateur Nestor

Fallbacks :
  WiFi absent → BLE LLM_RELAY → smartphone → Groq cloud
  LLM cloud KO → modèle Ollama local réseau (CONFIG)
```

## Hardware cible

| Composant | Détail |
|---|---|
| SoC | ESP32-S3 (Xtensa LX7 dual-core 240 MHz) |
| Flash | 16 MB |
| PSRAM | 8 MB OPI 80 MHz |
| Écran | Waveshare AMOLED 2.16" — 460×470 px |
| Touch | CST9220 I2C |
| PMU | AXP2101 |
| RTC | PCF85063 |
| IMU | QMI8658 |
| Codec audio | ES8311 (DAC/SPK) |
| Micros | ×2 I2S (intégrés) |
| MicroSD | SDMMC ou SPI |
| Batterie | LiPo 1000 mAh |

## Différences clés vs xiaoclaw upstream

| Feature | xiaoclaw upstream | Compagnon2 |
|---|---|---|
| LLM backend | Claude/OpenAI API | **Groq** (llama-3.3-70b, gratuit generous tier) |
| ASR | Serveur xiaozhi cloud | **Groq Whisper** (streaming) |
| TTS | Serveur xiaozhi cloud | **Groq PlayAI TTS** (WAV) |
| Board | 70+ boards generiques | **Waveshare AMOLED 2.16"** (460×470, CO5300) |
| Écran résolution | 480×480 (S3-BOX etc.) | **460×470** AMOLED |
| Agents par défaut | Aucun | 9 agents Nestor prédéfinis (depuis SPEC Compagnon) |
| BLE relay LLM | Non | **Oui** (fallback WiFi absent) |
| Agent Jardinier | Non | **Oui** (nettoyage / maintenance agents) |
| Agent Fabrique | Non | **Oui** (création agents à la volée par voix) |
| Rotation UI | 480×480 | **460×470** + rotation adaptée |
| PCF85063 RTC | Non | **Oui** (heure précise hors NTP) |

## Structure du projet

```
Compagnon2/
├── README.md
├── SPEC.md                    # Spécification complète
├── sdkconfig.defaults.esp32s3 # Config IDF pour 16MB flash / 8MB OPI PSRAM
├── CMakeLists.txt
├── partitions/
│   └── compagnon2.csv         # Table partitions 16MB
├── main/
│   ├── mimi/                  # Agent brain (hérité xiaoclaw/mimiclaw)
│   │   ├── agent/             # ReAct loop, runner, hooks, checkpoint
│   │   ├── memory/            # L0-L4 FATFS
│   │   ├── skills/            # Loader + cristallisation
│   │   ├── tools/             # Registry tools
│   │   ├── llm/               # LLM proxy → Groq
│   │   ├── cron/              # Scheduler
│   │   └── channels/          # BLE relay channel (NOUVEAU)
│   ├── audio/                 # Voice I/O (hérité xiaozhi)
│   │   ├── wake_words/        # ESP-SR
│   │   ├── asr_groq.cc/.h     # NOUVEAU : ASR Groq Whisper
│   │   └── tts_groq.cc/.h     # NOUVEAU : TTS Groq PlayAI
│   ├── bridge/                # Bridge voice ↔ agent
│   ├── display/               # LVGL + CO5300
│   │   └── waveshare_2_16/    # Board AMOLED 2.16" 460×470
│   ├── hal/                   # HAL hérité Compagnon
│   │   ├── display.cc/.h      # CO5300 QSPI
│   │   ├── touch.cc/.h        # CST9220
│   │   ├── pmu.cc/.h          # AXP2101
│   │   ├── imu.cc/.h          # QMI8658
│   │   └── rtc.cc/.h          # PCF85063 (NOUVEAU)
│   ├── net/
│   │   ├── ble_mgr.cc/.h      # BLE multi-chars (GPS + LLM_RELAY + AGENT_SYNC)
│   │   ├── ble_protocol.cc/.h # Protocole haut niveau
│   │   └── wifi_mgr.cc/.h     # WiFiManager non-bloquant
│   ├── ui/
│   │   ├── launcher.cc/.h     # Carousel LVGL 460×470
│   │   ├── status_bar.cc/.h   # Barre état (NTP/RTC + batterie)
│   │   ├── agent_ui.cc/.h     # NOUVEAU : vue agents LVGL
│   │   └── voice_ui.cc/.h     # NOUVEAU : feedback vocal (waveform)
│   ├── config/
│   │   ├── pin_config.h       # Broches Waveshare AMOLED 2.16"
│   │   ├── secrets.template.h # Template (gitignorée : secrets.h)
│   │   └── lv_conf.h          # Config LVGL 9
│   └── main.cc                # Entry point IDF
└── fatfs_data/                # FATFS flashé dans /fatfs
    ├── config/
    │   ├── SOUL.md             # Personnalité Nestor
    │   └── USER.md             # Infos utilisateur
    ├── memory/
    │   ├── MEMORY.md
    │   └── skill_index.json
    ├── skills/
    │   ├── gardener/SKILL.md   # Agent Jardinier
    │   ├── factory/SKILL.md    # Agent Fabrique
    │   └── auto/               # Auto-cristallisés
    └── agents/
        └── default_agents.json # 9 agents Nestor initiaux
```

## Quick Start

### Prérequis
- ESP-IDF v5.5+
- Python 3.10+
- CMake 3.16+

### Configuration

```bash
cp main/config/secrets.template.h main/config/secrets.h
# Éditer secrets.h : renseigner GROQ_API_KEY

idf.py set-target esp32s3
idf.py menuconfig  # → Compagnon2 → Secret Configuration
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### Clés requises

| Clé | Service | Obligatoire |
|---|---|---|
| `GROQ_API_KEY` | LLM (Llama 3.3 70B) + ASR Whisper + TTS PlayAI | **Oui** |
| `METEO_API_KEY` | meteo-concept.com (météo ESP32) | Non |
| `WIFI_SSID / PASS` | WiFi primaire | Non (portail captif si absent) |

**Groq free tier** : ~30 req/min LLM + Whisper + TTS gratuits — suffisant pour usage personnel.

## Licence

MIT — basé sur xiaoclaw (MIT) + xiaozhi-esp32 (MIT)
