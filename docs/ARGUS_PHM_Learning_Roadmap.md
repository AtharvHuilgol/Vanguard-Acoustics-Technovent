# ARGUS-PHM: Technical Stack & Engineering Learning Roadmap
## Student Team Curriculum — Atharv, Ganesh, Swaraj, Gaurav
### Tata InnoVent Edition 4 — "AI at the Edge" — Aerospace Vertical

---

## Roadmap Overview

```
PHASE TIMELINE (June 2026 → January 2027 Demo Day)

June–July       ██████  Module 1: Embedded Systems, SPI & FreeRTOS Core
July–August     █████   Module 2: Embedded DSP — FFT & Spectral Analysis
August–October  ███████ Module 3: TinyML Pipeline — Datasets, Training & Quantisation
October–Dec     ████████ Module 4: Hardware Test-Bench & Fault Induction
November–Jan    █████   Integration, Benchmarking & Demo Day Preparation

Team task split recommendation:
  Atharv:  Module 1 lead (embedded firmware, FreeRTOS, SPI driver)
  Ganesh:  Module 2 lead (DSP, FFT, signal processing mathematics)
  Swaraj:  Module 3 lead (Python ML pipeline, TFLite conversion, model training)
  Gaurav:  Module 4 lead (physical rig construction, fault induction, data collection)
  All:     Integration week — every team member runs the full stack end-to-end
```

---

## Module 1: Embedded Systems, High-Speed SPI & FreeRTOS Core

**Owner: Atharv — Estimated duration: 4–5 weeks**
**Objective:** Achieve reliable, non-blocking, DMA-driven acquisition of IIS3DWB vibration
data into a ping-pong buffer architecture, orchestrated by a properly prioritised FreeRTOS
task graph. No data loss permitted at 26,664 SPS.

---

### 1.1 Development Environment Setup

**Hardware required:**
- Raspberry Pi Pico (RP2040) — primary target (~₹400)
- STMicroelectronics IIS3DWB evaluation board (STEVAL-MKI195V1) — ~₹1,500–2,500
  (alternative: source the IIS3DWB bare IC and solder to a breakout PCB)
- USB–UART adapter for UART debug output (CP2102 or CH340 module — ~₹200)
- Logic analyser — Saleae Logic 8 or DSLogic Plus (~₹3,000–8,000). Non-negotiable
  for debugging SPI timing issues at 10 MHz.

**Software toolchain:**

```bash
# Install the Raspberry Pi Pico C/C++ SDK on Ubuntu/Debian
sudo apt update && sudo apt install -y cmake gcc-arm-none-eabi \
    libnewlib-arm-none-eabi build-essential git python3

git clone https://github.com/raspberrypi/pico-sdk.git --recurse-submodules
export PICO_SDK_PATH=$(pwd)/pico-sdk

# Alternatively: use PlatformIO (VS Code extension)
# Install VS Code → Extensions → PlatformIO IDE
# New project → Board: Raspberry Pi Pico → Framework: Arduino or pico-sdk
# The pico-sdk framework gives full access to hardware/spi.h and hardware/dma.h
# which are needed for low-level SPI+DMA control not available in Arduino HAL.

# Install VS Code extensions needed:
# - PlatformIO IDE
# - C/C++ (Microsoft)
# - Cortex-Debug (for OpenOCD/SWD debugging via Pico's debug pins)
```

**Recommended project folder structure:**
```
argus-phm-firmware/
├── CMakeLists.txt
├── pico_sdk_import.cmake
├── src/
│   ├── main.cpp
│   ├── spi_init.c / spi_init.h        ← Module 1 deliverable
│   ├── dma_buffers.c / dma_buffers.h  ← Module 1 deliverable
│   ├── freertos_tasks.c               ← Module 1 deliverable
│   ├── dsp_processing.c               ← Module 2 deliverable
│   └── ml_inference.cpp               ← Module 3 deliverable
├── lib/
│   ├── FreeRTOS-Kernel/               ← git submodule
│   ├── CMSIS_5/                       ← git submodule (DSP lib)
│   └── tflite-micro/                  ← git submodule
└── model/
    ├── model.h                        ← Module 3 deliverable (generated)
    └── argus_phm_int8.tflite
```

---

### 1.2 FreeRTOS Core Concepts — Mastery Checklist

FreeRTOS is not an operating system in the Linux sense. It is a real-time scheduler and
inter-task communication library. The concepts below are the complete set needed for this
project. Master them in order.

**Concept 1: Task creation and priority levels**

```c
// freertos_tasks.c
#include "FreeRTOS.h"
#include "task.h"

// Task handles — stored globally so tasks can be inspected/notified
TaskHandle_t hSpiAcqTask   = NULL;
TaskHandle_t hDspTask      = NULL;
TaskHandle_t hMlTask       = NULL;
TaskHandle_t hSafetyTask   = NULL;

void app_main(void) {
    // Priorities: higher number = higher priority in FreeRTOS
    // configMAX_PRIORITIES is typically 5 (0–4). Set it to 5 in FreeRTOSConfig.h.
    xTaskCreate(vSPI_DataAcq_Task,    "SPI_ACQ",    1024, NULL, 4, &hSpiAcqTask);
    xTaskCreate(vDSP_Processing_Task, "DSP_PROC",   2048, NULL, 3, &hDspTask);
    xTaskCreate(vML_Inference_Task,   "ML_INF",     4096, NULL, 2, &hMlTask);
    xTaskCreate(vSafety_Output_Task,  "SAFETY_OUT",  512, NULL, 4, &hSafetyTask);

    vTaskStartScheduler();  // Hands control to FreeRTOS — never returns
    for (;;);               // Should never reach here
}

// CRITICAL FreeRTOSConfig.h settings for this project:
// configUSE_PREEMPTION            = 1  (pre-emptive scheduling)
// configUSE_TIME_SLICING          = 0  (no equal-priority time slicing — avoid jitter)
// configUSE_TICKLESS_IDLE         = 0  (keep tick running for precise timing)
// configTOTAL_HEAP_SIZE           = (128 * 1024)  — 128 KB for FreeRTOS heap
// configMINIMAL_STACK_SIZE        = 256  (words, not bytes)
// configCHECK_FOR_STACK_OVERFLOW  = 2  (enable stack overflow detection — essential)
```

**Concept 2: Queues — the correct mechanism for inter-task data flow**

```c
// Queues are the ONLY safe way to pass data between tasks.
// Never use global variables without a mutex — it causes data races.

#include "queue.h"

// Global queue handles — created before tasks start
QueueHandle_t xDataQueue    = NULL;  // Raw ADC buffer pointers
QueueHandle_t xFeatureQueue = NULL;  // DSP output feature vectors
QueueHandle_t xAlertQueue   = NULL;  // ML classification results

// In app_main(), before xTaskCreate():
xDataQueue    = xQueueCreate(2, sizeof(int16_t*));     // 2 buffer pointers
xFeatureQueue = xQueueCreate(2, sizeof(float) * 512);  // 2 feature vectors (8 KB total)
xAlertQueue   = xQueueCreate(8, sizeof(AlertStruct));  // 8 alert structs (ring buffer)

// Producer (SPI task) — sends pointer to filled buffer:
int16_t* filled_buffer = get_filled_dma_buffer();
xQueueSend(xDataQueue, &filled_buffer, 0);  // 0 = don't block if full (drop oldest)

// Consumer (DSP task) — receives and processes:
int16_t* raw_buffer;
if (xQueueReceive(xDataQueue, &raw_buffer, portMAX_DELAY) == pdTRUE) {
    // Process raw_buffer → feature_vector
    float feature_vector[512];
    run_dsp_pipeline(raw_buffer, feature_vector);
    xQueueSend(xFeatureQueue, feature_vector, 0);
}
```

**Concept 3: Binary semaphores — synchronising ISR to task**

```c
// Binary semaphore is the correct primitive for ISR → Task signalling.
// NEVER call blocking FreeRTOS functions (vTaskDelay, xQueueReceive) from an ISR.
// Use the FromISR variants exclusively inside interrupt handlers.

#include "semphr.h"

SemaphoreHandle_t xDmaSemaphore = NULL;
// Create before tasks: xDmaSemaphore = xSemaphoreCreateBinary();

// DMA completion ISR — called when a full 2048-sample buffer has been transferred
void __isr dma_irq_handler(void) {
    dma_hw->ints0 = (1u << DMA_CHANNEL);  // Clear interrupt flag

    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xDmaSemaphore, &xHigherPriorityTaskWoken);
    // If the semaphore woke a higher-priority task, yield immediately
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// SPI Acquisition Task — blocks on semaphore, wakes only on DMA completion
void vSPI_DataAcq_Task(void* pvParams) {
    for (;;) {
        // Block here — zero CPU usage while waiting for DMA to complete
        xSemaphoreTake(xDmaSemaphore, portMAX_DELAY);
        // DMA is done. Swap buffers, post to queue.
        swap_ping_pong_buffers();
        int16_t* ready_buf = get_ready_buffer();
        xQueueSend(xDataQueue, &ready_buf, 0);
    }
}
```

**Concept 4: Stack overflow detection — mandatory during development**

```c
// In FreeRTOSConfig.h: configCHECK_FOR_STACK_OVERFLOW = 2
// This calls the hook below when a stack overflow is detected.
// Without this, a stack overflow corrupts adjacent memory silently — impossible to debug.

void vApplicationStackOverflowHook(TaskHandle_t xTask, char* pcTaskName) {
    // Log the offending task name to UART before halting
    printf("STACK OVERFLOW: task %s\n", pcTaskName);
    taskDISABLE_INTERRUPTS();
    for (;;);  // Halt — force watchdog reset
}
// If this fires: increase the stack size for the named task in xTaskCreate()
```

---

### 1.3 IIS3DWB SPI Driver — Register-Level Interfacing

The IIS3DWB communicates over SPI Mode 3 (CPOL=1, CPHA=1). All register reads require
a 7-bit register address with the MSB set to 1 (read bit). Writes set MSB to 0.

```c
// iis3dwb_driver.c

#define IIS3DWB_READ_BIT  0x80

// Internal SPI transfer — always CS low during transaction
static void spi_transfer(uint8_t* tx, uint8_t* rx, size_t len) {
    gpio_put(PIN_CS, 0);
    spi_write_read_blocking(spi0, tx, rx, len);
    gpio_put(PIN_CS, 1);
}

// Write a single register
void iis3dwb_write_reg(uint8_t reg, uint8_t value) {
    uint8_t buf[2] = {reg & 0x7F, value};  // MSB=0 for write
    spi_transfer(buf, NULL, 2);
}

// Read a single register (verify during init for sanity check)
uint8_t iis3dwb_read_reg(uint8_t reg) {
    uint8_t tx[2] = {reg | IIS3DWB_READ_BIT, 0x00};
    uint8_t rx[2] = {0};
    spi_transfer(tx, rx, 2);
    return rx[1];
}

// Verify WHO_AM_I register (should return 0x7B for IIS3DWB)
bool iis3dwb_verify_identity(void) {
    uint8_t who_am_i = iis3dwb_read_reg(0x0F);
    if (who_am_i != 0x7B) {
        printf("IIS3DWB not found. WHO_AM_I = 0x%02X (expected 0x7B)\n", who_am_i);
        return false;
    }
    printf("IIS3DWB verified. WHO_AM_I = 0x7B\n");
    return true;
}

// Read one acceleration sample (6 bytes: X_L, X_H, Y_L, Y_H, Z_L, Z_H)
void iis3dwb_read_xyz(int16_t* ax, int16_t* ay, int16_t* az) {
    uint8_t tx[7] = {0x28 | IIS3DWB_READ_BIT};  // OUTX_L_XL register
    uint8_t rx[7] = {0};
    spi_transfer(tx, rx, 7);
    *ax = (int16_t)((rx[2] << 8) | rx[1]);
    *ay = (int16_t)((rx[4] << 8) | rx[3]);
    *az = (int16_t)((rx[6] << 8) | rx[5]);
}
// Note: For continuous DMA acquisition, use FIFO burst read (register 0x3A, FIFO_DATA_OUT)
// rather than single-sample reads — eliminates per-sample CS toggling overhead.
```

**Module 1 Exit Criteria (do not proceed to Module 2 until all pass):**
- [ ] `iis3dwb_verify_identity()` returns `true` on hardware
- [ ] DMA ping-pong fills 2,048 samples at 26,664 SPS without loss (verify with logic analyser)
- [ ] FreeRTOS stack usage for each task is <80% of allocated stack (check via `uxTaskGetStackHighWaterMark()`)
- [ ] Watchdog fires correctly when `vSafety_Output_Task` is artificially stalled >500 ms
- [ ] No stack overflow hook triggered during 10-minute continuous run

---

## Module 2: Embedded Digital Signal Processing

**Owner: Ganesh — Estimated duration: 3–4 weeks**
**Objective:** Implement the complete time-domain to frequency-domain transformation
pipeline on the RP2040 using CMSIS-DSP. Produce a 512-bin L2-normalised magnitude
spectrum from raw IIS3DWB int16_t samples within a single DSP task execution cycle,
demonstrable on a logic analyser by measuring task execution time.

---

### 2.1 CMSIS-DSP Library Integration

CMSIS-DSP (Cortex Microcontroller Software Interface Standard — Digital Signal Processing)
is ARM's open-source library of optimised fixed-point and floating-point DSP primitives
for all Cortex-M architectures. It is the correct tool for FFT on embedded systems —
do not implement FFT from scratch.

**Integration into Pico SDK CMakeLists.txt:**

```cmake
# CMakeLists.txt — add after pico_sdk_init()

# Clone CMSIS_5 into lib/CMSIS_5 first:
# git clone https://github.com/ARM-software/CMSIS_5.git lib/CMSIS_5 --depth 1

set(CMSIS_DSP_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/lib/CMSIS_5/CMSIS/DSP)

# Include only the source files needed — avoids bloating flash
add_library(cmsis_dsp STATIC
    ${CMSIS_DSP_ROOT}/Source/TransformFunctions/arm_rfft_fast_f32.c
    ${CMSIS_DSP_ROOT}/Source/TransformFunctions/arm_rfft_fast_init_f32.c
    ${CMSIS_DSP_ROOT}/Source/TransformFunctions/arm_cfft_f32.c
    ${CMSIS_DSP_ROOT}/Source/TransformFunctions/arm_cfft_radix8_f32.c
    ${CMSIS_DSP_ROOT}/Source/CommonTables/arm_const_structs.c
    ${CMSIS_DSP_ROOT}/Source/CommonTables/arm_common_tables.c
    ${CMSIS_DSP_ROOT}/Source/ComplexMathFunctions/arm_cmplx_mag_f32.c
    ${CMSIS_DSP_ROOT}/Source/BasicMathFunctions/arm_mult_f32.c
    ${CMSIS_DSP_ROOT}/Source/SupportFunctions/arm_q15_to_float.c
    ${CMSIS_DSP_ROOT}/Source/SupportFunctions/arm_float_to_q7.c
    ${CMSIS_DSP_ROOT}/Source/StatisticsFunctions/arm_rms_f32.c
)

target_include_directories(cmsis_dsp PUBLIC
    ${CMSIS_DSP_ROOT}/Include
    lib/CMSIS_5/CMSIS/Core/Include
)

target_compile_definitions(cmsis_dsp PUBLIC
    ARM_MATH_CM0PLUS    # RP2040 is Cortex-M0+; enables M0+ optimisations
    __FPU_PRESENT=0     # M0+ has no hardware FPU; library uses SW float
)

target_link_libraries(your_project cmsis_dsp)
```

---

### 2.2 Hanning Window — Theory and Implementation

**Why windowing is not optional:**

The FFT assumes the analysed signal is periodic within the observation window. For a
vibration signal from a gearbox — which is nearly but not exactly periodic — this
assumption is violated. Without a window function, energy from a frequency bin "leaks"
into adjacent bins, creating a smearing effect that buries the low-amplitude sidebands
we need to detect. The Hanning window suppresses this leakage by tapering the signal
to zero at both ends of the buffer.

```
Spectral leakage comparison for detecting a −30 dB sideband:
  Rectangular window:  Sideband buried under 13.3 dB sidelobe of adjacent GMF peak
  Hanning window:      31.5 dB sidelobe suppression — sideband visible above noise floor
  Conclusion:          Hanning window is mandatory for this application
```

**Precompute coefficients at startup (saves runtime computation):**

```c
// dsp_processing.c

#include "arm_math.h"

#define FFT_SIZE  2048

static float hanning_window[FFT_SIZE];
static arm_rfft_fast_instance_f32 fft_instance;

void dsp_init(void) {
    // Compute Hanning coefficients once at startup, store in SRAM
    for (int n = 0; n < FFT_SIZE; n++) {
        hanning_window[n] = 0.5f * (1.0f - arm_cos_f32(
                            2.0f * PI * (float)n / (float)(FFT_SIZE - 1)));
    }
    // Initialise FFT instance (computes twiddle factors — expensive, do once)
    arm_rfft_fast_init_f32(&fft_instance, FFT_SIZE);
}
```

---

### 2.3 Full DSP Pipeline Implementation

```c
// dsp_processing.c  (continued)

static float float_buffer[FFT_SIZE];
static float windowed[FFT_SIZE];
static float fft_output[FFT_SIZE];   // Complex output: FFT_SIZE/2 complex pairs
static float magnitude[FFT_SIZE / 2];

void run_dsp_pipeline(const int16_t* raw_buffer, float* feature_vector_out) {

    // --- Step 1: Integer to float (Q1.15 → normalised float32) ---
    arm_q15_to_float((q15_t*)raw_buffer, float_buffer, FFT_SIZE);
    // Each sample is divided by 32768.0f — output range [-1.0, +1.0]

    // --- Step 2: Hanning window multiplication ---
    arm_mult_f32(float_buffer, hanning_window, windowed, FFT_SIZE);
    // Element-wise multiply: windowed[n] = float_buffer[n] × w[n]
    // This costs ~2048 multiply-accumulate operations — <1 ms on RP2040

    // --- Step 3: Real FFT (in-place, N=2048) ---
    arm_rfft_fast_f32(&fft_instance, windowed, fft_output, 0);
    // Output layout: [Re[0], Re[N/2], Re[1], Im[1], Re[2], Im[2], ...]
    // DC bin: fft_output[0] (real only)
    // Nyquist bin: fft_output[1] (real only)
    // All other bins: interleaved real/imag pairs

    // --- Step 4: Magnitude spectrum ---
    arm_cmplx_mag_f32(fft_output, magnitude, FFT_SIZE / 2);
    // |X[k]| = sqrt(Re[k]² + Im[k]²) for k = 0...N/2-1
    // Note: CMSIS arm_cmplx_mag_f32 uses an approximation; for maximum accuracy
    // at demonstration use arm_cmplx_mag_squared_f32 + sqrtf(), but the CMSIS
    // version is 3× faster and sufficient for classification purposes.

    // --- Step 5: Extract first 512 bins (DC to 6.65 kHz) ---
    // The IIS3DWB bandwidth is limited to 6 kHz by hardware filter.
    // Bins above index 511 contain only noise from filter rolloff — discard.
    // Bin k corresponds to frequency: f[k] = k × Δf = k × 13.02 Hz

    // --- Step 6: L2 normalisation (speed-invariant spectral shape) ---
    // Without normalisation, the spectrum amplitude scales with rotor speed.
    // A faster rotor produces higher vibration energy at all frequencies.
    // L2 normalisation extracts the spectral SHAPE (fault signature) independently
    // of speed, making the classifier speed-invariant.
    float sum_sq = 0.0f;
    for (int k = 0; k < 512; k++) sum_sq += magnitude[k] * magnitude[k];
    float norm = sqrtf(sum_sq) + 1e-8f;  // Epsilon prevents divide-by-zero
    for (int k = 0; k < 512; k++) feature_vector_out[k] = magnitude[k] / norm;

    // Post this feature_vector_out[512] to xFeatureQueue for ML inference
}
```

**Visualising your FFT output during development (essential for debugging):**

```python
# On your PC — connect Pico via UART and log the feature vector.
# Plot with matplotlib to verify the GMF peak appears at the correct bin.

import serial, struct, numpy as np, matplotlib.pyplot as plt

FS  = 26664   # Sampling rate (Hz)
N   = 2048    # FFT size
DF  = FS / N  # Frequency resolution: 13.02 Hz/bin

port = serial.Serial('/dev/ttyUSB0', 115200)
data = port.read(512 * 4)  # 512 float32 = 2048 bytes
spectrum = np.frombuffer(data, dtype=np.float32)
freqs = np.arange(512) * DF

plt.figure(figsize=(14, 5))
plt.plot(freqs, 20 * np.log10(spectrum + 1e-10), linewidth=0.8)
plt.xlabel('Frequency (Hz)')
plt.ylabel('Magnitude (dB, normalised)')
plt.title('ARGUS-PHM: Live Gear Vibration Spectrum — Healthy State')
plt.axvline(x=474, color='r', linestyle='--', label='GMF (474 Hz)')
plt.axvline(x=948, color='orange', linestyle='--', label='2× GMF')
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('healthy_spectrum.png', dpi=150)
plt.show()
# Expected result: strong peak at GMF, clean noise floor, no sideband structure
# After fault induction: sideband peaks appear at GMF ± 26.3 Hz
```

**Module 2 Exit Criteria:**
- [ ] FFT output plotted from real IIS3DWB data — gear mesh frequency peak visible at expected bin
- [ ] Hanning window effect demonstrated: plot with/without windowing; show sidelobe reduction
- [ ] L2 normalisation verified: run at two different motor speeds; confirm spectral shape is invariant
- [ ] `run_dsp_pipeline()` measured execution time < 30 ms on RP2040 @ 133 MHz (using GPIO toggle + oscilloscope)
- [ ] No floating-point undefined behaviour (verify with `-fsanitize=undefined` on host before deploying)

---

## Module 3: TinyML Pipeline Optimisation & Datasets

**Owner: Swaraj — Estimated duration: 5–6 weeks**
**Objective:** Train a compact 1D-CNN classifier in Python/TensorFlow on CWRU bearing
fault data, validate it, convert to INT8 TFLite Micro flatbuffer, deploy on RP2040,
and achieve measurable inference within 15 ms producing correct fault/healthy classification.

---

### 3.1 Python Environment Setup

```bash
# Create isolated Python environment
conda create -n argus_phm python=3.11 -y
conda activate argus_phm

# Core packages
pip install tensorflow==2.14.0          # TFLite converter included
pip install tensorflow-model-optimization # For quantisation-aware training if needed
pip install numpy scipy scikit-learn
pip install matplotlib seaborn pandas
pip install scipy                       # signal.welch for PSD cross-validation
pip install pyserial                    # For collecting data from Pico via UART

# Verify GPU (optional but speeds training)
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

---

### 3.2 CWRU Bearing Dataset — Acquisition and Pre-Processing

The Case Western Reserve University (CWRU) Bearing Fault Dataset is the standard public
benchmark dataset for bearing condition monitoring research. It was collected at 12 kHz
and 48 kHz sampling rates from drive-end and fan-end test bearings with controlled fault
diameters machined onto inner race, outer race, and ball surfaces.

**Download and parse:**

```python
# cwru_loader.py

import os, urllib.request, scipy.io, numpy as np
from pathlib import Path

# CWRU data is hosted as .mat files — download the 12kHz drive-end files
CWRU_BASE_URL = "https://engineering.case.edu/bearingdatacenter/download-data-file"
# Note: the actual direct download URLs vary; use the CWRU website or a Kaggle mirror.
# Recommended: search "CWRU Bearing Dataset Kaggle" for a structured ZIP download.

def load_mat_file(filepath: str, key: str = 'DE_time') -> np.ndarray:
    """Load drive-end (DE) time-domain vibration signal from .mat file."""
    mat = scipy.io.loadmat(filepath)
    # Keys vary by file: 'X097_DE_time', 'X105_DE_time', etc.
    # Find the correct key programmatically:
    de_key = [k for k in mat.keys() if 'DE_time' in k][0]
    return mat[de_key].flatten()

# Example: load 4 classes
data_files = {
    'healthy':     'Normal_0.mat',
    'inner_race':  'IR007_0.mat',   # 0.007" inner race fault
    'outer_race':  'OR007@6_0.mat', # 0.007" outer race fault at 6 o'clock
    'ball':        'B007_0.mat',    # 0.007" ball fault
}

raw_signals = {label: load_mat_file(f) for label, f in data_files.items()}
print({label: sig.shape for label, sig in raw_signals.items()})
# Output: {'healthy': (121265,), 'inner_race': (121265,), ...}
```

**Signal segmentation — creating fixed-length training samples:**

```python
# feature_extraction.py

import numpy as np
from scipy.signal import get_window

FS_CWRU   = 12000     # CWRU dataset sampling rate (Hz)
FS_TARGET =  6664     # Effective bandwidth of IIS3DWB prototype (after 6kHz AA filter)
FFT_SIZE  = 2048      # Must match firmware FFT_SIZE

def downsample_to_target(signal: np.ndarray, fs_in: int, fs_out: int) -> np.ndarray:
    """Resample CWRU 12kHz data to match prototype sensor effective rate.
    
    Engineering note: The IIS3DWB outputs at 26664 Hz but hardware-filters at 6 kHz.
    CWRU was collected at 12 kHz (Nyquist = 6 kHz). The frequency content is matched —
    downsample CWRU to 6664 Hz (half the IIS3DWB rate) so that the spectral bin
    alignment is consistent between training data and embedded inference data.
    
    Alternative: Use CWRU at 48 kHz and downsample to 26664 Hz to match exactly.
    The 48 kHz files have more usable bandwidth but require more memory to process.
    """
    from scipy.signal import resample_poly
    from math import gcd
    g = gcd(fs_out, fs_in)
    return resample_poly(signal, fs_out // g, fs_in // g)

def extract_spectral_features(segment: np.ndarray, fft_n: int = FFT_SIZE) -> np.ndarray:
    """Apply Hanning window, FFT, magnitude, L2-normalise. Returns 512-dim vector.
    
    This Python function MUST produce identical output to the C firmware DSP pipeline.
    Run both on the same input signal and compare outputs during validation.
    """
    assert len(segment) == fft_n, f"Segment length {len(segment)} != FFT size {fft_n}"
    window  = np.hanning(fft_n)
    windowed = segment * window
    spectrum = np.abs(np.fft.rfft(windowed))[:512]   # First 512 bins only
    norm     = np.linalg.norm(spectrum) + 1e-8
    return (spectrum / norm).astype(np.float32)

def segment_and_extract(signal: np.ndarray, label: int,
                        window_size: int = 2048, hop_size: int = 512) -> tuple:
    """Sliding window segmentation with 75% overlap.
    
    hop_size = 512 means each new window advances 512 samples (75% overlap).
    Overlap increases dataset size and improves generalisation.
    """
    features, labels = [], []
    for start in range(0, len(signal) - window_size, hop_size):
        segment  = signal[start : start + window_size].astype(np.float32)
        features.append(extract_spectral_features(segment))
        labels.append(label)
    return np.array(features), np.array(labels)

# Build full dataset
all_features, all_labels = [], []
label_map = {'healthy': 0, 'inner_race': 1, 'outer_race': 1, 'ball': 1}
# Collapse to binary: 0 = healthy, 1 = any fault (matches ARGUS-PHM output)

for label_name, signal in raw_signals.items():
    signal_ds = downsample_to_target(signal, FS_CWRU, FS_TARGET)
    feat, lab = segment_and_extract(signal_ds, label_map[label_name])
    all_features.append(feat)
    all_labels.append(lab)

X = np.vstack(all_features)  # Shape: (N_samples, 512)
y = np.concatenate(all_labels)
print(f"Dataset: {X.shape[0]} samples, {X.shape[1]} features")
print(f"Class balance: {np.mean(y==0)*100:.1f}% healthy, {np.mean(y==1)*100:.1f}% fault")
```

---

### 3.3 Model Construction and Training

```python
# model_train.py

import tensorflow as tf
from tensorflow import keras
import numpy as np
from sklearn.model_selection import train_test_split

# Split dataset: 70% train, 15% validation, 15% test
X_train, X_temp, y_train, y_temp = train_test_split(X, y, test_size=0.3,
                                                      stratify=y, random_state=42)
X_val, X_test, y_val, y_test = train_test_split(X_temp, y_temp, test_size=0.5,
                                                  stratify=y_temp, random_state=42)

# Expand dims for Conv1D: (N, 512) → (N, 512, 1)
X_train = X_train[..., np.newaxis]
X_val   = X_val[..., np.newaxis]
X_test  = X_test[..., np.newaxis]

def build_argus_phm_model(input_shape=(512, 1), n_classes=2) -> keras.Model:
    """
    Architecture designed for INT8 quantisation compatibility:
    - ReLU6 instead of ReLU (bounded activations — quantises cleanly to INT8)
    - BatchNorm BEFORE activation (improves quantised model accuracy)
    - GlobalAveragePooling instead of Flatten (fewer parameters, speed-invariant)
    - No Dropout (not supported in TFLite Micro without custom ops)
    """
    inputs = keras.Input(shape=input_shape, name='spectral_input')

    x = keras.layers.Conv1D(16, kernel_size=8, strides=2, padding='valid',
                             use_bias=False, name='conv1')(inputs)
    x = keras.layers.BatchNormalization(name='bn1')(x)
    x = keras.layers.Activation('relu6', name='act1')(x)
    x = keras.layers.MaxPool1D(pool_size=2, name='pool1')(x)

    x = keras.layers.Conv1D(32, kernel_size=4, strides=2, padding='valid',
                             use_bias=False, name='conv2')(x)
    x = keras.layers.BatchNormalization(name='bn2')(x)
    x = keras.layers.Activation('relu6', name='act2')(x)

    x = keras.layers.GlobalAveragePooling1D(name='gap')(x)

    x = keras.layers.Dense(16, activation='relu6', name='dense1')(x)
    outputs = keras.layers.Dense(n_classes, activation='softmax', name='output')(x)

    return keras.Model(inputs, outputs, name='ARGUS_PHM_v1')

model = build_argus_phm_model()
model.summary()
# Expected: ~2,978 total parameters; ~12 KB float32 model

model.compile(
    optimizer=keras.optimizers.Adam(learning_rate=1e-3),
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

callbacks = [
    keras.callbacks.EarlyStopping(monitor='val_loss', patience=10,
                                   restore_best_weights=True),
    keras.callbacks.ReduceLROnPlateau(monitor='val_loss', factor=0.5,
                                       patience=5, min_lr=1e-5),
    keras.callbacks.ModelCheckpoint('argus_phm_float32.h5', save_best_only=True)
]

history = model.fit(X_train, y_train,
                    validation_data=(X_val, y_val),
                    epochs=100,
                    batch_size=64,
                    callbacks=callbacks)

# Evaluate on held-out test set
loss, acc = model.evaluate(X_test, y_test, verbose=0)
print(f"\nTest accuracy: {acc:.4f} ({acc*100:.2f}%)")
# Target: >95% accuracy on CWRU test set before proceeding to quantisation
```

---

### 3.4 Post-Training Quantisation — Full INT8 Conversion

```python
# quantise_and_export.py

import tensorflow as tf
import numpy as np

# Load best model from training
model = tf.keras.models.load_model('argus_phm_float32.h5')

# Representative dataset generator — used for INT8 calibration
# Provide ~200 healthy-state samples to calibrate the activation range statistics
# The quantisation scheme maps float32 activations to INT8 using these statistics
def representative_data_gen():
    calibration_samples = X_train[y_train == 0][:200]  # Healthy samples only
    for sample in calibration_samples:
        yield [sample[np.newaxis, :].astype(np.float32)]  # Shape: (1, 512, 1)

# Configure converter for full INT8 quantisation
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type  = tf.int8   # Input tensor will be INT8
converter.inference_output_type = tf.int8   # Output tensor will be INT8

tflite_model_int8 = converter.convert()

# Validate INT8 accuracy (must remain >93% — <2% drop from float32 acceptable)
interpreter = tf.lite.Interpreter(model_content=tflite_model_int8)
interpreter.allocate_tensors()
input_details  = interpreter.get_input_details()
output_details = interpreter.get_output_details()

in_scale     = input_details[0]['quantization'][0]
in_zero_pt   = input_details[0]['quantization'][1]
out_scale    = output_details[0]['quantization'][0]
out_zero_pt  = output_details[0]['quantization'][1]

correct = 0
for i in range(len(X_test)):
    sample_q = np.round(X_test[i] / in_scale + in_zero_pt).astype(np.int8)
    interpreter.set_tensor(input_details[0]['index'], sample_q[np.newaxis])
    interpreter.invoke()
    out_q = interpreter.get_tensor(output_details[0]['index'])[0]
    p_healthy = (float(out_q[0]) - out_zero_pt) * out_scale
    p_fault   = (float(out_q[1]) - out_zero_pt) * out_scale
    predicted = 1 if p_fault > p_healthy else 0
    correct += (predicted == y_test[i])

int8_accuracy = correct / len(X_test)
print(f"INT8 model accuracy: {int8_accuracy:.4f} ({int8_accuracy*100:.2f}%)")
print(f"Accuracy drop from float32: {(acc - int8_accuracy)*100:.2f}%")
print(f"INT8 model size: {len(tflite_model_int8):,} bytes ({len(tflite_model_int8)/1024:.1f} KB)")

# Save binary
with open('argus_phm_int8.tflite', 'wb') as f:
    f.write(tflite_model_int8)

# Export as C header for embedding in firmware
with open('../src/model/model.h', 'w') as f:
    f.write('#pragma once\n')
    f.write('#ifdef __cplusplus\nextern "C" {\n#endif\n\n')
    hex_vals = ', '.join(f'0x{b:02x}' for b in tflite_model_int8)
    f.write(f'alignas(8) const unsigned char g_model_data[] = {{{hex_vals}}};\n')
    f.write(f'const unsigned int g_model_data_size = {len(tflite_model_int8)};\n\n')
    f.write('#ifdef __cplusplus\n}\n#endif\n')

print("model.h written — copy to firmware src/model/ directory")
```

---

### 3.5 TFLite Micro Runtime Integration in Firmware

```cpp
// ml_inference.cpp — complete inference task implementation

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "model/model.h"

// Static tensor arena — never heap-allocate in embedded TFLite
constexpr size_t kTensorArenaSize = 48 * 1024;  // 48 KB
alignas(16) static uint8_t tensor_arena[kTensorArenaSize];

// Interpreter and resolver — constructed once, reused every inference cycle
static tflite::MicroMutableOpResolver<6> resolver;
static tflite::MicroInterpreter* interpreter_ptr = nullptr;

void ml_inference_init(void) {
    // Register only operations present in the model (reduces flash usage)
    resolver.AddConv2D();
    resolver.AddDepthwiseConv2D();
    resolver.AddMaxPool2D();
    resolver.AddFullyConnected();
    resolver.AddSoftmax();
    resolver.AddReshape();

    const tflite::Model* model = tflite::GetModel(g_model_data);
    if (model->version() != TFLITE_SCHEMA_VERSION) {
        // Schema mismatch — wrong TFLite version; rebuild tflite-micro submodule
        panic("TFLite schema version mismatch");
    }

    static tflite::MicroInterpreter interpreter(model, resolver,
                                                tensor_arena, kTensorArenaSize);
    interpreter.AllocateTensors();
    interpreter_ptr = &interpreter;

    // Log actual tensor arena usage (run once and record for PRD memory table)
    size_t used = interpreter_ptr->arena_used_bytes();
    printf("TFLite arena used: %zu bytes of %zu KB\n", used, kTensorArenaSize / 1024);
}

void vML_Inference_Task(void* pvParams) {
    ml_inference_init();

    float feature_vector[512];
    TfLiteTensor* input  = interpreter_ptr->input(0);
    TfLiteTensor* output = interpreter_ptr->output(0);
    float in_scale    = input->params.scale;
    int   in_zero_pt  = input->params.zero_point;
    float out_scale   = output->params.scale;
    int   out_zero_pt = output->params.zero_point;

    for (;;) {
        // Block until DSP task provides a feature vector
        if (xQueueReceive(xFeatureQueue, feature_vector, portMAX_DELAY) != pdTRUE)
            continue;

        // Quantise float32 feature vector to INT8 (match Python quantise script)
        for (int k = 0; k < 512; k++) {
            int q = (int)roundf(feature_vector[k] / in_scale) + in_zero_pt;
            input->data.int8[k] = (int8_t)(q < -128 ? -128 : q > 127 ? 127 : q);
        }

        // Run inference and measure time
        uint32_t t0 = to_ms_since_boot(get_absolute_time());
        interpreter_ptr->Invoke();
        uint32_t t1 = to_ms_since_boot(get_absolute_time());

        // Dequantise output
        float p_healthy = (float(output->data.int8[0]) - out_zero_pt) * out_scale;
        float p_fault   = (float(output->data.int8[1]) - out_zero_pt) * out_scale;

        AlertStruct alert = {
            .fault_class  = (p_fault > 0.65f) ? 1u : 0u,
            .confidence   = p_fault,
            .timestamp_ms = t0,
            .inference_ms = t1 - t0   // Log this — target is ≤ 15 ms
        };
        printf("{\"ts\":%lu, \"cls\":%d, \"conf\":%.3f, \"inf_ms\":%lu}\n",
               alert.timestamp_ms, alert.fault_class,
               alert.confidence, alert.inference_ms);

        xQueueSend(xAlertQueue, &alert, 0);
    }
}
```

**Module 3 Exit Criteria:**
- [ ] Float32 model accuracy on CWRU test set: ≥ 95%
- [ ] INT8 model accuracy on CWRU test set: ≥ 93% (≤ 2% drop from float32)
- [ ] `model.h` successfully compiles into firmware without linker errors
- [ ] `interpreter->AllocateTensors()` succeeds with kTensorArenaSize = 48 KB
- [ ] Measured inference time logged to UART: ≤ 15 ms per inference cycle
- [ ] Python `extract_spectral_features()` and C `run_dsp_pipeline()` produce matching FFT output for identical input (cross-validate with numpy vs. CMSIS-DSP)

---

## Module 4: Hardware Test-Bench Prototyping & Transfer Learning

**Owner: Gaurav — Estimated duration: 6–8 weeks (overlaps with Module 3)**
**Objective:** Construct a low-cost motorised gear test rig, induce controlled physical
faults in the gear train, collect vibration datasets from the IIS3DWB, and use transfer
learning to adapt the CWRU-pretrained model to the rig's specific noise profile, achieving
reliable live fault classification on Demo Day.

---

### 4.1 Test-Bench Design and Construction

**Design philosophy:** The rig must be robust enough to run continuously for 4–6 hours
during data collection and repeatable enough to produce consistent baseline spectra.
Low-cost construction is a feature, not a compromise — it demonstrates the accessibility
of the solution.

**Bill of Materials:**

```
Component                               Source               Approx. Cost
──────────────────────────────────────────────────────────────────────────
High-torque DC gearmotor (12V, 30 RPM)  Robu.in / Amazon     ₹400–800
  (provides the drive shaft + internal gearing)
Plastic or metal spur gear pair         HobbyKing / local     ₹200–600
  (large ring gear + smaller pinion, >20 teeth each)
Motor driver (L298N or DRV8833)        Robu.in               ₹150
12V DC power supply (2A)               Local electronics      ₹300
Aluminium extrusion frame (30×30mm)    Robu.in               ₹400
Shaft coupling + shaft collar          Robu.in               ₹200
Ball bearing housing (pillow block)    Robu.in               ₹250–400
IIS3DWB eval board (STEVAL-MKI195V1)   Mouser / ST direct     ₹1,500–2,500
  OR bare IC + custom breakout PCB     JLCPCB (₹200 PCB)     ₹600–1,000
Epoxy adhesive (Araldite/Loctite)      Hardware store         ₹200
Raspberry Pi Pico                      Robu.in               ₹400
──────────────────────────────────────────────────────────────────────────
TOTAL ESTIMATED                                               ₹5,000–8,000
```

**Assembly steps:**

```
1. Mount gearmotor on aluminium extrusion frame using M3 bolts.
2. Attach driven gear (ring gear) to motor output shaft via shaft collar.
3. Mount driven pinion gear on a separate shaft supported by pillow block bearings.
4. Couple the two gear shafts so they mesh cleanly — minimal runout.
5. Mount IIS3DWB on the pillow block bearing housing using two-component epoxy:
   - Surface prep: clean with isopropyl alcohol, light abrasion with 400-grit paper.
   - Apply Araldite Standard or Loctite EA 9461 structural epoxy.
   - Cure at room temperature for 24 hours before running motor.
   - CRITICAL: The sensor must be rigidly bonded — any looseness kills high-frequency
     signal fidelity and introduces mechanical noise at resonances of the mount.
6. Connect IIS3DWB SPI lines to Pico via shielded twisted-pair cable (<30 cm).
   Use a ground plane around SPI traces — motor PWM switching couples noise into
   unshielded SPI lines at 10 MHz and causes corrupted readings.
7. Run motor driver PWM at 20 kHz minimum — below 20 kHz causes audible noise and
   mechanical vibration at PWM frequency that contaminates the vibration spectrum.
```

---

### 4.2 Baseline Data Collection Protocol

Before inducing any faults, collect a comprehensive healthy baseline. This is the most
important dataset — the model must have a thorough characterisation of normal operation
across the full speed and load range.

```
Baseline collection protocol:
  Speed range:     30%–100% PWM duty cycle (5 speed points)
  Duration:        10 minutes per speed point (continuous)
  Load conditions: Unloaded shaft + light friction load (rubber band brake)
  Temperature:     Ambient only (add temperature compensation in production)
  Repeat:          3 separate sessions, different days (captures day-to-day variability)
  Total baseline:  5 speeds × 10 min × 26664 SPS × 3 sessions
                   ≈ 240 million samples ≈ compressed to ~15,000 segmented windows

File naming convention:
  healthy_speed30pct_session1_20261001.npy
  healthy_speed50pct_session1_20261001.npy
  ... etc.
```

**Automated data collection script (run on PC, reads UART from Pico):**

```python
# collect_data.py

import serial, numpy as np, time
from pathlib import Path

def collect_vibration_windows(port: str, n_windows: int, label: str,
                               save_dir: str = 'data/raw') -> None:
    Path(save_dir).mkdir(parents=True, exist_ok=True)
    ser = serial.Serial(port, 115200, timeout=5)
    windows = []
    print(f"Collecting {n_windows} windows, label: {label}")

    for i in range(n_windows):
        # Pico sends 512 float32 values (feature vector) as binary or JSON
        raw = ser.read(512 * 4)  # 2048 bytes
        if len(raw) < 512 * 4:
            print(f"Warning: short read at window {i}")
            continue
        feature_vec = np.frombuffer(raw, dtype=np.float32).copy()
        windows.append(feature_vec)
        if i % 100 == 0:
            print(f"  {i}/{n_windows} collected")

    arr = np.array(windows)
    filename = f"{save_dir}/{label}_{int(time.time())}.npy"
    np.save(filename, arr)
    print(f"Saved {arr.shape} to {filename}")
    ser.close()
```

---

### 4.3 Fault Induction — Controlled Damage Methods

**Method 1: Gear tooth notch (tooth root crack simulation)**

This is the recommended primary fault for demonstration. A small notch cut into the root
of one gear tooth simulates the crack initiation site from which tooth fracture propagates.

```
Procedure:
1. Remove the driven gear from the shaft.
2. Secure gear in a bench vice — protect tooth flanks with soft jaw covers.
3. Using a triangular needle file (file width < 1.5 mm): file a notch 0.5–1.0 mm
   deep at the ROOT of ONE tooth only, centred on the tooth width.
4. Photograph the notch for documentation (jury will want to see the physical defect).
5. De-burr gently with fine emery cloth — do not round over the notch edges.
6. Reinstall gear and collect fault dataset immediately.

Expected vibration signature:
  - Impulsive event once per shaft revolution (when notched tooth engages)
  - In frequency domain: energy appears at shaft rotation frequency and harmonics
  - Sidebands at GMF ± N × f_shaft become detectable
  - RMS acceleration increases 2–5× above healthy baseline
```

**Method 2: Bearing debris contamination (abrasive wear simulation)**

```
Procedure:
1. Open the pillow block bearing housing.
2. Mix 0.1–0.2 grams of dry valve grinding compound (silicon carbide paste, 
   particle size 15–25 μm) into the bearing grease.
3. Apply contaminated grease to bearing race surfaces.
4. Reassemble and run for 2–5 minutes — the abrasive particles cause micro-pitting 
   on the bearing races.
5. STOP motor. Remove bearing, clean thoroughly, reinstall fresh bearing.
   (The debris contamination creates the fault signature in the collected data;
    you do not want the bearing to fail completely during Demo Day.)

Expected signature:
  - BPFO/BPFI frequency peaks appear in spectrum
  - Broadband noise floor rises 3–8 dB
```

**Safety warnings for fault induction:**
- Always wear safety glasses when filing metal — gear tooth chips are sharp.
- Run the motor behind a polycarbonate shield during first operation after fault induction.
- Never touch rotating components — the gear train runs at ≥30 RPM with ≥2 N·m torque.
- Disconnect motor power before touching any mechanical components.

---

### 4.4 Transfer Learning Protocol — CWRU Model to Custom Rig Data

The CWRU-trained model has learned spectral representations of bearing faults at a specific
speed (1797 RPM) with specific bearing geometry. Your test rig runs at a different speed
with different gear geometry. Direct deployment will produce degraded accuracy. Transfer
learning fine-tunes the pretrained model to your rig's specific acoustic environment using
a small amount of custom data — dramatically reducing the data collection burden.

```python
# transfer_learning.py

import tensorflow as tf
import numpy as np

# Load pre-trained CWRU model
base_model = tf.keras.models.load_model('argus_phm_float32.h5')
base_model.summary()

# Strategy: freeze convolutional feature extractor, retrain classifier head only.
# The conv layers have learned general spectral fault representations (GMF harmonics,
# sideband patterns). The dense layers adapt to your rig's specific noise floor and
# scaling. This requires only ~200–500 custom samples, not thousands.

# Freeze everything up to and including the GlobalAveragePooling layer
for layer in base_model.layers:
    if layer.name in ['conv1', 'bn1', 'act1', 'pool1', 'conv2', 'bn2', 'act2', 'gap']:
        layer.trainable = False
    else:
        layer.trainable = True  # dense1, output — only these update

# Verify frozen layers
for layer in base_model.layers:
    print(f"{layer.name}: trainable={layer.trainable}")

# Load custom rig data (collected in Module 4.2 and 4.3)
# Recommended: 300 healthy windows + 300 fault windows from your rig = 600 total
X_custom = np.load('data/processed/custom_rig_features.npy')  # Shape: (600, 512, 1)
y_custom  = np.load('data/processed/custom_rig_labels.npy')   # Shape: (600,)

# Split custom data: 80% train, 20% validation
from sklearn.model_selection import train_test_split
X_tr, X_vl, y_tr, y_vl = train_test_split(X_custom, y_custom,
                                            test_size=0.2, stratify=y_custom,
                                            random_state=42)

# Fine-tune with a low learning rate — prevents catastrophic forgetting of CWRU knowledge
base_model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-4),  # 10× smaller than initial
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

history_ft = base_model.fit(
    X_tr, y_tr,
    validation_data=(X_vl, y_vl),
    epochs=30,          # Short — converges quickly with frozen feature extractor
    batch_size=32,
    callbacks=[
        tf.keras.callbacks.EarlyStopping(monitor='val_loss', patience=8,
                                          restore_best_weights=True)
    ]
)

fine_tuned_acc = base_model.evaluate(X_vl, y_vl, verbose=0)[1]
print(f"Fine-tuned validation accuracy on custom rig: {fine_tuned_acc*100:.2f}%")
# Target: ≥ 90% accuracy on custom rig data before Demo Day
# If accuracy < 85%: collect more fault data at the specific demo speed

# Re-export to INT8 TFLite after fine-tuning
base_model.save('argus_phm_finetuned.h5')
# Then re-run quantise_and_export.py with the fine-tuned model
# The classifier head weights are now adapted to your rig — INT8 conversion gives
# the final model.h that ships to Demo Day
```

---

### 4.5 Demo Day Readiness Checklist

The following conditions must be met and verified **one week before** Demo Day.
Do not arrive at Demo Day with unresolved items from this list.

**Hardware:**
- [ ] IIS3DWB responds to `WHO_AM_I` query (returns 0x7B) on the demo power supply
- [ ] DMA acquisition runs without data loss for 60 continuous minutes
- [ ] Gear test bench runs at demo speed for 60 minutes without mechanical failure
- [ ] All solder joints inspected and re-touched if cold
- [ ] SPI cable secured with strain relief — no flex-induced disconnection
- [ ] Power supply is the same unit used during development (voltage regulators can vary)

**Software:**
- [ ] Firmware compiled with `-O2` optimisation (not `-O0` debug — 3× slower)
- [ ] Watchdog timer active and verified
- [ ] UART JSON output readable on PC with live plotting script running
- [ ] Inference time logged and confirmed ≤ 15 ms at demo speed
- [ ] Green LED lights on startup (healthy baseline confirmed from day's run-in)

**Demonstration script (rehearse this sequence 5 times):**
```
Step 1: Power on. Green LED illuminates. Explain healthy baseline detection.
Step 2: Show live UART JSON output on laptop — {cls:0, conf:0.03, inf_ms:12}
Step 3: Show frequency spectrum plot (matplotlib, real-time from UART data)
         — point out the GMF peak, explain sideband region is clean
Step 4: Stop motor. Install FAULTED gear (notched tooth). Restart motor.
Step 5: Within 3 seconds: Amber LED → Red LED. Buzzer activates.
Step 6: Show UART: {cls:1, conf:0.92, inf_ms:13} — inference time unchanged
Step 7: Show spectrum plot — sideband energy visible at GMF ± shaft_freq
Step 8: State the metric: "This detection happened in 13 milliseconds.
         Current Indian military HUMS takes 4 to 6 flight hours.
         We have reduced the undetected fault window by 960,000 times."
```

**Jury questions — prepare one-sentence answers:**
- "Is this real-time?" → "Yes. The inference runs every 76.7 ms. Detection was in 13 ms."
- "Is it cloud-connected?" → "No. It has no networking hardware. It is physically impossible to connect to a cloud server from this microcontroller — which is why it works in EW-jammed airspace."
- "What if it gives a false alarm?" → "The debounce protocol requires 3 consecutive fault readings — 230 ms of sustained anomaly. No flight transient lasts that long."
- "How is this different from the Edition 3 runner-up project?" → "That project monitored automotive drivetrains via cloud-capable hardware with broad vehicular sensor fusion. Our system runs INT8 TinyML inference on an ARM Cortex-M0+ microcontroller for aerospace gearbox acoustic emission profiling in permanently offline, EW-contested military environments. The physics, the domain, the hardware class, and the threat model are categorically different."

---

## Appendix A: Reference Datasets

| Dataset | Source | Content | Best Use |
|---|---|---|---|
| CWRU Bearing Data Center | engineering.case.edu | Bearing inner/outer/ball faults at 12 kHz and 48 kHz; 4 fault severity levels | Primary training dataset — pre-train model |
| NASA PRONOSTIA (FEMTO-ST) | NASA PCoE | Run-to-failure bearing data at 1800 and 1650 RPM | Degradation trajectory modelling |
| Paderborn University Dataset | groups.uni-paderborn.de | Real and artificial bearing damage, broader operating conditions | Supplementary validation |
| Custom rig dataset | Collected in Module 4 | Your gear train at demo speed, healthy and notch-faulted conditions | Fine-tuning (transfer learning) |

## Appendix B: Key Library Versions

| Library | Version | Purpose |
|---|---|---|
| TensorFlow | 2.14.0 | Model training and TFLite conversion |
| TFLite Micro | Pinned to TF 2.14 tag | Embedded runtime |
| FreeRTOS | 10.5.1 | RTOS kernel |
| CMSIS-DSP | 1.14.4 (CMSIS 5.9.0) | FFT and signal processing |
| Pico SDK | 1.5.1 | RP2040 hardware abstraction |
| PlatformIO | Latest | IDE and build system |
| scipy | 1.11.x | Signal processing in Python pipeline |
| scikit-learn | 1.3.x | Train/test split, metrics |

## Appendix C: Gear Mesh Frequency Quick Reference

```
For any spur/helical gear pair or epicyclic stage:

  f_GMF = Z × n_shaft (Hz)
  where:
    Z       = number of teeth on the gear of interest
    n_shaft = rotational speed of that shaft in revolutions per second (RPS = RPM/60)

  Bearing defect frequencies:
    BPFO = (N_b/2) × n_shaft × (1 − (d_b/d_p) × cos α)
    BPFI = (N_b/2) × n_shaft × (1 + (d_b/d_p) × cos α)
    BSF  = (d_p / 2d_b) × n_shaft × (1 − ((d_b/d_p) × cos α)²)
    FTF  = (n_shaft/2) × (1 − (d_b/d_p) × cos α)
  where:
    N_b = number of rolling elements (balls/rollers)
    d_b = ball/roller diameter (mm)
    d_p = pitch circle diameter (mm)
    α   = contact angle (degrees, typically 15°–25°)

  Sideband structure around GMF for a gear fault:
    f_sideband,±k = f_GMF ± k × f_shaft,  k = 1, 2, 3, ...
    A fault on a single tooth creates modulation at the shaft rotation frequency.
    Sideband spacing = f_shaft (shaft speed in Hz).
    Increasing number and amplitude of sidebands = increasing fault severity.
```

---

*ARGUS-PHM Technical Stack & Learning Roadmap — Version 1.0*
*Tata InnoVent Edition 4 — AI at the Edge — Aerospace Vertical / TASL*
*Team: Atharv · Ganesh · Swaraj · Gaurav*
