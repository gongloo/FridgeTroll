# FridgeTroll

This repository contains the ESPHome configuration and documentation for FridgeTroll, an advanced replacement for traditional mechanical refrigerator thermostats. It is specifically optimized for the SECOP 101N2000 controller but compatible with most modern variable-speed SECOP units.

## Key Features

- **Adaptive Compressor Control**: Logarithmic recovery profiler dynamically adjusts compressor RPM based on thermal load and target recovery time.
- **Predictive Braking**: Lead compensation algorithm calculates future temperature trajectories to prevent overshoot and minimize cycling.
- **Freeze Protection**: Dedicated fridge monitoring prevents accidental freezing of fresh food by overriding compressor state.
- **Intelligent Defrosting**: Automated nightly defrost cycles that adapt to natural compressor rest periods.
- **Dynamic Configuration**: Configure Dallas sensor addresses and telemetry (Syslog/InfluxDB) directly via the web UI—no recompilation required.
- **Stall Detection**: Automatically detects and corrects cooling stalls.
- **Anti-Short-Cycle Protection**: Protects your compressor from rapid power cycles.
- **Rich Telemetry**: Full integration with InfluxDB and Home Assistant.
- **Web Installer**: Pre-built firmware installable directly from your browser.

## Documentation & Installation

For full installation instructions, wiring diagrams, and a complete Bill of Materials, please see the **[FridgeTroll Documentation Site](https://gongloo.github.io/FridgeTroll/)**.
