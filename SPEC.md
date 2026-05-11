# Compagnon2 — Spécification Technique

> Version **v1.0** — mai 2026  
> Basé sur xiaoclaw (xiaozhi voice I/O + mimiclaw agent brain)  
> Adapté pour Nestor : Waveshare AMOLED 2.16", Groq, agents spécialisés

---

## 1. Vision

Compagnon2 est le firmware ESP32-S3 d'un assistant personnel autonome **Nestor** :
- **Parle** via réveil vocal hors-ligne (ESP-SR)
- **Comprend** via Groq Whisper ASR streaming
- **Réfléchit** via ReAct loop (agent brain mimiclaw) + Groq LLM
- **Mémorise** via hiérarchie FATFS L0-L4
- **Apprend** via auto-cristallisation de skills
- **Crée** de nouveaux agents à la volée (Agent Fabrique)
- **Entretient** sa base d'agents (Agent Jardinier = nettoyage/optimisation)
- **Survit** sans WiFi via BLE relay → smartphone → Groq cloud
- **Affiche** sur LVGL 460×470 AMOLED tactile

---

## 2. Hardware — Waveshare AMOLED 2.16"

### 2.1 Broches

| Signal | GPIO | Notes |
|---|---|---|
| LCD_CS | 12 | Chip select QSPI |
| LCD_SCLK | 38 | Horloge QSPI |
| LCD_SDIO0-3 | 4,5,6,7 | Bus QSPI |
| LCD_RESET / TOUCH_RES | 2 | **Partagé** |
| IIC_SDA | 15 | I2C (touch, PMU, IMU, RTC) |
| IIC_SCL | 14 | I2C horloge |
| AXP_INT | 13 | IRQ PMU (FALLING, IRAM_ATTR) |
| BTN_LEFT | 18 | App précédente (actif LOW) |
| BTN_RIGHT | 0 | App suivante / ouvrir |
| ES8311 DOUT | 8 | I2S SPK data out |
| PA_EN | 46 | Ampli SPK |
| MIC_BCLK | 9 | I2S mic bit clock |
| MIC_LRCK | 45 | I2S mic word select |
| MIC_DIN | 10 | I2S mic data in |
| MIC_MCLK | 16 | I2S mic master clock |
| SD_CLK | 39 | SDMMC clk |
| SD_CMD | 40 | SDMMC cmd |
| SD_D0 | 41 | SDMMC data |

### 2.2 Paramètres compilation

| Paramètre | Valeur |
|---|---|
| Board | ESP32-S3 |
| Flash | 16 MB |
| PSRAM | OPI 8 MB 80 MHz |
| Partition scheme | compagnon2.csv (custom) |
| CPU | 240 MHz |
| IDF | v5.5+ |

---

## 3. Architecture Firmware

### 3.1 Couches

```
┌─────────────────────────────────────────────────────┐
│                  UI Layer (LVGL 9)                  │
│  launcher · status_bar · agent_ui · voice_ui        │
├─────────────────────────────────────────────────────┤
│               Application Layer                     │
│  app_nestor · app_radars · app_bourse · app_meteo   │
├─────────────────────────────────────────────────────┤
│            Voice I/O (xiaozhi heritage)             │
│  wake_word(ESP-SR) · ASR(Groq Whisper) · TTS(Groq)  │
├──────────────────┬──────────────────────────────────┤
│   Bridge Layer   │      Agent Brain (mimiclaw)      │
│  voice ↔ agent   │  ReAct · Memory · Skills · Tools │
├──────────────────┴──────────────────────────────────┤
│                  Network Layer                      │
│  WiFi · BLE(GPS+relay+sync) · OTA · MCP client      │
├─────────────────────────────────────────────────────┤
│                   HAL Layer                         │
│  display · touch · pmu · imu · rtc · audio · sdcard │
└─────────────────────────────────────────────────────┘
```

### 3.2 Flux vocal principal

```
1. ESP-SR détecte le wake word "Nestor" → LED blink
2. Capture audio I2S (ES8311 ADC + 2x mic)
3. Compression OPUS → stream WebSocket vers Groq Whisper
4. Transcription texte → Bridge Layer
5. Bridge → Agent Router → sélection agent pertinent
6. ReAct loop : LLM (Groq llama-3.3-70b) + tool calling
7. Réponse texte → Groq PlayAI TTS → ES8311 DAC → SPK
8. Affichage feedback LVGL (waveform + texte)
```

### 3.3 Fallback BLE (WiFi absent)

```
1. WiFi non disponible → détecté dans wifi_mgr
2. Bridge → BLE LLM_RELAY channel
3. Requête JSON → smartphone connecté (PWA Nestor)
4. PWA → Groq cloud → réponse JSON
5. ESP32 reçoit réponse → TTS local ou BLE audio
```

---

## 4. Agent Brain (mimiclaw)

### 4.1 Agents Nestor prédéfinis

| ID | Nom | Rôle | Déclencheur |
|---|---|---|---|
| `nestor-orchestrateur` | Orchestrateur | Planification méta, routage | Défaut si agent non détecté |
| `nestor-jardinier` | Jardinier | **Nettoyage/optimisation agents** à intervalle ou sur demande | Cron hebdo + commande vocale |
| `nestor-fabrique` | Fabrique | Création agents à la volée par voix | "crée un agent qui..." |
| `nestor-web-analyst` | Web Analyst | Recherche + résumé web (Brave/Tavily) | Questions factuelles |
| `nestor-mensualites` | Mensualités | Suivi finances perso | "combien je dois ce mois" |
| `nestor-pea` | PEA | Analyse portefeuille bourse | Questions bourse |
| `nestor-histoires` | Histoires | Narration créative | "raconte-moi une histoire" |
| `nestor-recherche` | Recherche ciblée | Analyse approfondie source unique | "analyse cet article" |
| `nestor-recherche-web` | Recherche web | Multi-sources agrégées | "cherche sur internet" |

### 4.2 Agent Jardinier — spécificités

L'agent Jardinier n'est **pas** un agent de jardinage : c'est le **mainteneur de la base d'agents**.

**Déclencheurs :**
- Cron hebdomadaire (dimanche 3h)
- Commande vocale explicite : "Nestor, nettoie les agents"
- Correctif manuel : quand `metrics.corrections > seuil`

**Actions :**
- Audit skill_index.json : supprime skills `usage_count=0` après 30j
- Compacte MEMORY.md si > 50KB
- Consolide les sessions archivées de > 90j
- Re-génère les system prompts des agents dégradés (confiance < 0.6)
- Supprime agents custom orphelins (jamais utilisés)
- Rapport : liste des opérations dans MEMORY.md section `## Jardinier`

### 4.3 Agent Fabrique — spécificités

Permet la création d'agents à la volée par commande vocale :

```
Utilisateur : "Crée un agent pour me résumer mes courses"
→ Fabrique génère via LLM :
  {
    id: "nestor-courses-XXXX",
    name: "Courses",
    role: "shopping-assistant",
    system_prompt: "...",
    tags: ["courses", "liste", "budget"],
    backendId: "groq-llama"
  }
→ Sauvegarde dans /fatfs/agents/custom_agents.json
→ Enregistre dans agent_router
→ Confirme vocalement
```

### 4.4 Mémoire L0-L4

| Layer | Contenu | Stockage |
|---|---|---|
| L0 | Contraintes système | Hardcodé |
| L1 | Index skills | skill_index.json |
| L2 | Faits utilisateur | MEMORY.md |
| L3 | Skills auto-cristallisés | /skills/auto/ |
| L4 | Archives sessions | /sessions/ |

### 4.5 Auto-cristallisation

Quand une tâche multi-étapes réussit (≥2 tool calls), une skill est automatiquement créée dans `/fatfs/skills/auto/`. Le Jardinier fait le ménage de ces skills selon leur usage.

---

## 5. Groq — LLM + ASR + TTS

### 5.1 Modèles utilisés

| Service | Modèle | Endpoint | Notes |
|---|---|---|---|
| LLM | `llama-3.3-70b-versatile` | `POST /openai/v1/chat/completions` | Gratuit (generous tier) |
| LLM rapide | `llama-3.1-8b-instant` | idem | Fallback si rate limit |
| ASR | `whisper-large-v3` | `POST /openai/v1/audio/transcriptions` | Upload audio multipart |
| TTS | `playai-tts` | `POST /openai/v1/audio/speech` | WAV output |

### 5.2 Rate limits Groq free tier

| Ressource | Limite |
|---|---|
| LLM req/min | 30 |
| LLM tokens/min | 6000 |
| Whisper req/min | 20 |
| TTS req/min | 20 |

### 5.3 Configuration Kconfig

```
CONFIG_COMPAGNON_LLM_PROVIDER="groq"
CONFIG_COMPAGNON_LLM_MODEL="llama-3.3-70b-versatile"
CONFIG_COMPAGNON_LLM_MODEL_FAST="llama-3.1-8b-instant"
CONFIG_COMPAGNON_LLM_URL="https://api.groq.com/openai/v1/chat/completions"
CONFIG_COMPAGNON_ASR_URL="https://api.groq.com/openai/v1/audio/transcriptions"
CONFIG_COMPAGNON_ASR_MODEL="whisper-large-v3"
CONFIG_COMPAGNON_TTS_URL="https://api.groq.com/openai/v1/audio/speech"
CONFIG_COMPAGNON_TTS_MODEL="playai-tts"
CONFIG_COMPAGNON_TTS_VOICE="Fritz-PlayAI"
CONFIG_COMPAGNON_SECRET_API_KEY="<votre_cle_groq>"
```

---

## 6. BLE Multi-caractéristiques

| Caractéristique | UUID suffix | Usage |
|---|---|---|
| GPS | `BEB5483E-...` | Position lat,lon depuis téléphone |
| WIFI_SCAN | `0001` | Scan réseaux |
| WIFI_PROVISION | `0002` | Connexion WiFi |
| AGENT_SYNC | `0003` | Sync agents bidirectionnelle |
| TEXT_INPUT | `0004` | Clavier distant |
| LLM_RELAY | `0005` | **Relay LLM via smartphone (fallback WiFi)** |
| DEVICE_STATUS | `0006` | Statut device |

---

## 7. Partition Table (16 MB Flash)

| Partition | Taille | Usage |
|---|---|---|
| nvs | 32 KB | Config non-volatile |
| otadata | 8 KB | OTA |
| phy_init | 4 KB | RF |
| ota_0 | 5 MB | Firmware principal |
| ota_1 | 5 MB | OTA backup |
| assets | 5 MB | Wake word + assets LVGL |
| model | 1 MB | Réservé modèle local futur |
| fatfs | ~4 MB | Mémoire agents + sessions + skills |

---

## 8. Différences vs Compagnon v1 (Waveshare 2.16")

| Feature | Compagnon v1 | Compagnon2 |
|---|---|---|
| LLM | Relay PWA (pas de LLM embarqué) | **ReAct loop embarqué** (mimiclaw) |
| ASR | Aucun | **Groq Whisper streaming** |
| Réveil vocal | Non | **ESP-SR wake word** |
| Agents | Définis dans PWA | **Embarqués dans FATFS** |
| Mémoire | IndexedDB PWA | **FATFS L0-L4 on-device** |
| Auto-apprentissage | Non | **Auto-cristallisation skills** |
| Agent Jardinier | Manuel PWA | **Cron embarqué + commande vocale** |
| Agent Fabrique | PWA | **Embarqué, création vocale** |
| BLE | GPS only (implémenté) | **GPS + LLM_RELAY + AGENT_SYNC** |
| RTC | NTP seul | **PCF85063 + NTP fallback** |
| Résolution | 480×480 | **460×470** |

---

## 9. TODO / Roadmap

### Phase 1 — Base (ce repo)
- [x] SPEC + README
- [x] Structure fichiers + partitions
- [x] Config Groq (LLM + ASR + TTS)
- [x] Agents Nestor définis (FATFS)
- [x] Personnalité SOUL.md Nestor
- [ ] HAL Waveshare 2.16" 460×470 (portage depuis Compagnon v1)
- [ ] Voice I/O layer (ASR/TTS Groq, wake word ESP-SR)
- [ ] Bridge voice ↔ agent
- [ ] Agent brain ReAct loop (mimiclaw)
- [ ] Agent Jardinier cron
- [ ] Agent Fabrique vocal
- [ ] BLE LLM_RELAY channel
- [ ] LVGL UI agent_ui + voice_ui

### Phase 2 — Complétion
- [ ] App Radars (GPS BLE + Lufop)
- [ ] App Bourse (Twelve Data)
- [ ] App Météo (portage depuis Compagnon v1)
- [ ] SD card pour sessions longues
- [ ] OTA
