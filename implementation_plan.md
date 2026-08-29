# MotorSignal — Pre-Field-Test Engineering Hardening & Telemetry Audit Plan

Comprehensive engineering audit, mathematical analysis, and hardening specification for the **MotorSignal** telemetry architecture, Complementary Filter, timing synchronization, quality evaluation, simulation/replay suite, and Ride #4 regression dataset.

---

## User Review Required

> [!IMPORTANT]
> **Preservation of Working Complementary Filter**: The mathematical audit confirms that the 50Hz Adaptive Dynamic Complementary Filter is computationally lightweight ($O(1)$ ops), thermally efficient on motorcycle phone mounts, and mathematically sound. We will **NOT** replace it with an EKF, but will harden its gyro integration, continuous $\alpha$-blending, and orientation transformations.

> [!IMPORTANT]
> **Ride #4 Forensic Revelation**:
> - **51.53° Left Lean was a Kickstand / Stationary Dismount Artifact** (occurred at 16:08:02 at `0.0 MPH` in the final 3 seconds before session stop).
> - **24.40° Right Lean was a Valid Physical Turn** (occurred at 16:07:50 at `8.8 MPH` across 21 consecutive samples).
> - **Peak Accel (0.98G) and Braking (1.33G) also occurred at 0.0 MPH during dismount**.
> - **Harden Rule**: Personal Record (PR) qualification and maximum metric tracking will now be speed-gated (`speed >= 3.0 MPH`), preventing stationary parking/handling from skewing ride performance records.

---

## Phase-by-Phase Technical Specifications & Proposed Changes

### Phase 1 & 2: Telemetry Architecture & Authoritative 20Hz Clock
- **Clock Source**: Exactly **ONE** authoritative 20Hz clock (`_samplingTimer`, 50ms) driving sample synthesis.
- **Monotonic Sequencing**: Add sequential 64-bit integer `sequenceNumber`, `monotonicTimestampMs` (using `Stopwatch.elapsedMilliseconds` to eliminate wall-clock drift/jumps), and preserved source hardware timestamps.
- **Jitter & Gap Diagnostics**: Track sample-to-sample $\Delta t$ and log warning flags if interval deviates from $50\text{ms} \pm 15\text{ms}$.

### Phase 3: Strict Separation of Raw vs. Filtered Telemetry
- **Model Standard**:
  - **Raw Channels**: `rawAccelX/Y/Z`, `rawGyroX/Y/Z`, `rawMagX/Y/Z`, `gpsLatitude/Longitude/Altitude/Speed/Accuracy/Timestamp`.
  - **Filtered/Derived Channels**: `filteredLeanAngle`, `filteredPitchAngle`, `rollRateDegPerSec`, `longitudinalG`, `lateralG`, `derivedSpeedMph`, `heading`.
- **Database & Exports**: `telemetry.csv` and `telemetry.gpx` will retain raw channels alongside filtered states for forensic fidelity.

### Phase 4 & 5: Complementary Filter Hardening & Calibration Reference Frame
- **Continuous $\alpha$ Adaptation**: Replace step thresholds ($0.04g / 0.08g$) with smooth sigmoidal/continuous scaling $\alpha(|a| - g_0)$ to prevent step discontinuities during aggressive transitions.
- **Stationary Gyro Zero-Rate Bias**: During pre-ride calibration ("Hold motorcycle upright"), sample 100 consecutive gyro frames to compute static zero-rate bias ($b_{gx}, b_{gy}, b_{gz}$) and subtract it from integration.
- **Persistent Motorcycle-Specific Offsets**: Store pitch, roll, and gyro bias per motorcycle ID in `CalibrationService`.

### Phase 6 & 7: Ride #4 Regression Analysis & GPS Filtering
- **GPS Validation**: GPS speed and coordinates pass through strict bounds checking (accuracy $\le 50\text{m}$, speed $\le 220\text{ MPH}$, max position jump velocity $\le 110\text{ m/s}$).
- **PR Gating**: G-force and lean records require `speed >= 3.0 MPH` and `quality.isValid` to prevent dismount/kickstand artifacts.

### Phase 8 & 9: Derived Event Detection & Category-Based Quality System
- **Event Persistence**: Harden multi-sample debouncing on 0–60, 0–100, top speed, hard acceleration ($>0.5\text{g}$), hard braking ($>0.6\text{g}$), high lean ($>35^\circ$), and GPS anomalies.
- **Categorical Quality Diagnostics**: Replace single score with multidimensional metrics:
  - `Timing Integrity` (monotonicity, jitter)
  - `GPS Quality` (fix percentage, average accuracy, jump rejections)
  - `IMU Continuity` (sample dropouts, saturation count)
  - `Database Integrity` (samples written vs. lost)
  - `Filter Health` (dynamic confidence distribution)

### Phase 10: Database Reliability & Finite Lifecycle States
- **Lifecycle States**: `created` $\to$ `ready` $\to$ `recording` $\to$ `paused` $\to$ `finalizing` $\to$ `completed` / `recovered`.
- **Periodic Batch Flush**: 3-second periodic write + unclosed ride recovery upon launch.

### Phase 11 & 12: Production-Grade Telemetry Simulator / Replay Engine
- **Engine**: Build `TelemetryReplayEngine` capable of feeding either historical CSV files (e.g. Ride #4) or synthetic mathematical scenarios through the exact production pipeline (`TelemetryService` $\to$ `Validator` $\to$ `RideStateMachine` $\to$ `Database` $\to$ `ExportService`).
- **18 Required Scenarios**:
  1. Stationary motorcycle (idle vibration)
  2. Smooth acceleration
  3. 0–60 MPH run
  4. 0–100 MPH run
  5. Hard braking (1.2G)
  6. Smooth left corner (35° lean)
  7. Smooth right corner (25° lean)
  8. High-G acceleration (0.8G)
  9. High-G braking (1.4G)
  10. GPS dropout (tunnel)
  11. GPS accuracy degradation (canyon)
  12. GPS position jump glitch (multipath)
  13. IMU sensor dropout
  14. Duplicate timestamp anomaly
  15. Timestamp regression anomaly
  16. Irregular 20Hz jitter
  17. App backgrounding & recovery
  18. Storage write backpressure / overflow

### Phase 13: Automated Regression & Unit Test Suite
- Comprehensive unit and widget tests covering all 18 simulation scenarios, filter response, calibration persistence, and Ride #4 regression validation.

### Phase 14, 15 & 16: UX Safety, Camera Interfaces & Electrical Subsystem Architecture
- **UX Safety**: High-contrast, glanceable dials; screen-off background recording with wake-lock.
- **Camera Abstraction**: Clean `CameraProvider` interface with telemetry event hooks.
- **Electrical Prep**: Non-intrusive `MotorcycleElectricalService` interface for future Bluetooth/BLE voltage sensors.

---

## Proposed File Changes

#### [MODIFY] [lib/models/telemetry_sample.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/models/telemetry_sample.dart)
- Add `sequenceNumber`, `monotonicTimestampMs`, `rawAccelX/Y/Z`, `rawGyroX/Y/Z`, `rollRateDegPerSec`.

#### [MODIFY] [lib/services/sensor_fusion_service.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/sensor_fusion_service.dart)
- Continuous $\alpha$-adaptation, static gyro zero-bias subtraction, speed-gated G-force body projections.

#### [MODIFY] [lib/services/calibration_service.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/calibration_service.dart)
- Add gyro zero-rate bias calibration routines and persistence per motorcycle ID.

#### [MODIFY] [lib/services/telemetry_service.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/telemetry_service.dart)
- Monotonic stopwatch clock, sequence generator, jitter measurement, raw vs filtered preservation.

#### [MODIFY] [lib/services/validation/telemetry_validator.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/validation/telemetry_validator.dart)
- Categorical health scoring, speed-gated roll rate and acceleration anomaly checks.

#### [MODIFY] [lib/features/rides/ride_state_machine.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/features/rides/ride_state_machine.dart)
- Speed-gate personal records (`speed >= 3.0 MPH`), preserve raw channels in database companion.

#### [NEW] [lib/services/simulation/telemetry_simulation_engine.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/simulation/telemetry_simulation_engine.dart)
- Production-grade replay & synthetic scenario generator for all 18 verification cases.

#### [NEW] [lib/services/hardware/camera_provider.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/hardware/camera_provider.dart)
- Clean architecture interfaces for future camera action triggers.

#### [NEW] [lib/services/hardware/motorcycle_electrical_service.dart](file:///c:/Users/Bill/Desktop/flutter-app/lib/services/hardware/motorcycle_electrical_service.dart)
- Clean architecture interfaces for future BLE battery/charging voltage monitoring.

#### [NEW] [test/pre_field_test_engineering_hardening_test.dart](file:///c:/Users/Bill/Desktop/flutter-app/test/pre_field_test_engineering_hardening_test.dart)
- Comprehensive test suite covering all 18 simulation scenarios and Ride #4 regression.

---

## Verification Plan

### Automated Tests
- Run `flutter test test/pre_field_test_engineering_hardening_test.dart`
- Run full test suite: `flutter test` (all 130+ tests must pass)
- Static Analysis: `flutter analyze lib/ test/` (0 issues)

### Regression Verification
- Replay Ride #4 dataset through `TelemetrySimulationEngine` and verify:
  - 18,981 samples processed monotonically.
  - Max Left Lean at speed: physical turn evaluated accurately.
  - Kickstand dismount at 0 MPH properly classified as stationary.
  - Zero duplicate timestamps and zero dropped frames.
