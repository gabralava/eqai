# EQAI

## 1. Project Overview

**EQAI** is a **100% offline, on-device, system-wide parametric equalizer** designed to automatically calibrate headphones to a scientific reference target and allow intuitive sound shaping using abstract semantic controls.

### Core Product Capabilities
- **Automatic Headphone Calibration (AutoEQ)**: Ships an offline catalog of ~4,000 measured headphone frequency response (FR) curves sampled on a **512-point logarithmic grid (20 Hz – 20,000 Hz)**, normalized at 1 kHz. An on-device neural network predicts optimal 7-band biquad EQ settings to match a selected reference target (**Harman 2019** primary target; **Diffuse Field** and **Flat** optional targets).
- **Abstract Semantic Sound Tuning**: Users can refine audio using high-level semantic controls—`moreBass`, `richerSound`, `warmth`, and `clarity` — translated into 7-band EQ parameter deltas (`Δ[freq, gain, Q]`) by an on-device ML model.
- **Manual 7-Band Parametric Control**: Full manual adjustment of all 7 biquad bands (`LowShelf`, `PeakBell`, `HighShelf`), auto/manual preamp gain, quick presets, and instant A/B bypass comparison.
- **System-Wide Audio Equalization**: Applies equalization globally to device audio output (not restricted to an in-app music player) with automatic profile switching on wired/Bluetooth headphone connection.
- **Zero Cloud / Zero Runtime Backend**: All inference, database queries, DSP math, and audio routing run locally on the user's device. Python is used strictly offline at build/training time.
- **Platform Strategy**: **Android (API 28+ / Android 9+)** is the MVP target (`sessionId = 0`), while all lower-layer interfaces are explicitly designed for seamless expansion to **iOS, Windows, and Linux** without modifying Layers 1–3.

---

## 2. Definitive Technology Stack

| Layer / Concern | Locked-In Technology | Purpose |
| :--- | :--- | :--- |
| **Parametric EQ Spec** | **7-Band Biquad EQ** (`7 × 3 = 21` params) | Fixed model output tensor, Rust RBJ cascade size, and OS equalizer stage count (`LowShelf`, `PeakBell`, `HighShelf`). |
| **Layer 1 — UI Shell** | **Flutter / Dart** (`Material 3`) | Thin cross-platform presentation shell. |
| **Layer 1 — State & DI** | **Riverpod (`flutter_riverpod`)** | Lightweight reactive UI state and dependency injection. |
| **Layer 1 — FR Chart** | **Flutter `CustomPainter`** (Native Canvas API) | 60 fps logarithmic 20 Hz–20 kHz dB curve rendering (512 points) and 2D draggable 7-band control nodes. |
| **Layer 1 — Bundled AutoEQ DB** | **SQLite** (`.db` asset with `FTS5` + binary `FLOAT32 BLOB`s) | Read-only ~4,000-model headphone catalog (~10–12 MB; 2 KB raw `FLOAT32` blob per 512-pt curve; zero JSON startup parsing). |
| **Layer 1 — Local User DB** | **SQLite (`drift` / `sqlite3`)** | Read-write persistence for user `EQProfile`s, Bluetooth device mappings, cached 7-band inference outputs, and preferences. |
| **Layer 2 — On-Device AI** | **TensorFlow Lite (`tflite_flutter`)** in a **Dart Background Isolate** | Asynchronous off-UI-thread INT8 inference (`< 1 MB` models): `base_eq_int8.tflite` (1D CNN + MLP) and `abstract_tuning_int8.tflite` (semantic delta model). |
| **Layer 2 ↔ Layer 3 Interop** | **`flutter_rust_bridge`** (over `dart:ffi`, `#[repr(C)]`) | Memory-safe, cross-Isolate FFI binding passing `EqParameterFrame` up to Rust and returning `EqMathTelemetryFrame` (512-pt curves) down. |
| **Layer 3 — Math & DSP Core** | **Rust (`libeqai_math.so`)** | 100% OS-agnostic core: RBJ Audio EQ Cookbook coefficients (`b0, b1, b2, a1, a2`), lock-free Direct Form II Transposed 7-band PCM cascade, 512-pt FR evaluator, crossfade smoother, and auto-preamp clipping guard. |
| **Layer 4 — Android Audio (MVP)** | **Rust JNI (`jni-rs`) $\rightarrow$ `android.media.audiofx.DynamicsProcessing`** | Direct global audio equalization (`sessionId = 0`, `MODIFY_AUDIO_SETTINGS`, `bandCount = 7`) invoked straight from Rust—bypassing Flutter `MethodChannel`. |
| **Android Hardware Events** | Lightweight `BroadcastReceiver` + `EventChannel` | Captures `ACTION_HEADSET_PLUG` and Bluetooth ACL device names for automatic profile switching; registers `JavaVM` hook (`JNI_OnLoad`) for Rust. |
| **Offline ML Training** | **Python CLI + TensorFlow / Keras** | Build-time 1D CNN + MLP and semantic delta training, native `TFLiteConverter` INT8 post-training quantization. |
| **Offline Dataset & Tracking** | **NumPy, SciPy, Apache Parquet, HDF5 (`.h5`), MLflow** | AutoEQ scraping into Parquet, 512-pt normalized train/val/test tensors in HDF5, local reproducible CLI runs logged via MLflow. |

---

## 3. System Architecture

EQAI is documented from two complementary perspectives in `.agents/architecture/`:
1. **OSI-Inspired 4-Layer Hierarchical Architecture** ([`.agents/architecture/eqai-osi-layered-architecture.drawio`](.agents/architecture/eqai-osi-layered-architecture.drawio)) — defines strict layer adjacency and universal cross-platform Service Access Points (`SAP-1`, `SAP-2`, `SAP-3`).
2. **C4 Model Architecture (C1–C4)** ([`.agents/architecture/eqai-c4-architecture.drawio`](.agents/architecture/eqai-c4-architecture.drawio)) — details system context, build-time & runtime containers, internal components, and concrete Dart/Rust/Kotlin class & struct contracts.

### 3.1 4-Layer Hierarchical Architecture

Each layer communicates **only** with the layer immediately above or below it through formal Service Access Points (SAPs).

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 4 (Top-Most): OS Audio Subsystem Layer (Swappable Platform Backends)        │
│ • Android MVP: android.media.audiofx.DynamicsProcessing (sessionId = 0, jni-rs)   │
│ • iOS Target:  AUv3 Audio Unit Extension / AVAudioEngine (CoreAudio C-ABI)        │
│ • Windows:     Windows Audio Processing Object (sAPO/eAPO) / WASAPI (COM / C-ABI) │
│ • Linux:       PipeWire Filter-Chain (spa_node / LADSPA / LV2) / ALSA (C-ABI)     │
└─────────────────────────────────────────▲─────────────────────────────────────────┘
                                          │
      ═══════════ SAP-3: UniversalSystemAudioHal (Rust Trait & C-ABI VTable) ═══════
                                          │
┌─────────────────────────────────────────▼─────────────────────────────────────────┐
│ LAYER 3: Rust Mathematical & DSP Engine Layer (libeqai_math — 100% OS-Agnostic)   │
│ • 7-Band RBJ Biquad Coefficient Engine (b0, b1, b2, a1, a2)                       │
│ • Lock-Free Direct Form II Transposed 7-Band PCM Filter Cascade                   │
│ • 512-Point Logarithmic FR Evaluator (20 Hz – 20 kHz in dB)                       │
│ • Crossfade Parameter Interpolator & Auto-Preamp Clipping Guard                   │
└─────────────────────────────────────────▲─────────────────────────────────────────┘
                                          │
      ═══════════ SAP-2: UniversalEqEngineFfiContract (flutter_rust_bridge / C-ABI) 
                                          │
┌─────────────────────────────────────────▼─────────────────────────────────────────┐
│ LAYER 2: AI Model & Intent Resolution Layer (On-Device TFLite Background Isolate) │
│ • Path A: Base 7-Band AutoEQ Predictor (base_eq_int8.tflite: [1024] -> [21])      │
│ • Path B: Abstract Semantic Predictor (abstract_tuning_int8.tflite: -> Δ[21])     │
│ • Path C (OSI Caveat): Manual Override Pass-Through Mux (0 ms ML skip -> SAP-2)   │
└─────────────────────────────────────────▲─────────────────────────────────────────┘
                                          │
      ═══════════ SAP-1: UiToAiIntentContract (Dart Isolate SendPort / ReceivePort) 
                                          │
┌─────────────────────────────────────────▼─────────────────────────────────────────┐
│ LAYER 1 (Bottom-Most): Flutter Presentation & Persistence Shell Layer             │
│ • Thin UI Shell (Material 3, Riverpod, CustomPainter 60 fps 512-pt FR Chart)      │
│ • Local SQLite Persistence (AutoEQ FTS5 Asset DB + Drift User Profile DB)         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

#### OSI Architecture Caveats
1. **Manual Adjustment Pass-Through (Layer 2 Path C)**: To keep Layer 1 strictly decoupled from Layer 3 while supporting zero-latency manual slider drags, Layer 1 sends `SAP1_ForwardManualOverride` up to Layer 2, where a deterministic pass-through gate skips `.tflite` inference (`0 ms` overhead) and forwards the 7-band `EqParameterFrame` directly across `SAP-2` into Layer 3.
2. **Zero-Impact Top-Layer Swappability**: Only Layer 4 contains OS-specific code (`jni-rs` / `AUv3` / `APO` / `PipeWire`). Layers 1, 2, and 3—along with `SAP-1` and `SAP-2`—contain zero OS types, imports, or handles.

### 3.2 Service Access Point (SAP) Interface Contracts

#### `SAP-1` (`Layer 1 Flutter` ↔ `Layer 2 AI Model`)
- **Transport**: Asynchronous zero-copy Dart Isolate channel (`SendPort` / `ReceivePort` + `TransferableTypedData`).
- **Upward Primitives (`L1 -> L2`)**:
  - `SAP1_RequestAutoEq(headphone_fr_512: Float32List[512], target_fr_512: Float32List[512])`
  - `SAP1_RequestSemanticTuning(base_bands_7: BiquadBandSpec[7], semantic_vec_4: Float32List[4], headphone_fr_512: Float32List[512])`
  - `SAP1_ForwardManualOverride(manual_bands_7: BiquadBandSpec[7], manual_preamp_db: Option<f32>, bypass: bool)`
  - `SAP1_SetSystemBypass(bypass: bool)`
- **Downward Primitives (`L2 -> L1`)**:
  - `SAP1_OnTelemetryUpdated(effective_bands_7, composite_eq_512_db, compensated_fr_512_db, safe_preamp_db, inference_latency_us)`
  - `SAP1_OnHardwareRouteEvent(device_name: String, transport_type: Wired | Bluetooth | Speaker, sample_rate_hz: u32)`

#### `SAP-2` (`Layer 2 AI Model` ↔ `Layer 3 Rust Engine`)
- **Transport**: `flutter_rust_bridge` / C-ABI (`#[repr(C)]`, zero OS types).
- **Canonical Structs**:
  - `BiquadFilterType (u8)`: `0 = LowShelf`, `1 = PeakBell`, `2 = HighShelf`
  - `BiquadBandSpec (32 B, #[repr(C)])`: `{ band_idx: u8, filter_type: u8, enabled: u8, _pad: u8, freq_hz: f64, gain_db: f64, q_factor: f64 }`
  - `EqParameterFrame (240 B, #[repr(C)])`: `{ bands: [BiquadBandSpec; 7], preamp_override_db: f32, has_preamp_override: u8, bypass: u8, crossfade_duration_ms: u16, target_sample_rate_hz: u32 }`
  - `BiquadCoeffs (40 B, #[repr(C)])`: `{ b0: f64, b1: f64, b2: f64, a1: f64, a2: f64 }`
  - `EqMathTelemetryFrame (#[repr(C)])`: `{ coeffs: [BiquadCoeffs; 7], composite_eq_db: [f32; 512], compensated_fr_db: [f32; 512], computed_preamp_db: f32, peak_gain_db: f32, status_code: i32 }`
  - `RawDeviceEvent (#[repr(C)])`: `{ device_name_utf8: [u8; 128], transport: u8, sample_rate_hz: u32 }`
- **Primitives**:
  - `sap2_commit_eq_frame(frame: *const EqParameterFrame, measured_fr_512: *const f32, out_telemetry: *mut EqMathTelemetryFrame) -> i32`
  - `sap2_set_global_bypass(bypass: bool) -> i32`
  - `sap2_register_route_listener(cb: extern "C" fn(*const RawDeviceEvent)) -> i32`

#### `SAP-3` (`Layer 3 Rust Engine` ↔ `Layer 4 OS Audio Subsystem`) — Dual-Mode Universal HAL
To support both protected system-server equalizers and real-time audio plugin graphs without changing Layers 1–3, `SAP-3` (`pub trait UniversalSystemAudioHal: Send + Sync` & C-ABI `eqai_hal_vtable`) supports **two capability modes** discovered at startup via `capabilities()`:

- **Trait & C-ABI Contract**:
  - `fn capabilities(&self) -> HalCapabilities` (`supports_system_parametric_dsp`, `supports_pcm_stream_callback`, `min_sample_rate`, `max_bands`)
  - `fn init_session(&mut self, config: HalSessionConfig) -> Result<HalSessionHandle, HalError>`
  - `fn commit_parametric_state(&mut self, state: &HalEqState) -> Result<(), HalError>` *(Mode A parameter push / Mode B atomic coefficient swap)*
  - `extern "C" fn process_realtime_pcm_f32(ctx: *mut DspCascadeState, in_out_pcm: *mut f32, frames: u32, channels: u16, sample_rate: u32) -> i32` *(Mode B lock-free real-time PCM callback provided by L3 to L4)*
  - `fn subscribe_route_events(&mut self, sink: HalRouteEventCallback) -> Result<(), HalError>`
  - `fn shutdown(&mut self) -> Result<(), HalError>`

| Target OS (Layer 4) | Status | `SAP-3` Mode | Platform Mechanism |
| :--- | :--- | :--- | :--- |
| **Android (API 28+)** | **MVP Active** | **Mode A** (Parameter Push) | Rust `jni-rs` $\rightarrow$ `android.media.audiofx.DynamicsProcessing` (`sessionId = 0`, `7` bands + preamp). |
| **iOS / macOS** | Target Expansion | **Mode B** (Real-Time PCM) | `AUv3 Audio Unit Extension` / `AVAudioEngine` calling `process_realtime_pcm_f32()` via lock-free atomic coeff swap (App Group IPC). |
| **Windows** | Target Expansion | **Mode B** (Real-Time PCM) | System-wide `sAPO` / `eAPO` (`IAudioProcessingObjectRealtime` / `WASAPI`) calling `process_realtime_pcm_f32()` inside `APOProcess()`. |
| **Linux** | Target Expansion | **Mode B** (or Mode A) | `PipeWire Filter-Chain` (`spa_node` / LADSPA / LV2) or `ALSA pcm_extplug` calling `process_realtime_pcm_f32()` (or pushing biquad coeffs to PipeWire filter-chain nodes). |

---

## 4. End-to-End Data Flow Scenarios

### A. Automatic AI Calibration & Abstract Semantic Tuning (`"More Bass"`)
1. **L1 (Flutter)**: User picks a headphone model (loads `512`-pt `FLOAT32 BLOB` from SQLite `autoeq.db`) or moves a semantic slider (e.g., `moreBass = +0.6`). Sends `SAP1_RequestAutoEq` or `SAP1_RequestSemanticTuning` up across `SAP-1`.
2. **L2 (TFLite Isolate)**: Executes `base_eq_int8.tflite` (`[1024] -> [21]`) or `abstract_tuning_int8.tflite` (`[21 + 4 + 512] -> Δ[21]`), clamps parameter bounds (`freq ∈ [20, 20000] Hz`, `gain ∈ [-15, +15] dB`, `Q ∈ [0.3, 12.0]`), and commits `EqParameterFrame` (`crossfade_duration_ms = 50`) up across `SAP-2`.
3. **L3 (Rust Engine)**: Computes 7 RBJ biquad coefficient sets, evaluates `composite_eq_db[512]` and `compensated_fr_db[512]`, calculates safe negative preamp (`preamp_db = -max(0, peak_db)`), runs crossfade interpolation, and pushes `HalEqState` up across `SAP-3`.
4. **L4 (Android MVP)**: `jni-rs` driver invokes `setInputGainAllChannelsTo`, `setPreEqBandAllChannelsTo(0..6)`, and `setEnabled` on `DynamicsProcessing(sessionId = 0)`.
5. **Downward Return (`L3 -> L2 -> L1`)**: `EqMathTelemetryFrame` returns down `SAP-2` $\rightarrow$ `SAP-1` (`TransferableTypedData`); `Riverpod` notifies `CustomPainter` to redraw at 60 fps.

### B. Manual 7-Band Adjustment (OSI Caveat Pass-Through)
1. **L1 (Flutter)**: User drags one of the 7 band nodes on `CustomPainter`. Emits `SAP1_ForwardManualOverride` up across `SAP-1`.
2. **L2 (TFLite Isolate — Path C)**: Bypasses `.tflite` interpreters (`0 ms` ML latency) and forwards `EqParameterFrame` (`crossfade_duration_ms = 15`) up across `SAP-2`.
3. **L3 → L4 → L1**: Rust recalculates coefficients, composite 512-pt FR, and auto-preamp, pushes to L4 over `SAP-3`, and returns updated curves down `SAP-2` $\rightarrow$ `SAP-1`.

### C. Downward Hardware Route Event & Auto-Profile Switch
1. **L4 → L3**: Wired plug (`ACTION_HEADSET_PLUG`) or Bluetooth ACL connection (e.g., `"Sony WH-1000XM5"`) emits `HalRouteEvent` with device name and sample rate (`fs`) down `SAP-3`.
2. **L3 → L2 → L1**: Rust updates `fs` (recomputing $\omega_0 = 2\pi f_0 / f_s$) and relays `RawDeviceEvent` down `SAP-2` $\rightarrow$ `SAP1_OnHardwareRouteEvent` across `SAP-1`.
3. **L1 (Flutter)**: `ProfileRepository` queries SQLite for a saved profile matching `device_name` and triggers crossfade profile activation.

---

## 5. Offline Build-Time ML & Data Pipeline (`model/`)

The Python stack runs **exclusively offline** during development/CI to build the read-only Flutter assets:

1. **AutoEQ Scraping & Catalog Packaging**: Python CLI clones the AutoEQ repository (~4,000 headphones), parses measured FR curves and reference EQ data, writes an intermediate **Apache Parquet** catalog, and packs the runtime read-only **SQLite** database (`autoeq.db` with `FTS5` search index and raw 2 KB `FLOAT32 BLOB`s).
2. **Preprocessing & Dataset Export**: Interpolates every curve to the canonical 512-point logarithmic grid (20 Hz–20 kHz), normalizes level at 1 kHz, computes Harman 2019 target correction curves, splits into `70% / 15% / 15%` (`train / val / test`) with slight FR noise augmentation, and saves tensors in **HDF5 (`.h5`)**.
3. **Training & Tracking**: Trains two compact **TensorFlow / Keras** models tracked locally with **MLflow**:
   - **Base 7-Band AutoEQ Model**: `1D CNN` encoder (3 `Conv1D` blocks $\rightarrow$ `GlobalAvgPool`) + `MLP` decoder (`Dense 128 -> 64 -> 21` outputs for 7 bands × `[freq, gain, Q]`), MSE loss + physical frequency constraints, Adam + cosine LR schedule.
   - **Abstract Semantic Tuning Model**: Maps `[base_bands(21) + semantic_vec(4) + headphone_fr(512)]` $\rightarrow \Delta[21]$ band adjustments.
4. **INT8 Quantization & Asset Bundling**: Exports via native `tf.lite.TFLiteConverter` with post-training **INT8 quantization** (`< 1 MB` per model) and places `base_eq_int8.tflite`, `abstract_tuning_int8.tflite`, and `autoeq.db` into Flutter assets (`app/assets/`).

---

## 6. Current Repository Layout & Status

```text
eqai/
├── .agents/
│   ├── architecture/
│   │   ├── eqai-c4-architecture.drawio          # 4-page C4 model (C1–C4)
│   │   └── eqai-osi-layered-architecture.drawio # 4-page OSI stack & SAP-1/2/3 contracts
│   └── skills/
│       └── rust-skills/                         # Rust coding guidelines for agents
├── app/                                         # Layer 1 & 2: Flutter project (currently starter template, Dart SDK ^3.13.5)
│   ├── android/                                 # Android MVP host & BroadcastReceiver/JNI hook
│   ├── ios/, linux/, macos/, windows/           # Cross-platform Flutter targets
│   ├── lib/main.dart                            # Default starter main.dart
│   └── pubspec.yaml                             # Currently flutter + flutter_lints only
├── engine/                                      # Layer 3 & 4: Rust libeqai_math crate & SAP-3 HAL (currently empty)
├── model/                                       # Offline Python CLI ML & AutoEQ pipeline (currently empty)
├── tests/                                       # Cross-layer test suites (app/, engine/, model/ — currently empty)
├── skills-lock.json
└── README.md                                    # This file
```

## 7. Guidelines for AI Agents Working on This Repository

1. **Respect Strict Layer Boundaries**: Never let Layer 1 (Flutter) invoke Android `DynamicsProcessing` or hold OS audio logic. Never leak Android/iOS/Windows/Linux types into `SAP-1`, `SAP-2`, Layer 2, or Layer 3's core math modules.
2. **Preserve the 7-Band Invariant**: All tensor shapes (`[21] = 7 × 3`), Rust fixed arrays (`[BiquadBandSpec; 7]`, `[BiquadCoeffs; 7]`), Dart models, and `DynamicsProcessing` band counts must stay synchronized at **7 bands**.
3. **Zero Allocations on Real-Time Audio Paths**: Any code reachable from `SAP-3`'s `process_realtime_pcm_f32` (Mode B) must be strictly lock-free, allocation-free, and syscall-free (use double-buffered atomic coefficient swaps and denormal flush-to-zero).
4. **Follow Rust Guidelines**: Before writing or refactoring code in `engine/`, consult the local Rust skill rules in [`.agents/skills/rust-skills/SKILL.md`](.agents/skills/rust-skills/SKILL.md).
5. **Keep Architecture Docs in Sync**: If an interface signature (`SAP-1`, `SAP-2`, `SAP-3`) or major component changes, update both `.drawio` diagrams in `.agents/architecture/` and this `README.md` in the same PR.
