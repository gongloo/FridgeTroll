# FridgeTroll

This repository contains the ESPHome configuration and documentation for FridgeTroll, an advanced replacement for traditional mechanical refrigerator thermostats. It is specifically optimized for the SECOP 101N2000 controller but compatible with most modern variable-speed SECOP units.

## Key Features

- **Asymmetric PD + Feedforward Compressor Control**: Dynamic proportional-derivative algorithm with asymmetric cooling vs. warming derivative damping, ambient temperature feedforward, and fridge fan cooling feedforward for preemptive thermal load compensation.
- **Proportional Fridge Fan Modulation**: Linearly modulates fridge circulation fan speed within a configurable temperature ramp band with dedicated freeze protection overrides.
- **Freeze Protection**: Dedicated fridge monitoring prevents accidental freezing of fresh food by overriding compressor state.
- **Intelligent Defrosting**: Automated nightly defrost cycles that adapt to natural compressor rest periods.
- **Dynamic Configuration**: Configure Dallas sensor addresses and telemetry (Syslog/InfluxDB) directly via the web UI—no recompilation required.
- **Stall Detection**: Automatically detects and corrects cooling stalls.
- **Anti-Short-Cycle Protection**: Protects your compressor from rapid power cycles.
- **Rich Telemetry**: Full integration with InfluxDB and Home Assistant.
- **Web Installer & Package Support**: Install pre-compiled firmware directly from your browser, or import FridgeTroll as a centralized ESPHome package.

## Documentation & Installation

For full installation instructions, wiring diagrams, and a complete Bill of Materials, please see the **[FridgeTroll Documentation Site](https://gongloo.github.io/FridgeTroll/)**.
