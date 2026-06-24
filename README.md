# What Every CTO Gets Wrong About the Cost of Intelligence — Sovereign AI on Any Device

> **Most organisations are stalling on AI because the token bill is unpredictable and the ROI isn't quantifiable. This project flips the problem — a full AI coding, reasoning, and chat agent running on an Android phone. Zero cloud. Zero tokens. Zero ongoing cost.**

---

## ⚠️ Disclaimer

**This project is shared for educational and entertainment purposes only.**

- Every Android device has different hardware specifications, RAM configurations, and OS build variants. Results will vary across devices.
- Setup steps may differ based on your phone manufacturer, Android version, and Termux build.
- **Back up your device data before making any system-level changes** (enabling Developer Options, RAM Boost, or virtual memory settings).
- The author accepts **no liability or responsibility** for any damage, data loss, or unintended outcomes resulting from following this guide.
- Proceed **entirely at your own risk and at your own discretion**.
- This is not a commercial product, not a supported tool, and not professional IT advice.

---

## What This Is

A personal experiment and proof-of-concept demonstrating that a modern Android phone — with 8–12 GB RAM — can run a quantised large language model locally via [llama.cpp](https://github.com/ggerganov/llama.cpp) and [Termux](https://termux.dev), delivering:

| Capability | Detail |
|---|---|
| **Coding Agent** | Write, review, and debug code — fully local |
| **Reasoning Agent** | Multi-step inference and structured analysis |
| **Chat Agent** | Context-aware conversation, persistent session |
| **Zero Cloud** | No internet required after model download |
| **Zero Tokens** | No API calls, no billing, no rate limits |
| **Data Sovereignty** | All data stays on your device |

**Benchmark achieved on test device:**
- Output speed: **5–8 tokens/sec** (~5 words/sec — comfortable reading pace)
- Reasoning latency: **≤ 5× output length**
- Quality reference: **Claude Haiku** (lightweight cloud model baseline)

---

## Why This Matters — The Strategic Context

Enterprise AI adoption in 2026 faces three converging problems:

1. **Token cost unpredictability** — per-query cloud pricing is unforecastable for finance teams
2. **Data sovereignty pressure** — regulators in GCC, Singapore, and Malaysia are tightening where data can be processed
3. **Vendor dependency risk** — intelligence mediated by third-party APIs creates structural fragility

This proof-of-concept demonstrates the pattern that resolves all three:
- Quantised open-weight models on hardware you already own
- Zero data egress, zero marginal cost per query
- The same pattern scales from a phone to enterprise edge compute

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   ANDROID PHONE                         │
│                                                         │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │   Termux     │    │        Memory Layer          │   │
│  │  (Linux env) │    │  Physical RAM: 8–12 GB       │   │
│  │              │    │  RAM Boost (virtual): +4–6GB │   │
│  │  llama.cpp   │───▶│  Total addressable: ~16–18GB │   │
│  │  llama-cli   │    └──────────────────────────────┘   │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │              MODEL LAYER                         │   │
│  │  Qwen 2.5 1.5B Instruct — Q4_K_M or Q6_K        │   │
│  │  GGUF format — ~1.0–1.5 GB on disk               │   │
│  │  CPU inference — no GPU / NPU required           │   │
│  └──────────────────────────────────────────────────┘   │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │            AGENT INTERFACE LAYER                 │   │
│  │  Mode: -cnv (conversation / chat loop)           │   │
│  │  Context window: 4096 tokens                     │   │
│  │  Threads: 6 (tuned to CPU core count)            │   │
│  │  System prompt: custom persona / role            │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### The UMA Insight

Apple's Unified Memory Architecture (UMA) on M-series chips collapses RAM and SSD into a single shared memory pool — allowing large models to run that would overflow a discrete GPU's VRAM. On NVIDIA hardware, VRAM is the ceiling; when parameters overflow, the system falls back to PCIe bandwidth, and the fastest lane becomes the slowest lane.

Android's RAM Boost feature creates an analogous effect at a smaller scale — expanding addressable memory beyond physical RAM by pairing it with a virtual memory extension. Combined with quantised model weights (Q4/Q6), the 1.5B parameter model fits comfortably within the available memory pool.

---

## Quick Start

See the full step-by-step guide: **[docs/SOP-setup-guide.md](docs/SOP-setup-guide.md)**

**Minimum requirements:**
- Android phone with **8 GB RAM minimum** (12 GB recommended)
- **Termux** installed (F-Droid version recommended over Play Store)
- Internet connection for the initial one-time model download (~1.3 GB)
- Free storage: **3–5 GB** recommended

**Final working command (copy into Termux):**
```bash
llama-cli \
  -m qwen2.5-1.5b-instruct-q4_k_m.gguf \
  -c 4096 \
  -t 6 \
  -cnv \
  -p "You are a helpful, candid, and highly efficient AI assistant specialized in precise reasoning and clear explanations." \
  --color on
```

---

## Repository Structure

```
├── README.md                  # This file — overview and architecture
├── docs/
│   ├── SOP-setup-guide.md     # Full step-by-step setup and troubleshooting
│   └── solution-architecture.md  # Detailed architecture and design rationale
└── assets/
    └── cover.png              # Visual architecture diagram
```

---

## Model Options

| Model | Quantisation | Size | Best For |
|---|---|---|---|
| Qwen 2.5 1.5B Instruct | Q4_K_M | ~1.0 GB | Speed, lower RAM usage |
| Qwen 2.5 1.5B Instruct | Q6_K | ~1.3 GB | Higher quality, more RAM |
| Llama 3.2 3B Instruct | Q4_K_M | ~2.0 GB | More capable, needs 10GB+ RAM |

For 8 GB RAM devices: use Q4_K_M. For 12 GB RAM devices: Q6_K is stable.

---

## Author

**Enun Ismath Naina Mohamed**
Chief Data & AI Officer | AI Architect | Enterprise Intelligence Leader

[LinkedIn](https://www.linkedin.com/in/enunismathnainamohamed)

---

## License

MIT License — use freely, modify freely, share with attribution.
No warranties expressed or implied. See disclaimer at the top of this file.
