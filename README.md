# Compagnon2 — Firmware ESP32-S3 Waveshare AMOLED 2.16

> - **Hardware** : Waveshare ESP32-S3-Touch-AMOLED-2.16 (16MB Flash, 8MB PSRAM)
> - **Display** : CO5300 AMOLED 466×466, tactile CST92xx, Arduino GFX / LVGL 8.x
> - **LLM** : Groq API (clé gratuite, ultra-rapide, 0€/mois)
> - **Architecture** : Voice I/O (xiaozhi) + Agent Brain (mimiclaw) + LVGL UI
> - **Arduino IDE** v3.3.5 (Board : esp32 by Espressif 3.3.5)

---

## Matériel requis

| Composant | Référence |
|---|---|
| Microcontrôleur | Waveshare ESP32-S3-Touch-AMOLED-2.16 |
| Flash | 16MB (onboard) |
| PSRAM | 8MB Octal (onboard) |
| Display | CO5300 AMOLED 466×466 px via QSPI |
| Touch | CST92xx I2C (SDA=15, SCL=14) |
| Micro | ES7210 I2S quad-mic |
| Speaker | ES8311 I2S codec |
| IMU | QMI8658 I2C |
| PMIC | AXP2101 I2C |

---

## Architecture du firmware

```
┌─────────────────────────────────────────────────┐
│              COMPAGNON2 FIRMWARE                │
├──────────────────┬──────────────────────────────┤
│  Voice I/O Layer │  LVGL UI Layer               │
│  - Wake word     │  - Home screen               │
│  - ASR streaming │  - Agent list & switch       │
│  - TTS playback  │  - Chat history              │
│  - ES7210 mic    │  - Config / WiFi / API key   │
│  - ES8311 spk    │  - Skills browser            │
├──────────────────┴──────────────────────────────┤
│              Bridge Layer                       │
│  (voice text ↔ agent commands)                  │
├─────────────────────────────────────────────────┤
│             Agent Brain (mimiclaw)              │
│  - Groq LLM API (llama3-8b-8192)                │
│  - ReAct loop + tool calling                    │
│  - Multi-agent profiles (Bourse, Travail…)      │
│  - Memory L0-L4 sur LittleFS                    │
│  - Auto-cristallisation de skills               │
│  - Cron scheduler                               │
│  - Lua scripting                                │
├─────────────────────────────────────────────────┤
│              PWA Config App                     │
│  (serveur HTTP embarqué sur ESP32)              │
│  - WiFi setup, Groq API key                     │
│  - Agent profiles editor                        │
│  - Skills manager                               │
└─────────────────────────────────────────────────┘
```

---

## Installation rapide

### 1. Prérequis

- **Arduino IDE 2.x** avec board package `esp32 by Espressif **3.3.5**`
- **Bibliothèques** (copier dans `~/Arduino/libraries/`) :
  - `lvgl` (v8.3.x — fournie dans `libraries/lvgl/`)
  - `GFX_Library_for_Arduino` (fournie dans `libraries/GFX_Library_for_Arduino/`)
  - `SensorLib` (TouchDrvCSTXXX, SensorQMI8658)
  - `XPowersLib` (AXP2101)

### 2. Configuration

Copier `src/config/secrets.h.template` en `src/config/secrets.h` et remplir :

```cpp
#define WIFI_SSID     "votre_ssid"
#define WIFI_PASSWORD "votre_mdp"
#define GROQ_API_KEY  "gsk_xxxxxxxxxxxxxxxxxxxx"
```

> **Clé Groq gratuite** : https://console.groq.com — modèle `llama3-8b-8192`

### 3. Paramètres Arduino IDE

| Paramètre | Valeur |
|---|---|
| Board | ESP32S3 Dev Module |
| Flash Size | 16MB |
| Partition Scheme | Custom → `partitions_16mb.csv` |
| PSRAM | OPI PSRAM |
| USB Mode | Hardware CDC and JTAG |
| Upload Speed | 921600 |

### 4. Flasher

Ouvrir `Compagnon2.ino` dans Arduino IDE, sélectionner le bon port COM, cliquer Upload.

---

## PWA de configuration

Après le premier démarrage, connectez-vous au WiFi AP `Compagnon2-Setup` (mdp : `compagnon`) puis allez sur `http://192.168.4.1`. Vous pouvez y configurer :

- Réseau WiFi et clé Groq
- Créer/modifier/supprimer des profils d'agents
- Voir et gérer les skills auto-cristallisés
- Consulter l'historique des sessions

Une fois configuré et connecté au WiFi principal, la PWA est accessible sur `http://<ip_esp>/`.

---

## Structure des fichiers

```
Compagnon2/
├── Compagnon2.ino          # Sketch principal Arduino
├── partitions_16mb.csv     # Table de partitions 16MB
├── src/
│   ├── config/
│   │   ├── pin_config.h    # Pins Waveshare AMOLED 2.16
│   │   ├── board_config.h  # Config display/touch
│   │   └── secrets.h       # WiFi + Groq API key (gitignored)
│   ├── display/
│   │   ├── display_init.h  # Init CO5300 + LVGL
│   │   └── ui/
│   │       ├── ui_home.h   # Écran principal
│   │       ├── ui_chat.h   # Historique conversation
│   │       ├── ui_agents.h # Gestionnaire d'agents
│   │       └── ui_config.h # Paramètres
│   ├── agent/
│   │   ├── agent_manager.h # Multi-agent profile switcher
│   │   ├── groq_llm.h      # Client HTTP Groq API
│   │   ├── react_loop.h    # ReAct reasoning loop
│   │   ├── tool_registry.h # Registre de tools
│   │   └── memory.h        # LittleFS memory L0-L4
│   ├── voice/
│   │   ├── voice_init.h    # ES7210 + ES8311 I2S
│   │   └── wake_word.h     # Wake word detection
│   └── pwa/
│       └── pwa_server.h    # Serveur HTTP + PWA
├── data/                   # LittleFS data (agents, skills, memory)
│   ├── agents/
│   │   ├── general/SOUL.md
│   │   ├── jardinier/SOUL.md
│   │   └── travail/SOUL.md
│   ├── skills/
│   └── memory/
│       └── MEMORY.md
└── libraries/              # Libs locales (copier dans Arduino/libraries)
    ├── lvgl/
    ├── GFX_Library_for_Arduino/
    └── lv_conf.h
```

---

## Roadmap v0 → v1

- [x] Base LVGL + tactile CO5300/CST92xx
- [x] Groq LLM client (WiFi)
- [x] ReAct loop + tool calling basique
- [x] Multi-agent profiles (Jardinier, Travail, Général)
- [x] Memory LittleFS L0-L4
- [x] PWA config embarquée
- [ ] Wake word ESP-SR (v0.5)
- [ ] Streaming ASR (v0.5)
- [ ] Auto-cristallisation skills (v1)
- [ ] BLE fallback relay (v1)
- [ ] Cron scheduler (v1)
- [ ] Lua scripting (v1)
