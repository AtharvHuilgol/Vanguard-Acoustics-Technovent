# ARGUS-PHM: Product Requirements Document
## Edge-Optimized Real-Time Structural Vibration PHM for High-Altitude Rotorcraft Drivetrains
### Tata InnoVent Edition 4 — "AI at the Edge" — Aerospace Vertical / TASL

---

## Document Control

| Field | Value |
|---|---|
| Document ID | ARGUS-PRD-001 |
| Version | 1.0 |
| Date | June 2026 |
| Competition | Tata Technologies InnoVent Edition 4 |
| Vertical | Aerospace — Tata Advanced Systems Limited (TASL) |
| Target Platform | Airbus H125 / H125M Final Assembly Line, Hyderabad |
| Status | Final Submission Draft |

---

## 1. Executive Summary & The 30-Second Comprehension Hook

### 1.1 Problem Statement: The Mechanical Failure Cascade

The Epicyclic (Planetary) Reduction Gear Stage of the Main Rotor Gearbox (MRG) is the single
highest-consequence mechanical component in any rotorcraft. It sustains the full cyclic
aerodynamic load of the main rotor blade assembly while transmitting peak torque from the
turboshaft engine to the rotor head. Understanding its failure pathway is the foundational
argument for ARGUS-PHM's existence and must be communicated in the first 60 seconds of
any jury presentation.

**The Failure Cascade — Five-Stage Sequence:**

```
STAGE 1: Micro-pitting initiation
  Cyclic Hertzian contact stress exceeds the surface fatigue limit of case-hardened
  tooth flanks. Micro-pits (5–50 μm diameter) nucleate at asperity contacts on the
  involute tooth profile.

  Vibration signature: Sidebands at (GMF ± N × f_shaft) begin to emerge 15–25 dB
  above the noise floor. Detectable with structure-borne sensing; invisible to
  ground-only HUMS analysis during an ongoing sortie.

         ↓  [10 to several hundred flight hours of undetected propagation]

STAGE 2: Progressive pitting and tooth fracture propagation
  Micro-pits coalesce into macro-pitting. Subsurface crack propagation initiates at
  metallic inclusion sites under rolling-contact fatigue. Tooth root cracks develop
  from stress concentration at the gear root fillet radius.

  Vibration signature: GMF sideband amplitude increases 6–20 dB relative to Stage 1
  baseline. Second and third harmonic distortion becomes measurable. RMS acceleration
  level rises 2–4× above healthy baseline.

         ↓

STAGE 3: Catastrophic tooth loss
  One or more gear teeth fracture completely from the root. Hardened steel debris
  (typically 1–8 mm fragments) circulates in the lubrication system, causing
  secondary erosive and abrasive damage to adjacent gear stages, planet bearings,
  and the ring gear bore.

  Vibration signature: Impulsive shock events at the planet pass frequency.
  Broadband acceleration magnitude increases >40 dB within 100 ms. Non-recoverable
  without immediate shutdown.

         ↓

STAGE 4: MRG Seizure
  Debris-induced bearing failure causes total planetary stage seizure. The main
  rotor shaft loses its driven connection to the engine. No partial degraded mode
  exists — seizure is complete and instantaneous.

         ↓

STAGE 5: Catastrophic Aircraft Loss
  Loss of main rotor drive at altitude. Autorotation entry must be initiated within
  1.5 seconds. At operational altitudes above 4,500 m (Siachen Glacier, Leh, LAC
  forward operating bases), reduced air density lowers main rotor inertia margin
  and severely compresses the successful autorotation envelope. Fatal outcome
  probability approaches certainty above 3,000 m AGL at low forward airspeed.
```

**The window ARGUS-PHM protects:** The interval between Stage 1 and Stage 3 — typically
10 to several hundred flight hours depending on cyclic load amplitude and lubrication
quality — is the actionable detection window. ARGUS-PHM ensures this window is never
allowed to close undetected.

---

### 1.2 Operational Context: TASL Airbus H125 and the Current HUMS Deficiency

**Tata Advanced Systems Limited (TASL)** has established India's first rotorcraft Final
Assembly Line (FAL) for the **Airbus H125 and H125M** in Hyderabad. The H125 is the primary
light utility helicopter in the Indian civil market (mountain rescue, powerline inspection,
Himalayan tourism, filming). The H125M is its militarised variant, targeted at the Indian
Army for reconnaissance, communication relay, and VVIP transport in high-altitude frontier
environments. TASL is producing these airframes on Indian soil, for Indian operational
conditions — and currently, no indigenous AI-native PHM solution exists for them.

**Current Military HUMS Architecture and its Fatal Structural Flaw:**

```
[Onboard MEMS accelerometers / vibration pickups]
          ↓
[Data Acquisition Unit: logs raw binary data to onboard SSD/PCMCIA]
          ↓   ← NO onboard processing. No inference. No real-time output.
[Aircraft completes sortie and lands at base]
          ↓
[Ground crew physically extracts storage media]
          ↓
[Ground station workstation with proprietary HUMS software (Honeywell/Safran)]
          ↓
[Maintenance engineer reviews automated report: minimum 2–4 hour turnaround]
          ↓
[Decision: continue / ground for inspection]
```

This architecture creates an irreducible structural vulnerability: **the 4-to-6-flight-hour
undetected fault propagation window.** Between successive ground-based HUMS analysis
sessions — typically every 1 to 3 sorties — an aircraft accumulates 4 to 6 flight hours
of operational exposure with a developing planetary gear fault completely invisible to
maintenance personnel. In high-tempo military operations at forward operating bases without
diagnostic infrastructure (Leh, Thoise, Partapur, Dzukou), this window regularly extends
to 12–18 flight hours across consecutive operational days.

**ARGUS-PHM eliminates this window entirely:**

| Metric | Ground-Station HUMS (Current) | ARGUS-PHM (Proposed) |
|---|---|---|
| Fault detection latency | 4–6 flight hours | **15 milliseconds** |
| Connectivity requirement | Physical media download | **None — fully offline** |
| Processing location | Ground workstation (Pentium-class) | **RP2040 MCU in cockpit** |
| Detection granularity | Sortie-level | **Sample-level, real-time** |
| Latency improvement factor | Baseline | **~960,000×** |
| Unit cost (hardware) | ₹40–100 lakh (imported Honeywell) | **₹15,000–50,000** |

---

### 1.3 The Unassailable Edge Case: Electronic Warfare and the Death of Cloud

Unlike every automotive edge AI application that will be presented at InnoVent Edition 4,
ARGUS-PHM does not choose the edge because it is faster than cloud. **The cloud is
physically non-existent in the primary operational environment of this aircraft.**

The Indian Army's H125M operates across three classes of connectivity-denied environments:

**Class A — EW-Contested Tactical Zones (LOC / LAC):**
Active adversary electronic warfare platforms on both the Line of Control (Pakistan) and
the Line of Actual Control (China) deploy broadband jamming across L-Band, S-Band, C-Band,
and Ku-Band frequencies. GPS denial, SATCOM disruption, and tactical datalink jamming are
documented operational realities. No streaming PHM architecture can function in this
environment. The node either processes locally or it does not process at all.

**Class B — High-Altitude Terrain Operations (Siachen, >5,400 m):**
Terrain-induced RF shadow zones, extreme cold (-40°C to -55°C) degrading RF front-end
performance, absence of cellular infrastructure, and line-of-sight limitations create
reliable communications blackouts. The Siachen Glacier — the world's highest battlefield —
has no broadband network presence accessible to a rotorcraft operating at altitude.

**Class C — Extended Maritime Operations (Andaman & Nicobar Islands):**
Bandwidth-limited SATCOM links are shared across all operational systems and mission payloads.
PHM telemetry is never the priority payload and is routinely pre-empted. Latency to a
continental cloud data centre from A&N via SATCOM is 600–1,200 ms — rendering real-time
inference physically impossible regardless of link availability.

**Architectural conclusion:** Any system requiring periodic cloud sync, remote inference,
or telemetry offload for PHM functionality will fail in all three primary operating
environments of this aircraft. ARGUS-PHM is architected from first principles for
**permanent, indefinite, fully offline operation**. The RP2040 microcontroller is not a
cost optimisation choice — it is the only physics-compatible processing architecture.

---

## 2. System Architecture & Environmental Constraint Handling

### 2.1 Full Hardware Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  PHYSICAL LAYER — Main Rotor Gearbox Casing (6061-T6 aluminium alloy housing)   │
│                                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐   │
│  │  STMicroelectronics IIS3DWB — Industrial MEMS Accelerometer               │   │
│  │                                                                             │   │
│  │  Sensing modality:  Structure-borne vibration (NOT airborne acoustic)      │   │
│  │  Mounting:          Direct-bonded to external gearbox casing surface,      │   │
│  │                     M2 screw fastening + structural epoxy (Loctite EA9361) │   │
│  │  Axes:              3-axis (X, Y, Z)                                       │   │
│  │  Frequency response: Flat DC → 6 kHz (hardware anti-aliasing at 6 kHz)    │   │
│  │  Output data rate:  26,664 Hz (26.7 kSPS) — configurable                  │   │
│  │  Measurement range: ±16 g (configurable ±2/±4/±8/±16 g)                   │   │
│  │  Noise density:     75 μg/√Hz (ultra-low noise for sub-threshold pitting)  │   │
│  │  Interface:         SPI mode 3, up to 10 MHz, DRDY interrupt pin           │   │
│  │  Operating temp:    −40°C to +85°C (covers all H125 operating envelopes)   │   │
│  │  Supply voltage:    1.71 V to 3.6 V                                        │   │
│  └───────────────────────────────┬───────────────────────────────────────────┘   │
│                                  │ 4-wire SPI + DRDY line (5 conductors total)   │
└──────────────────────────────────┼──────────────────────────────────────────────-┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  EDGE COMPUTE NODE — Primary Prototype Hardware                                  │
│                                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐   │
│  │  Raspberry Pi Pico (RP2040 SoC)                                            │   │
│  │                                                                             │   │
│  │  Core:              Dual ARM Cortex-M0+ at 133 MHz                         │   │
│  │  SRAM:              264 KB (on-chip, no external DRAM latency)             │   │
│  │  Flash:             2 MB (model flatbuffer + firmware)                     │   │
│  │  DMA:               8 channels, supports SPI peripheral → SRAM transfers   │   │
│  │  PIO:               Programmable I/O for deterministic SPI timing          │   │
│  │  Firmware stack:    FreeRTOS 10.5.x + CMSIS-DSP + TFLite Micro            │   │
│  └──────────────────────────────┬────────────────────────────────────────────┘   │
│                                 │                                                 │
│  ┌──────────────────────────────▼────────────────────────────────────────────┐   │
│  │  Output Interface                                                           │   │
│  │  GPIO 0 (Green LED):   System nominal — P(healthy) > 0.85                  │   │
│  │  GPIO 1 (Amber LED):   Caution zone — P(fault) ∈ [0.65, 0.85)             │   │
│  │  GPIO 2 (Red LED):     Fault detected — P(fault) > 0.85 for ≥ 3 cycles    │   │
│  │  GPIO 3 (Buzzer):      Audible alert (Demo Day)                            │   │
│  │  UART TX (115200):     Structured JSON diagnostic log                      │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
                                   │
                    Certification Upgrade Path
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  PRODUCTION HARDWARE — STM32H743ZI (Nucleo-H743ZI2 Development Board)           │
│  ARM Cortex-M7 at 480 MHz, double-precision FPU, 1 MB SRAM, 2 MB flash         │
│  DO-160G environmental qualification target; DO-178C DAL C software path        │
│  Identical TFLite Micro C++ API — zero firmware porting required                │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 FreeRTOS Concurrency Architecture

Four tasks execute concurrently on the dual-core RP2040. Core 0 runs the FreeRTOS
scheduler with Tasks 1–3; Core 1 is dedicated to Task 4 (safety output) via the Pico SDK
`multicore_launch_core1()` API, ensuring safety-critical GPIO control is never pre-empted
by inference operations.

```
Task                   Priority   Stack     Responsibility
─────────────────────────────────────────────────────────────────────────────
vSPI_DataAcq_Task      4 (High)   1024 W    IIS3DWB SPI reads via DMA;
                                            fills ping-pong raw int16_t buffers;
                                            posts buffer pointer to xDataQueue.
                                            Never blocks on inference.

vDSP_Processing_Task   3          2048 W    Pops buffer from xDataQueue;
                                            Hanning window → arm_rfft_fast_f32();
                                            magnitude spectrum → L2-normalize;
                                            posts float32 feature vector to xFeatureQueue.

vML_Inference_Task     2          4096 W    Pops feature vector from xFeatureQueue;
                                            quantize to INT8 → TFLite interpreter;
                                            evaluates confidence thresholds;
                                            posts AlertStruct to xAlertQueue.

vSafety_Output_Task    4 (High)   512 W     Monitors xAlertQueue;
                                            implements N=3 debounce counter;
                                            drives GPIO LEDs and buzzer;
                                            feeds hardware watchdog timer.
─────────────────────────────────────────────────────────────────────────────

Inter-task communication primitives:
  xDataQueue:     Depth 2  |  Item: int16_t* (pointer to filled DMA buffer)
  xFeatureQueue:  Depth 2  |  Item: float32_t[512] (spectral feature vector)
  xAlertQueue:    Depth 8  |  Item: struct AlertStruct {
                                     uint8_t  fault_class;    // 0 = healthy, 1 = fault
                                     float    confidence;     // P(fault), 0.0–1.0
                                     uint32_t timestamp_ms;  // FreeRTOS tick count
                                   }

DMA synchronisation:
  xDmaSemaphore:  Binary semaphore — DMA completion ISR signals vSPI_DataAcq_Task
  Ping-pong:      Buffer A fills while Buffer B is processed by DSP task;
                  swap on DMA complete interrupt — zero CPU cycles consumed
                  during acquisition phase.
```

---

## 3. Functional & Technical Specifications

### 3.1 Data Acquisition Layer

**Sampling Rate Selection and Nyquist Compliance:**

The sampling frequency must satisfy the Nyquist-Shannon criterion for all target frequency
components while remaining within the IIS3DWB's hardware-filtered bandwidth.

```
Target mechanical frequencies (representative H125-class epicyclic):
  Main rotor speed (carrier):           n_c  = 395 RPM = 6.58 Hz
  Input shaft to epicyclic stage:       n_s  ≈ 1,580 RPM = 26.3 Hz  (example 4:1 ratio)
  Gear Mesh Frequency (planet-annulus): f_GMF = Z_annulus × n_c
                                             = 72 teeth × 6.58 Hz ≈ 474 Hz
  3rd harmonic of GMF:                  3 × 474 ≈ 1,422 Hz
  5th harmonic of GMF:                  5 × 474 ≈ 2,370 Hz
  Bearing defect frequencies (BPFI):    ≈ N_b/2 × n_s × (1 + d_b/d_p × cos α) ≈ 130 Hz
  Bearing defect frequencies (BPFO):    ≈ N_b/2 × n_s × (1 − d_b/d_p × cos α) ≈  85 Hz
  Sideband spacing:                     = n_s = 26.3 Hz (shaft rotation rate)

Selected sampling rate:   f_s = 26,664 Hz (IIS3DWB maximum ODR)
Nyquist frequency:        f_N = 13,332 Hz
Margin above 5th GMF harmonic:  13,332 / 2,370 = 5.6× margin
IIS3DWB hardware AA filter:    −3 dB at 6 kHz → aliasing physically eliminated
Effective acquisition bandwidth: DC to 6 kHz
```

Note: Actual H125 MRG gear tooth counts are proprietary Airbus Helicopters data.
Production deployment uses confirmed OEM specifications. Representative values above are
used for prototype design and are characteristic of comparable epicyclic stages.

**SPI Protocol Configuration (Pico C SDK):**

```c
// spi_init.c — IIS3DWB hardware initialisation

#include "pico/stdlib.h"
#include "hardware/spi.h"
#include "hardware/dma.h"
#include "hardware/irq.h"

#define PIN_MISO  16
#define PIN_MOSI  19
#define PIN_SCK   18
#define PIN_CS    17
#define PIN_DRDY  20

void iis3dwb_spi_init(void) {
    // 10 MHz SPI clock — within IIS3DWB 10 MHz spec, exceeds minimum for stable reads
    spi_init(spi0, 10 * 1000 * 1000);
    spi_set_format(spi0, 8, SPI_CPOL_1, SPI_CPHA_1, SPI_MSB_FIRST);  // Mode 3

    gpio_set_function(PIN_MISO, GPIO_FUNC_SPI);
    gpio_set_function(PIN_MOSI, GPIO_FUNC_SPI);
    gpio_set_function(PIN_SCK,  GPIO_FUNC_SPI);
    gpio_init(PIN_CS);
    gpio_set_dir(PIN_CS, GPIO_OUT);
    gpio_put(PIN_CS, 1);  // Deselect

    // DRDY interrupt — fires on every completed sample conversion
    gpio_set_irq_enabled_with_callback(PIN_DRDY, GPIO_IRQ_EDGE_RISE,
                                       true, &drdy_isr_handler);
}

void iis3dwb_configure(void) {
    // CTRL1_XL: ODR = 26664 Hz, ±16 g full scale, LPF enabled
    iis3dwb_write_reg(0x10, 0xA0);
    // CTRL4_C: DRDY pin active, SPI only
    iis3dwb_write_reg(0x13, 0x04);
    // FIFO_CTRL4: Continuous FIFO mode
    iis3dwb_write_reg(0x0A, 0x06);
}
```

**DMA Ping-Pong Buffer — CPU Starvation Prevention:**

```c
// dma_buffers.c

#define BUFFER_SIZE      2048   // samples per FFT window
#define BYTES_PER_SAMPLE    2   // int16_t

static int16_t dma_buf_A[BUFFER_SIZE];   // Buffer A: active fill
static int16_t dma_buf_B[BUFFER_SIZE];   // Buffer B: DSP processes while A fills
static volatile uint8_t active_buf = 0;  // 0 = A filling, 1 = B filling

// DMA completion ISR: swaps buffers and signals acquisition task via semaphore
void __isr dma_complete_handler(void) {
    dma_hw->ints0 = (1u << DMA_CHAN_SPI_RX);  // Clear interrupt
    active_buf ^= 1;                           // Atomic buffer swap
    // Reconfigure DMA to fill newly active buffer
    dma_channel_set_write_addr(DMA_CHAN_SPI_RX,
                               active_buf ? dma_buf_B : dma_buf_A, true);
    // Signal vSPI_DataAcq_Task: ready buffer available
    BaseType_t xHigherPriTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDmaSemaphore, &xHigherPriTaskWoken);
    portYIELD_FROM_ISR(xHigherPriTaskWoken);
}
```

---

### 3.2 DSP & Pre-Processing Layer

The complete time-domain to frequency-domain transformation executes within the
`vDSP_Processing_Task` using ARM CMSIS-DSP library functions, which are optimised for
fixed-point and single-precision floating-point arithmetic on Cortex-M architectures.

**Full DSP Pipeline:**

```
Step 1: Integer-to-float conversion
  arm_q15_to_float(raw_int16_buffer, float_buffer, BUFFER_SIZE);
  // Converts signed 16-bit ADC codes to normalised float32 in range [-1.0, +1.0]
  // Scales by 1/32768 per CMSIS-DSP convention

Step 2: Hanning window application (spectral leakage suppression)
  Hanning formula: w[n] = 0.5 × (1 − cos(2πn / (N−1)),  n = 0 … N−1
  Implementation:  arm_mult_f32(float_buffer, hanning_coefficients, windowed, N);
  Coefficients:    Precomputed at startup, stored in flash (8,192 bytes for N=2048)

  Engineering rationale:
    - Rectangular window:  13.3 dB sidelobe suppression
    - Hanning window:      31.5 dB sidelobe suppression
    - Critical for resolving GMF sidebands that are 20–40 dB below the main GMF
      peak — a rectangular window's spectral leakage would bury these sidebands

Step 3: Real-valued FFT
  arm_rfft_fast_instance_f32 fft_instance;
  arm_rfft_fast_init_f32(&fft_instance, BUFFER_SIZE);  // Init once at startup
  arm_rfft_fast_f32(&fft_instance, windowed, fft_output, 0);
  // Output: 1024 complex pairs (interleaved real, imag) covering DC to f_s/2
  // Frequency resolution: Δf = f_s / N = 26664 / 2048 = 13.02 Hz per bin
  // This resolves the 26.3 Hz sideband spacing with 2-bin precision

Step 4: Magnitude spectrum
  arm_cmplx_mag_f32(fft_output, magnitude, BUFFER_SIZE / 2);
  // |X[k]| = sqrt(real[k]² + imag[k]²) for k = 0 … N/2−1
  // Hardware multiply-accumulate on M0+ core: ~4 cycles per sample

Step 5: Feature vector extraction and normalisation
  // Truncate to first 512 bins (covers DC to 6.65 kHz — full IIS3DWB bandwidth)
  // Apply L2 normalisation to prevent magnitude drift from speed variation:
  //   sum_sq = 0; for k=0..511: sum_sq += magnitude[k]²;
  //   norm = sqrt(sum_sq);
  //   for k=0..511: feature_vector[k] = magnitude[k] / norm;
  // Result: float32_t feature_vector[512] — range-independent spectral shape

Step 6: Float32 to INT8 quantisation for TFLite input tensor
  // Apply scale and zero_point from TFLite quantisation parameters:
  //   q = round(f / scale + zero_point), clipped to [-128, 127]
  arm_float_to_q7(feature_vector, input_tensor_data, 512);
```

**Memory Footprint (RP2040, 264 KB SRAM):**

```
Component                              Bytes    Notes
──────────────────────────────────────────────────────────────────────────
FreeRTOS kernel + heap                  6,144   TCBs, queues, timer structs
Task stacks (4 tasks × per-spec)       30,720   1024+2048+4096+512 words × 4 B
DMA ping-pong buffers (×2)             16,384   2 × 2048 × int16_t
Float32 conversion buffer               8,192   2048 × float32_t
Hanning window coefficients             8,192   Precomputed; flash-loadable
FFT instance + internal twiddle        ~4,000   CMSIS-DSP arm_rfft_fast_f32
FFT complex output buffer               8,192   1024 complex × 8 bytes
Magnitude spectrum buffer               4,096   1024 × float32_t
L2-normalised feature vector            2,048   512 × float32_t
TFLite tensor arena                    65,536   64 KB — model activations
INT8 model flatbuffer                  20,480   ~20 KB for described architecture
Queue storage + AlertStruct             1,024
Miscellaneous / alignment               5,000
──────────────────────────────────────────────────────────────────────────
TOTAL ESTIMATED                       179,008   68% of 264 KB — 84 KB headroom
```

---

### 3.3 TinyML Edge Inference Layer

**Model Architecture — INT8 Quantised 1D Convolutional Neural Network:**

Input to the model is the 512-bin L2-normalised magnitude spectrum from Step 5 of the DSP
pipeline, quantised to INT8. The architecture is deliberately constrained to minimise
parameter count while preserving the ability to resolve multi-scale spectral features
(coarse GMF envelope and fine sideband structure).

```
Layer                    Output Shape    Parameters    Notes
────────────────────────────────────────────────────────────────────────────────
Input                    (512,)          0             INT8, spectral magnitude vector
Reshape                  (512, 1)        0             1D signal for convolution
Conv1D(16, k=8, s=2)     (253, 16)       144           Captures coarse GMF texture
BatchNorm + ReLU6        (253, 16)       64            ReLU6 preferred for INT8 stability
MaxPool1D(pool=2)        (126, 16)       0
Conv1D(32, k=4, s=2)     (62, 32)      2,080           Captures fine sideband patterns
BatchNorm + ReLU6        (62, 32)        128
GlobalAveragePooling1D   (32,)           0             Collapses spatial dimension;
                                                       invariant to speed variation
Dense(16) + ReLU6        (16,)           528
Dense(2) + Softmax       (2,)            34            [P(healthy), P(fault)]
────────────────────────────────────────────────────────────────────────────────
Total parameters:        ~2,978
INT8 flatbuffer size:    ~18 KB (estimated)
Tensor arena required:   ~48 KB
Target inference time:   ≤ 15 ms on RP2040 @ 133 MHz (benchmark during build)
```

**Post-Training Quantisation (Full INT8) — TFLite Conversion Script:**

```python
# quantise_model.py

import tensorflow as tf
import numpy as np

# Load trained Keras model
model = tf.keras.models.load_model('argus_phm_float32.h5')

def representative_data_gen():
    """Provide 200 representative healthy-state spectra for calibration."""
    for sample in healthy_calibration_dataset[:200]:
        yield [sample.astype(np.float32)[np.newaxis, :]]  # Shape: (1, 512)

# Configure INT8 full-integer quantisation
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations          = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type   = tf.int8
converter.inference_output_type  = tf.int8

tflite_model = converter.convert()

# Save binary flatbuffer
with open('argus_phm_int8.tflite', 'wb') as f:
    f.write(tflite_model)

# Export as C array for embedding in firmware
with open('model.h', 'w') as f:
    hex_array = ', '.join(f'0x{byte:02x}' for byte in tflite_model)
    f.write(f'alignas(8) const unsigned char g_model_data[] = {{{hex_array}}};\n')
    f.write(f'const unsigned int g_model_data_size = {len(tflite_model)};\n')

print(f"Model size: {len(tflite_model):,} bytes ({len(tflite_model)/1024:.1f} KB)")
```

**TFLite Micro C++ Inference Invocation (Pico firmware):**

```cpp
// ml_inference.cpp

#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/schema/schema_generated.h"
#include "model.h"

// Statically allocate tensor arena — no dynamic heap allocation
constexpr int kTensorArenaSize = 48 * 1024;
alignas(16) static uint8_t tensor_arena[kTensorArenaSize];

// Register only the operations used by this model (minimises flash footprint)
static tflite::MicroMutableOpResolver<5> resolver;
resolver.AddConv2D();
resolver.AddDepthwiseConv2D();
resolver.AddMaxPool2D();
resolver.AddFullyConnected();
resolver.AddSoftmax();

// Build interpreter — performed once at system startup
const tflite::Model* model = tflite::GetModel(g_model_data);
tflite::MicroInterpreter interpreter(model, resolver, tensor_arena,
                                     kTensorArenaSize);
interpreter.AllocateTensors();

// Per-inference function
AlertStruct run_inference(const int8_t* feature_vector_int8) {
    TfLiteTensor* input = interpreter.input(0);
    memcpy(input->data.int8, feature_vector_int8, 512);

    uint32_t t_start = to_ms_since_boot(get_absolute_time());
    TfLiteStatus status = interpreter.Invoke();
    uint32_t t_elapsed = to_ms_since_boot(get_absolute_time()) - t_start;

    TfLiteTensor* output = interpreter.output(0);
    float p_healthy = (output->data.int8[0] - output->params.zero_point)
                      * output->params.scale;
    float p_fault   = (output->data.int8[1] - output->params.zero_point)
                      * output->params.scale;

    return AlertStruct {
        .fault_class   = (p_fault > 0.65f) ? 1u : 0u,
        .confidence    = p_fault,
        .timestamp_ms  = to_ms_since_boot(get_absolute_time()),
        .inference_ms  = t_elapsed   // Log actual inference time during testing
    };
}
```

---

## 4. Non-Functional Requirements & Jury Defense Protocols

### 4.1 Model Uncertainty Handling — Graceful Degradation Protocol

A safety-advisory system in an airborne context cannot emit binary outputs without a
confidence qualification layer. The following three-tier protocol prevents false positives
during legitimate flight transients (autorotation training, run-up, hard landing) while
maintaining high sensitivity to genuine gear deterioration.

```
TIER 1: NOMINAL OPERATION
  Condition:      P(healthy) > 0.85 for current inference cycle
  LED output:     Green ON, Amber OFF, Red OFF
  Pilot advisory: None
  Log entry:      {timestamp, class:"HEALTHY", conf_healthy:P, rms:current_rms}
  Action:         Continue normal operation

TIER 2: CAUTION — Ambiguous Confidence Region
  Condition:      P(fault) ∈ [0.65, 0.85)
  LED output:     Amber ON
  Pilot advisory: No immediate crew alert; maintenance crew alerted on ground
  Debounce logic: Increment consecutive_caution_counter.
    If counter ≥ 3 → escalate to Tier 3.
    If single Tier 2 event returns to Tier 1 → clear counter, log as transient.
  Rationale:      Single-sample exceedances from aerodynamic transients, 
                  landing shock, or high-g manoeuvres are expected. The 3-cycle 
                  requirement (3 × 76.7 ms = 230 ms) exceeds the duration of 
                  all identified flight transient events.

TIER 3: FAULT DETECTED — Mandatory Ground Inspection Required
  Condition:      P(fault) > 0.85 for ≥ 3 consecutive inference cycles
  LED output:     Red ON, continuous buzzer
  Pilot advisory: "DRIVETRAIN PHM ALERT — LAND AS SOON AS PRACTICABLE"
                  (UART message formatted for ARINC 429 / MIL-STD-1553 future integration)
  Log entry:      {timestamp, class:"FAULT", conf_fault:P, run_hours, consecutive_count}
  Reset:          Requires explicit ground maintenance crew power-cycle acknowledgement.
                  Prevents inadvertent reset by aircrew during post-fault hover.

SENSOR HEALTH MONITOR (parallel, always running):
  Condition A:    |RMS acceleration| > 50 g → sensor saturation or mechanical shock
  Condition B:    |RMS acceleration| < 0.01 g → sensor disconnection or power fault
  Action:         Amber LED — SYSTEM DEGRADED; suspend fault classification output
  Rationale:      Prevents a silent "healthy" classification when the sensor is
                  inoperative — the most dangerous failure mode for a safety monitor.

WATCHDOG TIMER:
  Hardware watchdog on RP2040 must be fed by vSafety_Output_Task every 500 ms.
  If task deadlocks or inference hangs beyond 500 ms: MCU resets, system restarts
  in safe state (Amber LED, suspended classification, UART reset log entry).
```

---

### 4.2 Certification Roadmap — Jury Defense Protocol

This section provides precise, pre-prepared answers to the three certification questions
that will be asked by any technically competent jury member.

**Question 1: "ESP32 / Raspberry Pi Pico are consumer parts. They're not certifiable
for flight. How does your system ever fly on a real aircraft?"**

```
Answer:
This prototype validates the signal processing algorithm and TinyML model architecture
on COTS hardware. The production certification path uses the STM32H743 family from
STMicroelectronics — ARM Cortex-M7 at 480 MHz with double-precision FPU, radiation
tolerant metal-can packaging, and extended temperature range. The TFLite Micro C++
interpreter is hardware-agnostic: it compiles and runs identically on any ARMv7-M target.
The firmware DMA, FreeRTOS, and CMSIS-DSP calls map directly to the STM32 HAL with
near-zero porting effort. The algorithm is the IP. The silicon is a procurement decision.
```

**Question 2: "What's DO-160G and what does your system need to pass?"**

```
DO-160G — Environmental Conditions and Test Procedures for Airborne Equipment (RTCA):

Section  |  Test Category           |  Relevance to ARGUS-PHM
─────────────────────────────────────────────────────────────────────────────────
Sec. 4   |  Temperature/Altitude    |  −55°C to +70°C operational; 40,000 ft altitude
Sec. 7   |  Vibration               |  Must survive MRG-level vibration (ironic for PHM)
Sec. 15  |  Magnetic Effect         |  Minimal magnetic emissions from low-power MCU
Sec. 16  |  Power Input             |  28 VDC aircraft bus compatibility (voltage regulator)
Sec. 20  |  RF Susceptibility       |  Must not malfunction under EW jamming environment;
         |                          |  critical given LOC/LAC operational context
Sec. 21  |  RF Emissions            |  Must not emit RF that interferes with avionics
Sec. 25  |  ESD                     |  Electrostatic discharge hardening
```

**Question 3: "What's DO-178C and what DAL level does your software need?"**

```
DO-178C — Software Considerations in Airborne Systems and Equipment Certification:

Design Assurance Level (DAL) for ARGUS-PHM:

DAL A (Catastrophic): Required if software failure directly causes fatal crash.
                      NOT applicable — ARGUS-PHM does not actuate any flight control.

DAL B (Hazardous):    Required if failure contributes to hazardous condition.
                      Potentially applicable if output is integrated with master caution.

DAL C (Major):        Required if failure causes significant reduction in safety margins.
                      APPLICABLE: ARGUS-PHM is an advisory-only monitoring system.
                      DAL C mandates: requirements traceability, structural code coverage
                      to Modified Condition/Decision Coverage (MC/DC), formal test plans.

Production path:      DAL C toolchain using IAR Embedded Workbench with DO-178C
                      certification kit on STM32H7. The TFLite Micro C++ runtime has
                      documented deterministic execution properties suitable for DAL C.

The TinyML layer requires additional argument: IEEE 2894-2024 (Recommended Practice
for Organizational Governance of Artificial Intelligence) guidance for ML in
safety-critical systems is the emerging framework. ARGUS-PHM's binary classification
output is verifiable against a fixed confidence threshold — simpler to assure than
generative or reinforcement learning architectures.
```

---

### 4.3 Sustainability & Financial Impact

**Condition-Based Maintenance vs. Time-Based Overhaul:**

| Impact Category | Metric | Quantification |
|---|---|---|
| MRG scheduled overhaul cost (H125 class) | Per aircraft | ₹25–40 lakh |
| Unscheduled MRG removal (missed fault) | Per event | ₹80–200 lakh (parts + downtime + lost operational availability) |
| TBO extension with CBM | Industry average (ATA MSG-3) | 20–35% |
| ARGUS-PHM hardware cost (COTS PoC) | Per aircraft | ₹15,000–50,000 |
| Imported HUMS equivalent (Honeywell/Safran) | Per aircraft | ₹40–100 lakh |
| **Cost displacement ratio** | **Atmanirbhar** | **96–99.5%** |
| Annual savings (50-aircraft fleet, CBM TBO extension) | Fleet-level | ₹8–14 crore |
| Indian military rotorcraft fleet (Cheetah/Chetak/Dhruv) | Addressable | 330+ airframes, zero with domestic AI PHM |

**Atmanirbhar Bharat Strategic Alignment:**

India imports 100% of military rotorcraft HUMS technology from Honeywell (US), Safran
(France), and Kongsberg (Norway). ARGUS-PHM is the first demonstration of an AI-native,
software-defined PHM system built on hardware manufacturable under India's semiconductor
and electronics programmes. It directly addresses:

- MoD Defence Procurement Policy 2020: 68% domestic procurement target
- HAL Dhruv ALH fleet (330+ airframes): zero currently equipped with onboard AI PHM
- TASL H125 production: zero domestic HUMS contracts awarded at programme inception
- PM GATI Shakti: strategic self-reliance in defence critical systems

**Environmental Sustainability:**

- Each avoided premature MRG replacement saves ~15 kg of high-grade machined alloy
  (AMS 6265 case-hardened steel, 9310 alloy gear material) from premature scrapping.
- For a 50-aircraft fleet over 10 years with 30 avoided premature replacements:
  ~450 kg alloy diverted from waste; ~36 tonne CO₂e avoided (avoided manufacturing energy).
- Cadmium/chromium plating waste from overhaul shop processes reduced proportionally
  to avoided strip-and-replate cycles.

---

## 5. Multi-Vertical Scalability Matrix

The ARGUS-PHM signal processing pipeline is hardware-agnostic and mechanically domain-
agnostic. The entire SPI → DMA → FreeRTOS → Hanning → FFT → TFLite stack is re-usable
across any rotating mechanical system by re-training the model on component-specific
vibration data. No firmware changes are required — only the `model.h` flatbuffer is
replaced.

### Stage 1: Intra-Aircraft (12–18 months post-Demo Day)

| Component | Target Fault Mode | Sensor Mounting Point | Model Re-training Delta |
|---|---|---|---|
| Tail rotor drive shaft hanger bearings | BPFO/BPFI outer/inner race defects | IIS3DWB on hanger bearing housing | Retrain classifier head on hanger vibration data; identical firmware |
| Swashplate pitch link bearings | Fretting corrosion, looseness | Contact accelerometer at pitch control pivot | Adjust frequency range of interest in FFT bin selection |
| Main gearbox input bevel gear stage | Tooth pitting at high-speed end | Bonded AE sensor at input flange | Recalculate GMF for bevel geometry; identical DSP pipeline |
| Engine power turbine bearings | BPFI signatures at ~50,000 RPM shaft | Miniaturised IIS3DWB at gearbox input | Upsample ODR considerations; GMF in kHz range |

### Stage 2: Cross-Vehicle — TASL UAVs and Loitering Munitions (18–36 months)

TASL's **ALS-50 Loitering Munition** and indigenous cargo drone programs use high-duty-
cycle BLDC motors in propulsion roles where bearing and winding health is mission-critical.
The pipeline adaptation is minor:

```
BLDC Motor PHM for UAV propulsion:
  Sensor:         IIS3DWB bonded to motor housing (or ADXL355 for lower cost)
  Target freqs:   Electrical commutation frequency = RPM/60 × pole_pairs
                  Bearing defects: BPFI/BPFO at shaft rotation harmonics
  Fault classes:  Bearing outer race defect | Winding imbalance | Rotor eccentricity
  Hardware:       Identical RP2040/ESP32 — smaller model reduces SRAM requirement
  Firmware delta: Zero changes — only model.h replaced
  Training data:  CWRU dataset (bearing) + synthetic BLDC winding imbalance data
```

### Stage 3: Cross-Vertical Fusion — Tata Motors EV Drivetrain (36–60 months)

This is the commercially transformative scalability argument. The identical hardware and
firmware pipeline can be deployed for Tata Motors' EV drivetrain PHM by substituting
only the trained model — demonstrating hardware platform reuse across the full Tata Group.

```
H125 MRG (Aerospace)          →    Tata Nexon EV / Tiago EV (Automotive)
──────────────────────────────────────────────────────────────────────────────
Epicyclic planetary gearbox   →    Single-speed helical reduction gearbox
Carrier: 395 RPM (rotor)      →    Motor: 2,000–12,000 RPM operating range
GMF ≈ 474 Hz                  →    GMF = Z_gear × n_motor / 60 (range: 1–10 kHz)
Fault: gear tooth pitting     →    Fault: bearing fatigue, gear tooth wear, rattle
Sensor: IIS3DWB at 26.7 kHz   →    Identical hardware — no procurement change
FFT: N = 2048 buffer          →    N = 2048 or 4096 for higher-freq EV gearbox
TFLite model: H125 Aerospace  →    Retrain same 1D-CNN on EV gearbox dataset
Deployment: cockpit-mounted   →    BMS enclosure or rear axle housing
FreeRTOS: Identical           →    Identical
Firmware: Identical           →    Identical
```

**Business case for the jury:** ARGUS-PHM's architecture means TASL can manufacture a
single, unified embedded PHM hardware module — domestically sourced, certified under one
qualification programme — and deploy it across rotorcraft, UAV propulsion, and Tata Motors
EV drivetrains by distributing only a new trained model flatbuffer. One hardware platform.
One firmware stack. Three multi-billion-dollar addressable markets within the Tata Group.

---

*ARGUS-PHM Product Requirements Document — Version 1.0 — Tata InnoVent Edition 4*
*Classification: Engineering Competition Submission*
