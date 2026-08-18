# Biometric Fingerprint Attendance Logger with Embedded FSM & FreeRTOS

A biometric attendance system built on the ATmega328P (Arduino Uno R3) microcontroller. This project resolves single-sensor entry/exit (IN/OUT) ambiguity using a deterministic Finite State Machine (FSM) and implements real-time task scheduling via FreeRTOS. Hardware schematics are validated in Proteus Design Suite, while behavioral state logic and preemptive task scheduling are verified using SystemC 2.3.3 on EDA Playground.

---

## Key Features

* **Single-Sensor Ambiguity Resolution**: Uses an FSM to automatically toggle a user's attendance status between `IN` and `OUT` based on previous records stored in EEPROM.
* **Preemptive RTOS Multitasking**: Decomposes firmware execution into three distinct FreeRTOS tasks (polling, state management, and display update) to eliminate super-loop latency.
* **Safe Inter-Task Communication**: Transmits matched user IDs from the scanning task to the FSM task via a FreeRTOS thread-safe queue (`eventQueue`).
* **Real-Time Timestamping**: Interfaces with an I²C RTC (DS3231 / DS3338U-33) to attach precise date and time records to attendance logs.
* **Serial CSV Logging**: Streams structured attendance entries (`ID, Name, Date, Day, Time, Status`) over hardware UART for PC/Excel data collection[cite: 2, 3].
* **Dual Simulation Validation**: Verified via Proteus EDA for circuit connectivity and SystemC 2.3.3 for cycle-accurate FSM and RTOS scheduler modeling[cite: 2, 3].

---

## Hardware Architecture & Pin Mapping

The system uses a 7805 voltage regulator with 1 µF input/output decoupling capacitors to supply a stable +5 V rail from a 12 V source[cite: 2, 3].

| Component | Interface / Pins | Details |
| :--- | :--- | :--- |
| **Arduino Uno R3** | ATmega328P Main MCU | Controls peripheral interfacing, FSM logic, and FreeRTOS kernel. |
| **R307 Fingerprint Sensor** | Hardware UART (Pins 0/1) or SoftwareSerial (Pins 2/3) @ 57600 bps | Captures fingerprint images and matches minutiae templates[cite: 2, 3]. |
| **DS3231 / DS3338U-33 RTC** | I²C Bus (SDA -> A4, SCL -> A5) @ Address `0x68` | Stores calendar and clock registers with battery backup |
| **16×2 Character LCD** | 4-bit Parallel (D3–D7, D11–D13) or I²C (`0x27`) | Displays user check-in status and live RTC time feedback |
| **Power Regulation** | 12V DC Input -> 7805 Regulator | Provides clean +5V supply filtered by capacitors C1 and C2. |

---

## Software Architecture

### FreeRTOS Task Decomposition

The system firmware is split into three FreeRTOS tasks governed by a fixed-priority preemptive scheduler:

| Task Name | Priority | Period / Timing | Description |
| :--- | :--- | :--- | :--- |
| **`Task_FSM`** | `3` (Highest) | Event-driven (`portMAX_DELAY`) | Consumes IDs from `eventQueue`, toggles user IN/OUT status, and emits serial CSV log. |
| **`Task_Fingerprint`** | `2` | Periodic (100 ms) | Polls the R307 sensor; posts verified user IDs to `eventQueue` when in `STATE_IDLE`. |
| **`Task_Display`** | `1` (Lowest) | Periodic (500 ms) | Displays user check-in info for 3 seconds before reverting to live RTC time. |

### FSM State Logic

The deterministic Finite State Machine transitions across 5 states:
* **`IDLE`**: Awaits a fingerprint match event.
* **`VERIFY`**: Queries prior state (`last_state_in`) from EEPROM/database.
* **`IN`**: Triggered if the user was `OUT` or unrecorded; logs entry timestamp.
* **`OUT`**: Triggered if the user was `IN`; logs departure timestamp.
* **`LOG`**: Asserts log trigger for UART transmission and returns to `IDLE`.


## Simulation & Validation

1. **Proteus Schematic Simulation**:
   * Verified complete schematic wiring and peripheral bus connections.
   * Confirmed proper voltage regulation (+5V) and 4-bit parallel LCD interface timing.

2. **SystemC FSM Verification**:
   * Simulated state transitions in SystemC 2.3.3 on EDA Playground.
   * Confirmed check-in state output at `T=30ns` and check-out state output at `T=90ns`.

3. **SystemC Task Scheduler Verification**:
   * Validated preemptive task dispatching under FreeRTOS logic.
   * Confirmed that `Task_FSM` (Priority 3) immediately preempts `Task_Fingerprint` (Priority 2) upon queue event unblock (`T=200ms`) with zero priority inversions.

---

## Getting Started

### Prerequisites

* **Arduino IDE** (v1.8.x or v2.x) with the following libraries:
  * `FreeRTOS` (`Arduino_FreeRTOS`)
  * `Adafruit_Fingerprint`
  * `RTClib`
  * `LiquidCrystal_I2C`
* **Proteus Design Suite** (v8.x+) for schematic capture and circuit simulation.
* **SystemC 2.3.3 Compiler** (GCC) or an active account on [EDA Playground](https://www.edaplayground.com).

### How to Run

1. **Proteus Simulation**:
   * Open the `.pdsprj` schematic in Proteus Design Suite.
   * Compile the Arduino sketch in Arduino IDE to generate the compiled `.hex` binary.
   * Load the `.hex` file into the ATmega328P microcontroller property panel in Proteus.
   * Run the simulation.

2. **SystemC Simulation**:
   * Copy the SystemC source code (`AttendanceFSM.cpp` or `rtos_attendance_scheduler.cpp`) to EDA Playground.
   * Select **SystemC 2.3.3** and **GCC** compiler.
   * Click **Run** to view console output logs.
