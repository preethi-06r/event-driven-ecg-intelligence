# event-driven-ecg-intelligence
STM32-based event-driven ECG intelligence system for low-power biomedical monitoring.
# 🫀 Event-Driven Hierarchical ECG Intelligence for Wearables (STM32)

A power-aware, tiered ECG intelligence pipeline that runs continuous cardiac monitoring on resource-constrained wearable hardware — without draining the battery in a day.

---

## 📌 Problem Statement

Continuous ECG monitoring is computationally expensive and power-hungry — a critical bottleneck for battery-limited wearable devices. Running full-blown arrhythmia classification on every single heartbeat is wasteful, since the vast majority of beats are perfectly normal.

This project designs a hierarchical, event-driven ECG intelligence system on the STM32 that only spends CPU/power budget when something actually looks suspicious.

---

## 🏗️ System Architecture

Three tiers, escalating in cost and only activated when needed:

| Tier | Role | Runs On | Power Cost | Trigger Frequency |
|------|------|---------|------------|-------------------|
| **Tier 0 — Sentinel** | Hardware-level "is a beat happening?" detector | STM32 ADC analog watchdog (pure hardware, CPU asleep) | ~µW | Continuous |
| **Tier 1 — Screener** | Lightweight normal vs. suspicious beat classifier | Quantized INT8 tiny model on main core | mW, short bursts | Every beat (~1–1.5 Hz) |
| **Tier 2 — Specialist** | Full arrhythmia classification (AFib, PVC, etc.) | Deeper CNN/LSTM, invoked only on escalation | mW–tens of mW, rare | ~1–5% of beats |

**Core principle:** the MCU spends most of its life in STOP mode. Wake-ups are triggered by hardware (analog watchdog on the ADC), not by continuous polling.

```
ECG Signal ──▶ [Tier 0: HW Watchdog] ──▶ (beat detected?)
                                              │
                                              ▼
                                    [Tier 1: Screener Model]
                                              │
                              normal ◀────────┼────────▶ suspicious
                                │                              │
                          stay in low-power                    ▼
                            duty cycle                [Tier 2: Specialist Model]
                                                                │
                                                                ▼
                                                     [Alert / BLE Transmission]
```

---

## 🌟 Key Innovation 1: Patient-Adaptive Confidence Gating

Most cascade-based ECG systems use a fixed threshold to decide when to escalate from Tier 1 → Tier 2. The problem: "normal" ECG morphology varies significantly across individuals — a fixed threshold either over-triggers on some patients or misses events on others.

This system instead:

- Builds a personalized baseline during an initial learning window (resting HR, HRV, typical QRS width) using lightweight running statistics — no heavy on-device training required.
- Adapts the escalation threshold per patient in real time based on that baseline.
- Closes the loop with power policy: when escalation rate rises (device senses increased risk), sampling rate and Tier-1 inference frequency temporarily increase, then back off automatically once things stabilize.

This turns a static cascade into a self-calibrating, risk-adaptive system — coupling diagnostic confidence directly to power/sampling policy.

---

## 🌟 Key Innovation 2: DMA-Driven Relay Buffer for CPU-Awake Low-Power Monitoring

### The Problem with Sleep-Wake Cycling During Normal Activity

The naive power strategy — putting the CPU into STOP/deep-sleep and waking it on every beat — introduces two real costs that compound over time:

1. **Wake-up latency:** Every STOP → Active transition takes 5–20 µs of overhead, during which the ADC may have already advanced past the beat onset.
2. **Thermal and power micro-transients:** Repeated full sleep/wake cycles cause frequent voltage regulator and PLL restarts, which are themselves energy-expensive and can introduce noise on the analog front-end supply rail.

During periods of normal, sustained cardiac activity (regular sinus rhythm, stable HRV), constantly toggling STOP mode is counterproductive.

---

### The Solution: DMA + Circular Relay Buffer

During normal heart activity, the CPU does **not** enter STOP mode. Instead, it stays in a low-power **RUN mode at reduced clock** (e.g., 4–8 MHz on STM32L4), while a **DMA channel autonomously handles all ADC sample transfers** into a **circular relay buffer** in SRAM — completely without CPU involvement.

```
                    ┌──────────────────────────────────────────┐
                    │           DMA Channel (autonomous)        │
                    │                                          │
  AD8232 ECG  ──▶  │  ADC (continuous)  ──DMA──▶  [Relay      │
  Analog Signal     │                              Buffer      │
                    │                              Circular    │
                    │                              SRAM Ring]  │
                    └────────────────────┬─────────────────────┘
                                         │
                              DMA Half/Full Transfer
                              Complete Interrupt
                                         │
                                         ▼
                          ┌──────────────────────────┐
                          │  CPU (low-power RUN mode) │
                          │  4–8 MHz, reduced Vcore   │
                          │                          │
                          │  Wakes ONLY on DMA IRQ   │
                          │  → reads relay buffer    │
                          │  → runs Tier-1 screener  │
                          │  → goes back to low-power│
                          │    idle (WFI, not STOP)  │
                          └──────────────────────────┘
```

---

### How the Relay Buffer Works

The relay buffer is a **circular (ring) buffer** in SRAM, sized to hold one full ECG beat window (e.g., 256–512 samples at 360 Hz ≈ 700–1400 ms of signal). The DMA controller is configured in **circular double-buffer mode**, filling the ring continuously.

```c
/* Example STM32 HAL DMA setup (Person A firmware) */

#define RELAY_BUF_SIZE  512   // samples — holds ~1.4 s of ECG at 360 Hz

int16_t relay_buffer[RELAY_BUF_SIZE];  // SRAM relay ring

// ADC + DMA configured for circular mode:
// - DMA fires a Half-Transfer interrupt at sample 256
// - DMA fires a Transfer-Complete interrupt at sample 512 (wraps to 0)
// CPU processes the "cold half" while DMA fills the "hot half"

HAL_ADC_Start_DMA(&hadc1, (uint32_t*)relay_buffer, RELAY_BUF_SIZE);
// CPU is now free — DMA feeds ECG samples autonomously
```

**Double-half-buffer processing pattern:**

```
 Relay Buffer (512 samples, circular):
 ┌──────────────────┬──────────────────┐
 │  First Half      │  Second Half     │
 │  [0 ... 255]     │  [256 ... 511]   │
 └──────────────────┴──────────────────┘
        ▲                    ▲
        │                    │
   HT interrupt:        TC interrupt:
   CPU reads [0..255]   CPU reads [256..511]
   while DMA fills      while DMA fills
   [256..511]           [0..255] again
```

The CPU wakes from **WFI (Wait For Interrupt)** — a lightweight idle that keeps clocks and state alive — rather than from STOP mode. This means:

- No PLL/oscillator restart penalty
- No ADC reconfiguration on wake
- Sub-microsecond response to the DMA interrupt
- CPU core power reduced via **voltage scaling (VOS)** and low clock divider, not full shutdown

---

### Power State During Normal Activity vs. Escalation

```
Normal Sinus Rhythm (majority of time):
┌─────────────────────────────────────────────────────────┐
│ CPU: Low-power RUN (4–8 MHz, VOS Range 2)               │
│ DMA: Active, filling relay buffer continuously          │
│ ADC: Continuous conversion, DMA-driven                  │
│ CPU duty cycle: Short WFI idle → DMA IRQ → Tier-1 →    │
│                 WFI idle → DMA IRQ → Tier-1 → ...       │
│ Estimated current: ~1–3 mA                              │
└─────────────────────────────────────────────────────────┘

Suspicious Beat Detected (Tier-1 escalates):
┌─────────────────────────────────────────────────────────┐
│ CPU: Full-speed RUN (80 MHz) for Tier-2 inference       │
│ DMA: Continues filling relay buffer (no gap in signal)  │
│ Duration: ~20–100 ms for Tier-2 inference               │
│ Then: CPU drops back to low-power RUN automatically     │
└─────────────────────────────────────────────────────────┘

Extended Inactivity / No Signal (e.g., device removed):
┌─────────────────────────────────────────────────────────┐
│ CPU: STOP mode (full deep sleep, ~2–5 µA)               │
│ DMA: Halted                                             │
│ Wake source: RTC timeout or analog watchdog threshold   │
└─────────────────────────────────────────────────────────┘
```

---

### Why Not Just Use STOP Mode During Normal Activity?

| Criterion | STOP Mode on Every Beat | DMA Relay Buffer (Low-Power RUN) |
|-----------|------------------------|----------------------------------|
| ADC continuity | ❌ ADC must restart on wake | ✅ ADC runs uninterrupted |
| Wake latency | ~5–20 µs PLL restart overhead | <1 µs WFI → IRQ response |
| Signal integrity | Risk of sample gaps at wake edge | Zero sample loss, continuous ring |
| Tier-1 response | Beat may be partially captured | Full beat always in relay buffer |
| Power during active monitoring | Repeated transients from PLL restarts | Steady low-power baseline |
| Best suited for | Long idle with no signal | Normal sustained cardiac activity |

**Design Rule:** STOP mode is reserved for the scenario where no cardiac signal is detected for an extended window (e.g., device is off-wrist or the user is completely still for >30 s). During active monitoring, the DMA relay buffer approach keeps the CPU in a stable, responsive, low-power state — not asleep.

---

### Integration with the Tier Architecture

The relay buffer acts as the **data spine** between all three tiers:

```
AD8232 ──▶ ADC ──DMA──▶ [Relay Buffer Ring]
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
              Tier 0 HW    Tier 1 reads  Tier 2 reads
              Watchdog     from buffer   same buffer
              monitors     on DMA IRQ    on escalation
              threshold    (low-power    (full CPU speed,
              live          RUN mode)    brief burst)
```

- **Tier 0** (analog watchdog) monitors the live ADC signal threshold in hardware, independent of the relay buffer.
- **Tier 1** reads the most recent beat window from the relay buffer on every DMA half/full-transfer interrupt — CPU stays in low-power RUN.
- **Tier 2** is triggered by Tier 1's escalation flag; it reads the same relay buffer (or a timestamped snapshot) and runs at full CPU speed for the brief duration of the inference.
- **No sample is lost** during any tier transition because DMA continues filling the ring regardless of what the CPU is doing.

---

## 👥 Team Split

| | **Person A — Hardware, Signal & Power** | **Person B — ML & Intelligence** |
|---|---|---|
| **Ownership** | Acquisition, wake-up logic, power state machine | Model design, training, quantization, deployment |
| **Tasks** | Analog front-end (AD8232) → STM32 ADC integration • Analog watchdog config for HW-triggered wake • DMA circular relay buffer setup (double-half-buffer mode) • Power FSM (WFI-idle ↔ Low-Power RUN ↔ Full RUN ↔ STOP) • Dynamic clock/voltage scaling controller • BLE alert transmission | MIT-BIH / PTB-XL preprocessing • Tier-1 lightweight model training • Tier-2 CNN-LSTM training • Quantization via STM32Cube.AI (X-CUBE-AI) • Confidence-gating logic • Relay buffer read interface for ML inference |
| **Deliverable** | Power-optimized firmware + DMA relay pipeline + wake FSM | Two deployed, benchmarked models + escalation logic |

---

## 🛠️ Tech Stack

- **MCU:** STM32 (Cortex-M series, e.g. STM32L4/F4 for low-power modes)
- **ECG Front-End:** AD8232 (or equivalent analog ECG AFE)
- **DMA Mode:** Circular double-buffer, half/full-transfer interrupts
- **ML Deployment:** STM32Cube.AI / X-CUBE-AI for on-device inference
- **Model Training:** Python, TensorFlow/Keras or PyTorch
- **Datasets:** MIT-BIH Arrhythmia Database, PTB-XL
- **Communication:** BLE for alert transmission

---

## 📂 Repository Structure

```
├── firmware/              # STM32 embedded firmware (Person A)
│   ├── power_fsm/         # WFI-idle / Low-Power RUN / Full RUN / STOP state machine
│   ├── adc_watchdog/      # Analog watchdog + continuous ADC config
│   ├── dma_relay_buffer/  # DMA circular buffer setup, IRQ handlers, buffer read API
│   └── ble_transmit/
├── models/                # Training + quantization pipeline (Person B)
│   ├── tier1_screener/
│   ├── tier2_specialist/
│   └── quantized/
├── data/                  # Dataset preprocessing scripts
├── docs/                  # Architecture diagrams, reports
└── README.md
```

---

## 🚀 Getting Started

```bash
# Clone the repo
git clone https://github.com/<your-username>/hierarchical-ecg-stm32.git
cd hierarchical-ecg-stm32

# Set up model training environment
cd models
pip install -r requirements.txt

# Flash firmware (STM32CubeIDE required)
cd ../firmware
# Open project in STM32CubeIDE and build/flash to target board
# DMA + ADC configuration is in firmware/dma_relay_buffer/relay_buf_init.c
```

---

## 📊 Evaluation Metrics

| Metric | Target |
|--------|--------|
| Tier-1 escalation rate | < 5% of beats |
| Tier-2 classification accuracy | > 95% on held-out MIT-BIH set |
| Average current draw (normal sinus, DMA relay mode) | < 3 mA (Low-Power RUN + DMA) |
| Average current draw (idle / off-wrist, STOP mode) | < 10 µA |
| Wake latency (WFI → DMA IRQ → Tier-1 start) | < 1 µs |
| Wake latency (STOP → Active, device reattach) | < 50 ms |
| Zero sample loss during Tier 1 → Tier 2 escalation | Required |

---

## 📄 License

MIT License — feel free to fork and build on this.

---

## 🙌 Acknowledgments

Built as a final-year Biomedical Engineering project exploring low-power, event-driven ML for wearable cardiac monitoring.
