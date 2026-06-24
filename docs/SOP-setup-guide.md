# SOP — Setup Guide: Local LLM Agent on Android

**Standard Operating Procedure**
**Version:** 1.0
**Author:** Enun Ismath Naina Mohamed
**Last Updated:** June 2026

---

## ⚠️ Important Disclaimer

**This guide is provided for educational and entertainment purposes only.**

- Hardware specifications, Android versions, and Termux builds vary widely across devices. These steps were validated on a specific test device; your device may require different configuration.
- **Back up all data on your phone before proceeding.** Enabling Developer Options, RAM Boost, or virtual memory settings are system-level changes.
- The author accepts **no liability or responsibility** for any damage, data loss, performance issues, or unintended consequences.
- **Proceed entirely at your own risk and at your own discretion.**
- Not professional IT advice. Not a supported commercial product.

---

## Overview

This SOP covers the complete process for installing and running a local, offline AI assistant on an Android phone using:

- **Termux** — Linux terminal environment for Android
- **llama.cpp** — High-performance CPU inference engine for quantised LLMs
- **Qwen 2.5 1.5B Instruct** — Lightweight reasoning model in GGUF format

**Estimated setup time:** 20–40 minutes (plus model download time, varies by connection speed)

---

## Phase 0 — Prerequisites

### Device Requirements

| Requirement | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB | 12 GB |
| Storage free | 3 GB | 5 GB |
| Android version | 10+ | 12+ |
| CPU cores | 6 | 8+ |

### What You Need

- [ ] Android phone meeting minimum specs above
- [ ] A working internet connection (for initial download only)
- [ ] 30–60 minutes uninterrupted time
- [ ] Phone plugged into charger (model download and first run are CPU-intensive)

---

## Phase 1 — Enable Developer Options and RAM Boost

> **⚠️ At your own risk. Steps vary by manufacturer.**

### Step 1.1 — Enable Developer Options

1. Open **Settings** on your phone
2. Navigate to **About Phone** → **Software Information**
3. Tap **Build Number** 7 times rapidly
4. You will see: *"You are now a developer"*
5. Developer Options will appear in your main Settings menu

### Step 1.2 — Enable RAM Boost (Virtual Memory)

> **Note:** This feature is named differently across manufacturers:
> - Samsung: **RAM Plus**
> - Xiaomi/POCO: **Memory Extension**
> - OnePlus: **RAM Boost**
> - Honor/Huawei: **Memory Turbo** or **RAM Expansion**
> - Realme/OPPO: **RAM Expansion**

1. Open **Settings** → **About Phone** (or **Battery & Device Care** on Samsung)
2. Find the RAM/Memory section
3. Enable **RAM Plus / Memory Extension / RAM Boost** and set to maximum available (typically 4–8 GB)
4. Reboot your phone to apply

**Effect:** This creates a virtual memory pool backed by your storage, expanding your effective addressable memory from 8 GB to ~12–16 GB.

---

## Phase 2 — Install Termux

> **Important:** Use the F-Droid version of Termux, not the Google Play Store version. The Play Store version is outdated and may lack required packages.

### Step 2.1 — Install F-Droid

1. Open your browser and go to: `https://f-droid.org`
2. Download and install the F-Droid APK
3. When prompted, allow installation from unknown sources for your browser app
4. Install F-Droid

### Step 2.2 — Install Termux via F-Droid

1. Open F-Droid
2. Search for **Termux**
3. Install the official Termux package (by Fredrik Fornwall)
4. Open Termux once installed

### Step 2.3 — Initial Termux Setup

Run the following in Termux to update and install core tools:

```bash
pkg update && pkg upgrade -y
pkg install wget curl -y
```

Allow storage access when prompted:
```bash
termux-setup-storage
```

---

## Phase 3 — Install llama.cpp

### Step 3.1 — Install via Package Manager

```bash
pkg install llama-cpp -y
```

This installs the full llama.cpp suite including `llama-cli` and `llama-server`.

### Step 3.2 — Verify Installation

```bash
llama-cli --version
```

You should see a version number output. If you get `command not found`, repeat Step 3.1 or try:

```bash
pkg install clang cmake -y
```

---

## Phase 4 — Download the Model

> The model downloads once. After that, everything runs fully offline.

### Step 4.1 — Create a Models Directory

```bash
mkdir -p ~/models
cd ~/models
```

### Step 4.2 — Download Qwen 2.5 1.5B Instruct (Q4 — Recommended for 8 GB devices)

```bash
wget -O qwen2.5-1.5b-instruct-q4_k_m.gguf \
  "https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf"
```

**File size:** ~1.0 GB — allow 5–15 minutes depending on your connection.

### Alternative — Q6 Quantisation (Higher Quality, for 12 GB devices)

```bash
wget -O qwen2.5-1.5b-instruct-q6_k.gguf \
  "https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q6_k.gguf"
```

**File size:** ~1.3 GB

### Step 4.3 — Verify Download

```bash
ls -lh ~/models/
```

Confirm the file is present and the size matches expected (~1.0 GB for Q4, ~1.3 GB for Q6).

---

## Phase 5 — Launch Your AI Agent

### Step 5.1 — The Working Command (Q4 Model)

```bash
llama-cli \
  -m ~/models/qwen2.5-1.5b-instruct-q4_k_m.gguf \
  -c 4096 \
  -t 6 \
  -cnv \
  -p "You are a helpful, candid, and highly efficient AI assistant specialized in precise reasoning and clear explanations." \
  --color on
```

### Step 5.2 — Parameter Reference

| Parameter | Value | Purpose |
|---|---|---|
| `-m` | path to .gguf file | Load the local model |
| `-c 4096` | 4096 tokens | Context window size — balances memory and conversation history |
| `-t 6` | 6 threads | CPU thread count — set to number of efficiency/performance cores |
| `-cnv` | — | Conversation mode (replaces deprecated `-i` flag) |
| `-p "..."` | system prompt | Define agent persona and behaviour |
| `--color on` | on | Coloured terminal output for readability |

### Step 5.3 — Expected Behaviour

After running the command:
1. You will see model loading progress (memory map loading)
2. A `>` or `User:` prompt will appear
3. Type your question and press Enter
4. The model responds at ~5–8 tokens/second

**First response is slower** — the model is warming up. Subsequent responses are faster.

---

## Phase 6 — Tune for Your Device

### Adjust Thread Count (`-t`)

Set `-t` to match your device's CPU core count:

| Device CPU | Recommended `-t` |
|---|---|
| 6-core (2P + 4E) | `-t 4` or `-t 5` |
| 8-core (2P + 6E) | `-t 6` |
| 8-core (4P + 4E) | `-t 6` or `-t 7` |
| 10-core+ | `-t 8` |

### Reduce Context if RAM is Tight

If the model crashes or is very slow, reduce context:

```bash
-c 2048
```

### Switch to Lighter Quantisation

If Q6 is unstable, fall back to Q4:

```bash
-m ~/models/qwen2.5-1.5b-instruct-q4_k_m.gguf
```

---

## Troubleshooting

### Error: `invalid argument: -i`

The `-i` (interactive) flag is deprecated in newer llama.cpp builds.
**Fix:** Replace `-i` with `-cnv`

### Error: `expected value for argument --flash-attn`

Newer versions require an explicit value.
**Fix:** Use `--flash-attn on` or remove the flag entirely (it defaults to auto)

### Error: `expected value for argument --color`

**Fix:** Use `--color on` instead of just `--color`

### Error: `no GGUF files found in repository`

The `-hf` auto-downloader cannot resolve the HuggingFace repo structure in some Termux environments.
**Fix:** Use `wget` to manually download the model file (see Phase 4 above)

### Error: `failed to download model from Hugging Face`

**Fix:** Download manually via `wget` as shown in Phase 4. Ensure Termux has internet permissions (not blocked by VPN or corporate firewall).

### Process Killed / Out of Memory

Android's Phantom Process Killer may terminate Termux when RAM usage is high.
**Fixes:**
1. Enable RAM Boost / RAM Plus (see Phase 1)
2. Pin Termux in recent apps (tap the lock icon on the Termux card)
3. Disable battery optimisation for Termux in Settings → Apps → Termux → Battery
4. Reduce context: change `-c 4096` to `-c 2048`
5. Use Q4 quantisation instead of Q6

### Very Slow Generation (< 2 tokens/sec)

**Fixes:**
1. Close all other apps — Android's memory manager will be stealing RAM
2. Reduce thread count: try `-t 4`
3. Reduce context: try `-c 2048`
4. Switch to Q4 model (faster inference than Q6)
5. Ensure phone is plugged in — thermal throttling on battery can halve performance

### Termux Closes Unexpectedly

1. Keep screen on while running: Settings → Developer Options → Stay Awake
2. Disable battery saver / power saving mode while running the model
3. Allow Termux to run unrestricted in battery settings

---

## Keeping Your Session Running

To prevent Termux from closing when you switch apps:

```bash
# Install the Termux wake lock utility
pkg install termux-api -y

# Acquire a wake lock (prevents the phone from sleeping during inference)
termux-wake-lock
```

---

## What's Next

Once your local agent is running:

- **llama-server mode** — expose a local HTTP API compatible with OpenAI client libraries, enabling front-end apps to connect
- **Model upgrades** — try Llama 3.2 3B (Q4, ~2 GB) on 12 GB RAM devices for higher capability
- **Custom system prompts** — tune the `-p` parameter for specific roles: coder, analyst, tutor

---

*This guide was compiled from hands-on experimentation and troubleshooting on an Android device. Your experience may vary. Always back up your data before making system changes.*
