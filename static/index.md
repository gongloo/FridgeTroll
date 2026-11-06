# FridgeTroll

An ESPHome-based smart controller for SECOP compressor-driven refrigerators, providing advanced temperature management, automated defrost, and telemetry.

## Overview

FridgeTroll is designed as an off-the-shelf, installable firmware primarily targeted at the **SECOP 101N2000** compressor controller (though it should work with most modern SECOP controllers). It replaces the standard mechanical thermostat with an ESP32, offering fine-grained control over compressor speed, dedicated fridge and compressor fan management, and rich telemetry for Home Assistant via InfluxDB and native ESPHome APIs.

# Installation

You can use the button below to install the pre-built firmware directly to your device via USB from the browser.

<esp-web-install-button manifest="firmware/fridgetroll.manifest.json"></esp-web-install-button>

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

## Bill of Materials (BOM)

To build the FridgeTroll hardware interface, you will need:

- **Microcontroller**: Wemos S3 Mini (or similar ESP32 board)
- **Power Supply**: Wemos DC Power Shield (or similar 5V USB-C power supply)
- **Fridge Fan**: Coolerguys 12vDC Waterproof IP67 Fan (High Speed, 25x10mm) (or similar 12V DC fan)
- **Compressor Fan**: Noctua NF-P12 redux-1700 PWM (or similar 120mm PWM fan)
- **Temperature Sensors**: DS18B20 1-Wire sensors (x4 recommended: Fridge, Freezer, Compressor, Ambient)
- **Optocouplers**: PC817 (x3)
  - Compressor Enable Pin (T)
  - Compressor PWM Pin (P)
  - Door Switch Pin
- **Transistor**: 2N2222 NPN Transistor (for driving the Fridge Fan)
- **Diode**: 1N4007 or similar (Flyback protection for the Fridge Fan)
- **Resistors**: Various (for optocoupler LEDs, pull-ups, and pull-downs depending on your specific board layout)

## Interfacing with the SECOP Controller

The SECOP controller provides several terminals that need to be interfaced with the ESP32. Due to voltage differences (the SECOP operates on 12V/24V and supplies pull-up voltages), **opto-isolation is required** for the compressor control signals and the door switch to protect the ESP32.

### SECOP Terminal Functions

On standard SECOP / Danfoss variable-speed electronic units (such as the 101N2000):

- **C (Common / Control)**: Common control supply voltage and reference line.
- **T (Thermostat)**: Compressor enable input. Bridging Terminal `C` to Terminal `T` commands the compressor to start. Opening `C` to `T` turns it off.
- **P (Programming / Speed PWM)**: Variable speed control input. Pulsing an open-collector signal between `P` and `C` modulates compressor RPM.
- **D (Diagnostic LED)**: Output terminal for fault diagnostic codes (connected via LED to `+`).

### Wiring Schematic

```
           +---------------------------------------------+
           |           SECOP Electronic Unit             |
           |                                             |
           |   (+) Battery Positive (12V/24V)            |
           |   (-) Battery Ground (GND)                  |
           |   (C) Control Common ----+                  |
           |   (P) Speed PWM <--------|----+             |
           |   (T) Thermostat Run <---|----|----+        |
           |   (D) Diagnostic LED     |    |    |        |
           +--------------------------|----|----|--------+
                                      |    |    |
                     PC817 [PWM]      |    |    |
                    +------------+    |    |    |
ESP32 GPIO10 -------| 1 (A) 4 (C)|---------+    |   (Collector to Terminal P)
ESP32 GND   --------| 2 (K) 3 (E)|----+         |   (Emitter to Terminal C)
                    +------------+    |         |
                                      |         |
                    PC817 [Enable]    |         |
                    +------------+    |         |
ESP32 GPIO12 -------| 1 (A) 4 (C)|--------------+   (Collector to Terminal T)
ESP32 GND   --------| 2 (K) 3 (E)|----+             (Emitter to Terminal C)
                    +------------+
```

### 1. Compressor Enable (SECOP Terminals T & C)

The compressor turns on when Terminal `T` is connected to Common Terminal `C`.

- **Circuit**: Use a PC817 optocoupler. Connect the ESP32's `compressor_enable_pin` (GPIO12) to the optocoupler's anode (Pin 1) via a current-limiting resistor (e.g., 330Ω–1kΩ), and connect ESP32 Ground to the cathode (Pin 2). Connect the optocoupler's collector (Pin 4) to SECOP Terminal `T`, and the emitter (Pin 3) to SECOP Terminal `C`.

### 2. Compressor Speed PWM (SECOP Terminals P & C)

Dictates the compressor speed (2,000 to 4,000 RPM) via a 200 Hz open-collector PWM duty cycle.

- **Circuit**: Use a PC817 optocoupler. Connect the ESP32's `compressor_pwm_pin` (GPIO10) to the optocoupler's anode (Pin 1) via a resistor (e.g., 330Ω–1kΩ), and connect ESP32 Ground to the cathode (Pin 2). Connect the optocoupler's collector (Pin 4) to SECOP Terminal `P`, and the emitter (Pin 3) to SECOP Terminal `C`.
- **Power Optimization**: When the compressor is off, FridgeTroll automatically sets the PWM duty cycle to 0.0%, de-energizing the optocoupler LED to conserve energy and prevent optocoupler aging.

### 3. Compressor Fan

Controls the cooling fan for the compressor itself.

- **Circuit**: The Noctua PWM fan can be driven directly by the ESP32. Connect the 12V and Ground pins of the fan to a 12V power supply and common Ground. Connect the fan's PWM pin directly to the ESP32's `compressor_fan_pwm_pin`. Connect the fan's tachometer pin directly to the ESP32's `compressor_fan_speed_pin` (the ESP32 will handle the pull-up internally).

### 4. Door Switch

Monitors if the fridge door is open.

- **Circuit**: Use a PC817 optocoupler. Connect the switch across the optocoupler's LED side (with appropriate resistor). The output side connects to the ESP32 `door_pin` (using `INPUT_PULLUP`) and Ground.
- **Sensing Logic**: To sense the door status, solder a +12V lead to the switched side of the stock door switch (the one that controls the fridge light). Ensure the "input" side of the switch has constant +12V. This allows the optocoupler to detect the 12V signal when the door is open/closed, isolating the 12V circuit from the ESP32.

### 5. Fridge Fan (Internal)

Controls the circulation fan inside the fridge compartment.

- **Circuit**: Use a 2N2222 NPN transistor to switch the fan's ground connection. Connect the ESP32 `fridge_fan_pin` to the transistor's base via a resistor (e.g., 1kΩ). Connect the fan's negative lead to the transistor's collector, and the emitter to Ground.
- **Critical**: Place a flyback diode (e.g., 1N4007) across the fan's positive and negative terminals (cathode to positive, anode to negative) to protect the transistor from voltage spikes when the motor turns off.

## Physical Installation & Wiring

### Component Mounting

A successful FridgeTroll installation requires precise placement of sensors and fans for optimal thermal control:

- **Fridge Sensor**: Mounted inside the main refrigerator compartment to monitor food-safe temperatures.
- **Freezer Sensor**: Mounted inside the freezer compartment.
- **Compressor Sensor**: Mounted directly against the compressor body (using thermal tape or a clip) to monitor for overheating.
- **Ambient Sensor**: Mounted in the air intake path, directly in front of the compressor cooling fan, to measure the temperature of the air being used to cool the coils.
- **Compressor Fan**: The stock fan should be replaced with the high-performance Noctua PWM fan to allow the ESP32 to ramp cooling based on compressor temperature.
- **Fridge Fan**: This fan should be mounted to facilitate air exchange between the freezer and fridge. A great location is in one of the existing baffle holes in the drip tray between the two compartments.

### Parasitic Wiring for DS18B20

To simplify wiring, the DS18B20 sensors are intended to be used in **Parasitic Power Mode**. This allows all sensors to share a single data line and ground.

- **Wiring**: For each sensor, tie the **VDD (+)** and **GND (-)** pins together and connect them to the system Ground.
- **Data Bus**: All **DQ (Data)** wires from the four sensors are connected together and run to the ESP32's `one_wire_bus_pin`.
- **Pull-up**: Ensure there is a ~4.7kΩ pull-up resistor between the Data line and the ESP32's 3.3V rail.

### Cable Routing Tips

Reusing existing paths can save significant time and avoid drilling into the fridge chassis:

- **Thermostat Rewiring**: You can often reuse the wires from the original mechanical thermostat to carry the 1-Wire data bus into the fridge compartment, as well as the door sensor +12V.
- **Freezer Access**: Use the existing temperature probe tube (that originally housed the mechanical sensing bulb) as a conduit for the freezer DS18B20 wiring.
- **Fan Cables**: Fridge fan wires can often be sneaked through the same hole used by the refrigerant tubes to get from the compressor area into the internal compartments.

# Software Configuration

Once FridgeTroll is installed and connected to your network, most configuration can be handled directly through the built-in Web UI.

## 1. Setting Sensor IDs

FridgeTroll uses DS18B20 1-Wire sensors, each of which has a unique 64-bit hardware address. To map your physical sensors to the correct software entities (Fridge, Freezer, Compressor, Ambient):

1.  Open the FridgeTroll Web UI in your browser.
2.  Navigate to the **Debug Tools** section.
3.  Click the **Dump Sensor IDs** button.
4.  Open the ESPHome logs (either via the web interface or serial monitor). You will see a list of discovered sensor addresses in the format `0xXXXXXXXXXXXXXXXX`.
5.  Match these addresses to your physical sensors (you can dip a sensor in warm water to see which one fluctuates in the logs).
6.  Scroll to the **Sensor Addresses** section in the Web UI.
7.  Paste the 18-character hex address (including the `0x` prefix) into the corresponding text field for each sensor.
8.  The changes are applied immediately; no restart is required for sensor mapping.

## 2. Telemetry & Logging (UDP)

FridgeTroll can send high-frequency telemetry to InfluxDB and system logs to a Syslog server via UDP. These settings are found in the **UDP Configuration** section.

- **Syslog IP / Port**: Enter the destination IP address and port (default 514) for your Syslog collector.
- **InfluxDB IP / Port**: Enter the destination IP address and port (default 8089) for your InfluxDB instance. FridgeTroll uses the InfluxDB UDP listener (Line Protocol).

> **Important:** Changes to InfluxDB UDP settings apply immediately, but Syslog configuration changes require a device restart to take effect. Use the **Restart** button in the **Host** section after updating Syslog settings.

## 3. Tuning Performance

The **Settings** section allows you to fine-tune the cooling algorithms and operational limits directly from the Web UI or Home Assistant without reflashing:

### Compressor Control (Asymmetric PD + Feedforward)

FridgeTroll regulates freezer temperature using an asymmetric Proportional-Derivative (PD) controller augmented by ambient and circulation fan feedforward terms:

- **Proportional P-Multiplier** (`pd_p_multiplier_number`): Scales compressor speed proportionally to the positive temperature error above target. Increasing this produces a more aggressive speed boost when the freezer warms up.
- **Derivative Cooling D-Multiplier** (`pd_d_multiplier_number`): Acts as a dynamic brake while actively cooling. When the compartment is chilling rapidly, this derivative term throttles the compressor speed down in advance to prevent undershoot and freezing.
- **Derivative Warming D-Multiplier** (`pd_d_multiplier_warming_number`): Detects rapid warming trajectories (such as warm food loading or extended door openings) and ramps up compressor RPM to arrest temperature rise.
- **Ambient Feedforward Multiplier & Threshold** (`ambient_ff_multiplier_number`, `ambient_ff_threshold_number`): When the ambient temperature around the refrigerator exceeds this threshold, a feedforward offset is added to compressor RPM to compensate for heightened thermal conduction through the cabinet walls.
- **Recovery Evaluation Window** (`compressor_window_number`): Time window (in seconds) of temperature history used to calculate the real-time cooling/warming rate (°C/h).
- **Compressor Speed Min Interval** (`compressor_speed_change_interval_number`): Minimum duration between compressor speed changes to prevent rapid modulation and controller wear.

### Fridge Fan Circulation & Freeze Protection

- **Fridge Fan Min Speed** (`fridge_fan_min_speed_number`): The minimum baseline speed percentage when the fridge thermostat calls for cooling.
- **Fridge Fan Max Speed** (`fridge_fan_max_speed_number`): The maximum speed percentage when the fridge compartment temperature is significantly above setpoint.
- **Fridge Fan Ramp Band** (`fridge_fan_ramp_band_number`): The temperature window above setpoint over which the fan speed is proportionally ramped from min to max.
- **Freeze Protection**: If fridge compartment temperature drops to ≤ 1.0°C, the compressor is forced off unconditionally until the fridge warms back to ≥ 1.5°C, guaranteeing fresh food never freezes.

### Defrost Management

- **Defrost Target** (`defrost_target_temp`): The target freezer temperature during the pre-defrost "deep chill" cycle, ensuring stored items remain safely frozen during the subsequent rest phase.

## Advanced: ESPHome Package Integration

If you manage your smart home devices through a centralized ESPHome dashboard or configuration directory, you can cleanly integrate FridgeTroll as an **ESPHome Package**. This allows you to compile from source, keep your custom credentials secure in a centralized `secrets.yaml`, configure your local timezone, and disable the standalone Improv provisioning service since credentials are pre-configured.

### Example Configuration (`ESPHome/config/fridgetroll.yaml`)

To extend FridgeTroll as a package and override Wi-Fi, API, timezone, and OTA configurations:

```yaml
substitutions:
  # Optionally override timezone or device name
  timezone: "America/Chicago"

packages:
  # Reference the base fridgetroll.yaml package file
  fridgetroll: !include ../../FridgeTroll/fridgetroll.yaml

# Remove Improv Serial and Improv Wi-Fi provisioning components (not needed for centralized configs)
improv_serial: !remove
improv_ble:
  !remove # Supply your network credentials from your central secrets.yaml

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

ota:
  - id: !extend esphome_ota
    password: !secret ota_password

api:
  encryption:
    key: !secret api_encryption_key
```

Once configured, you can build, flash, and manage your FridgeTroll node alongside all your other ESPHome nodes:

```bash
esphome run fridgetroll.yaml
```
