# FPGA-Based Smart Wiper & Rain Detection System

A real-time digital hardware system implemented on a **Digilent Basys 3 (Artix-7) FPGA** using **VHDL**. The system monitors environmental moisture via an FC-37/MH-RD rain sensor, controls a wiper mechanism (**SG90 Servo** via 50 Hz PWM), indicates state via onboard LEDs, and displays real-time status messages on a **16x1 HD44780 LCD** in 4-bit mode.

## System Architecture & Pin Mapping
- **Controller:** Digilent Basys 3 FPGA (100 MHz System Clock - `W5`)
- **Rain Sensor Input:** Digital `DO` (Active-Low) connected to PMOD `JA2` (`L2`)
- **Actuator:** SG90 Servo Motor driven by 50 Hz PWM (`0.75 ms` to `2.25 ms` pulse width, ±45° sweep) on PMOD `JA1` (`J1`)
- **Display Interface:** HD44780 LCD in 4-bit mode connected to PMOD `JB` (`RS: A14`, `E: A16`, `D4-D7: B15, B16, A15, A17`)
- **Status Indicators:** Onboard LEDs (`LD0`: Dry, `LD1`: Rain Detected)

## Repository Structure
- `src/rain_system_top.vhd`: Top-level VHDL design integrating the main FSM, PWM generator, and custom 4-bit HD44780 LCD controller.
- `src/constraints.xdc`: Basys 3 physical pin constraints and 100 MHz clock definition.
- `images/`: Vivado simulation waveforms and hardware prototype verification photos.
