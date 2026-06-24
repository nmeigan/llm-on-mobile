# Solution Architecture — Local LLM Agent on Android

**Document Type:** Technical Architecture Reference
**Version:** 1.0
**Author:** Enun Ismath Naina Mohamed
**Scope:** On-device sovereign AI agent using Android + Termux + llama.cpp

---

## ⚠️ Disclaimer

This document is provided for educational and informational purposes only. Hardware results vary by device. Proceed at your own risk. The author accepts no liability for any outcomes. Back up your data before making any system changes. See the full disclaimer in [README.md](../README.md).

---

## 1. Executive Summary

This architecture delivers a fully sovereign, offline-capable AI agent running on commodity Android hardware. The system achieves professional-grade reasoning and coding assistance at zero marginal cost, with zero data egress, using only hardware the user already owns.

**Key architectural properties:**

| Property | Value |
|---|---|
| Inference location | On-device (local CPU) |
| Network dependency | None (after one-time model download) |
| Data egress | Zero — no data leaves the device |
| Marginal cost per query | $0.00 |
| Model parameters | 1.5 billion (Qwen 2.5 1.5B Instruct) |
| Quantisation | Q4_K_M (4-bit) or Q6_K (6-bit) |
| Runtime | llama.cpp via Termux |

---

## 2. The Architectural Insight — Unified Memory on Android

### 2.1 The Apple UMA Reference Pattern

Apple's M-series chips implement Unified Memory Architecture (UMA): a single shared memory pool accessible by both the CPU and GPU at full memory bandwidth. This eliminates the VRAM bottleneck that constrains discrete GPU setups. A MacBook M2 with 16 GB unified memory can run a 7B parameter model that would overflow an NVIDIA GPU with 8 GB VRAM.

On NVIDIA hardware, the constraint hierarchy is:
```
VRAM (fastest, smallest) → PCIe bandwidth → System RAM (slowest, largest)
```
When a model's parameters exceed VRAM, the system spills to system RAM via PCIe. Bandwidth drops from ~900 GB/s (HBM) to ~64 GB/s (PCIe 4.0). The fastest lane becomes the slowest lane. Throughput collapses.

### 2.2 Android RAM Boost — A Mobile UMA Analogue

Android's RAM Boost (marketed as RAM Plus, Memory Extension, or Memory Turbo by different manufacturers) creates a virtual memory pool by pairing physical RAM with a reserved block of high-speed storage. This extends the effective addressable memory space without requiring dedicated hardware.

```
Physical RAM: 12 GB
RAM Boost (virtual): +6 GB
Effective addressable pool: ~18 GB
```

For a 1.5B parameter model at Q4_K_M quantisation (~1.0 GB), the entire model loads within the physical RAM layer. No spilling occurs. The CPU executes inference entirely within the fast memory tier.

### 2.3 Why This Works — The Quantisation Bridge

Full-precision (FP32) 1.5B parameters = ~6 GB
FP16 = ~3 GB
Q4_K_M (4-bit, mixed) = ~1.0 GB
Q6_K (6-bit) = ~1.3 GB

4-bit quantisation reduces model size by ~6× versus FP16 with minimal accuracy degradation on task-specific benchmarks. The model's weights are compressed to 4 bits per parameter, with K-means grouping (the `_K_M` suffix) preserving the most important weight distributions.

**Net result:** A model requiring 3 GB in cloud deployment loads in 1 GB on device, fits within the physical RAM tier, and runs at full CPU throughput.

---

## 3. System Architecture

### 3.1 Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER LAYER                              │
│                                                                 │
│  Input: Natural language query (code, reasoning, chat)          │
│  Output: Generated response at 5–8 tokens/sec                   │
│  Interface: Termux terminal (text-based)                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                      RUNTIME LAYER                              │
│                                                                 │
│  llama-cli (llama.cpp)                                          │
│  ├── Conversation mode: -cnv                                    │
│  ├── Context window: 4096 tokens                                │
│  ├── Thread management: -t 6 (tuned to CPU core count)          │
│  ├── System prompt injection: -p "..."                          │
│  └── Memory-mapped model loading (mmap)                         │
│                                                                 │
│  Termux (Linux environment on Android)                          │
│  ├── Package manager: pkg (apt-based)                           │
│  ├── Filesystem: ~/models/ (model storage)                      │
│  └── Process management: foreground terminal process            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                       MODEL LAYER                               │
│                                                                 │
│  Qwen 2.5 1.5B Instruct — GGUF Format                          │
│  ├── Architecture: Transformer (decoder-only)                   │
│  ├── Parameters: 1.5 billion                                    │
│  ├── Quantisation: Q4_K_M (~1.0 GB) or Q6_K (~1.3 GB)          │
│  ├── Context: up to 32K tokens (limited to 4K on device)        │
│  ├── Training: instruction-tuned for chat and reasoning         │
│  └── Storage: ~/models/*.gguf (local filesystem)                │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                      HARDWARE LAYER                             │
│                                                                 │
│  CPU: Multi-core ARM (Snapdragon / Dimensity / Exynos)          │
│  ├── Performance cores: 2–4 (high-frequency inference)          │
│  └── Efficiency cores: 4–6 (sustained throughput)               │
│                                                                 │
│  Memory:                                                        │
│  ├── Physical RAM: 8–12 GB LPDDR5                               │
│  ├── RAM Boost (virtual): +4–8 GB (storage-backed)             │
│  └── Effective pool: 12–18 GB                                   │
│                                                                 │
│  Storage: UFS 3.1 / UFS 4.0 (model file: 1.0–1.3 GB)           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow

```
User Input (Termux terminal)
        │
        ▼
llama-cli tokeniser
        │  (text → token IDs via Qwen tokeniser vocab)
        ▼
Context buffer (4096 token rolling window)
        │
        ▼
Transformer forward pass (CPU, 6 threads)
        │  (~5–8 tokens/sec generation rate)
        ▼
Sampler (temperature, top-p, repetition penalty)
        │
        ▼
De-tokeniser → human-readable text
        │
        ▼
Terminal output (colour-coded via --color on)
```

### 3.3 Memory Allocation at Runtime

| Component | Memory Usage |
|---|---|
| Android OS baseline | ~2.5–3.5 GB |
| Termux process | ~50–100 MB |
| llama.cpp runtime | ~100–200 MB |
| Model weights (Q4_K_M) | ~1.0 GB |
| KV cache (4096 context) | ~300–500 MB |
| **Total** | **~4.5–5.5 GB** |

On an 8 GB device, this leaves ~2.5–3.5 GB for OS and other processes — workable but tight. On 12 GB, comfortable headroom exists.

---

## 4. Performance Profile

### 4.1 Observed Benchmarks (Test Device: 12 GB RAM Android)

| Metric | Target | Achieved |
|---|---|---|
| Output speed | ≥ 5 tok/sec | 5–8 tok/sec |
| Reasoning latency | ≤ 5× output length | Within target |
| Quality baseline | Claude Haiku equivalent | Met for coding + reasoning tasks |
| Startup time | < 10 seconds | 5–8 seconds (warm), 10–15 sec (cold) |

### 4.2 Performance Factors

**Positive impact on speed:**
- Higher RAM (less memory pressure)
- RAM Boost enabled (extended addressable space)
- CPU pinned (Termux not in background)
- Phone plugged in (no thermal throttling)
- Fewer background apps

**Negative impact on speed:**
- Phantom Process Killer activity (Android memory management)
- Thermal throttling (phone hot / battery saver active)
- Background app activity consuming RAM
- Large context (> 2048 tokens in active window)

---

## 5. Agent Modes and Use Cases

### 5.1 Coding Agent

**System prompt:**
```
You are an expert software engineer. You write clean, efficient, well-commented code. 
When asked to review code, identify bugs, suggest improvements, and explain your reasoning.
```

**Validated tasks:**
- Python script generation
- SQL query construction
- Bash/shell scripting
- Code review and refactoring suggestions
- Debugging with error message analysis

### 5.2 Reasoning Agent

**System prompt:**
```
You are an analytical reasoning assistant. Think step by step. 
Break problems into components, reason through each, and synthesise a clear conclusion.
```

**Validated tasks:**
- Multi-step logical reasoning
- Decision framework analysis
- Root cause analysis
- Structured problem decomposition

### 5.3 General Chat Agent

**System prompt (used in this project):**
```
You are a helpful, candid, and highly efficient AI assistant specialized 
in precise reasoning and clear explanations.
```

**Validated tasks:**
- General Q&A
- Summarisation
- Explanation of technical concepts
- Drafting and editing text

---

## 6. Scalability Pattern — From Phone to Enterprise Edge

The pattern proven on a phone maps directly to enterprise edge deployments:

| Scale | Hardware | Use Case |
|---|---|---|
| Personal | Android phone (12 GB RAM) | Individual sovereign AI agent |
| Team | Raspberry Pi 5 / Mini PC (32 GB RAM) | Shared local inference endpoint |
| Branch | Edge server (128 GB RAM) | Bank branch AI, offline capable |
| Data centre | Sovereign compute cluster | National / institutional AI, air-gapped |

**Invariant properties at all scales:**
- Quantised open-weight models
- Local inference, zero egress
- Persistent context on-device
- Zero per-query cost

This is the reference architecture for any organisation requiring AI capability without data sovereignty risk.

---

## 7. Security and Privacy Properties

| Property | Status |
|---|---|
| Data leaves the device | Never (post-download) |
| Model weights are proprietary | No — Qwen 2.5 is Apache 2.0 licensed |
| API keys required | None |
| Account / login required | None |
| Telemetry / usage tracking | None |
| Conversation logging | None (unless explicitly configured) |

**Identity sovereignty:** Your queries, context, and conversation history exist only on your device. No third party has access to your interaction data. Your personal context — your thinking patterns, your professional reasoning style — is not stored on, or shaped by, someone else's infrastructure.

---

## 8. Known Limitations

| Limitation | Detail |
|---|---|
| Model size | 1.5B parameters — capable but not GPT-4 class |
| Context window | Limited to 4096 tokens on device (vs 100K+ in cloud) |
| Multi-modal | Text only — no image, audio, or video input |
| Speed | 5–8 tok/sec vs 50–100+ tok/sec from cloud APIs |
| Knowledge cutoff | Fixed at model training date — no live web access |
| Fine-tuning | Not practical on-device with current tooling |

---

*Architecture designed and validated by Enun Ismath Naina Mohamed.*
*Tested on Android hardware, June 2026.*
