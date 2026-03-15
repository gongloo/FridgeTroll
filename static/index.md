# FridgeTroll

An ESPHome-based smart controller for SECOP compressor-driven refrigerators, providing advanced temperature management, stall detection, and telemetry.

## Overview
FridgeTroll is designed as an off-the-shelf, installable firmware primarily targeted at the **SECOP 101N2000** compressor controller (though it should work with most modern SECOP controllers). It replaces the standard mechanical thermostat with an ESP32, offering fine-grained control over compressor speed, dedicated fridge and compressor fan management, and rich telemetry for Home Assistant via InfluxDB and native ESPHome APIs.

# Installation

You can use the button below to install the pre-built firmware directly to your device via USB from the browser.

<esp-web-install-button manifest="firmware/fridgetroll.manifest.json"></esp-web-install-button>

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

## Bill of Materials (BOM)
To build the FridgeTroll hardware interface, you will need:
* **Microcontroller**: Wemos S3 Mini (or similar ESP32 board)
* **Power Supply**: Wemos DC Power Shield (or similar 5V USB-C power supply)
* **Fridge Fan**: Coolerguys 12vDC Waterproof IP67 Fan (High Speed, 25x10mm) (or similar 12V DC fan)
* **Compressor Fan**: Noctua NF-P12 redux-1700 PWM (or similar 120mm PWM fan)
* **Temperature Sensors**: DS18B20 1-Wire sensors (x4 recommended: Fridge, Freezer, Compressor, Ambient)
* **Optocouplers**: PC817 (x3)
  * Compressor Enable Pin (T)
  * Compressor PWM Pin (P)
  * Door Switch Pin
* **Transistor**: 2N2222 NPN Transistor (for driving the Fridge Fan)
* **Diode**: 1N4007 or similar (Flyback protection for the Fridge Fan)
* **Resistors**: Various (for optocoupler LEDs, pull-ups, and pull-downs depending on your specific board layout)

## Interfacing with the SECOP Controller

The SECOP controller provides several pins that need to be interfaced with the ESP32. Due to voltage differences (the SECOP can Output 12V/24V on some pins), **opto-isolation is required** for the compressor control signals and the door switch to protect the ESP32.

### 1. Compressor Enable (Pin T)
The compressor turns on when Pin C is connected to Pin T (or driven by a PWM signal). 
* **Circuit**: Use a PC817 optocoupler. Connect the ESP32's `compressor_enable_pin` to the optocoupler's anode (via a current-limiting resistor). Connect the optocoupler's collector to the SECOP Pin C, and the emitter to Ground.

### 2. Compressor PWM (Pin P)
This pin dictates the compressor speed.
* **Circuit**: Use a PC817 optocoupler. Connect the ESP32's `compressor_pwm_pin` to the optocoupler's anode (via a resistor). Connect the optocoupler's collector to Pin T and the emitter to Ground (Pin P).

### 3. Compressor Fan
Controls the cooling fan for the compressor itself.
* **Circuit**: The Noctua PWM fan can be driven directly by the ESP32. Connect the 12V and Ground pins of the fan to a 12V power supply and common Ground. Connect the fan's PWM pin directly to the ESP32's `fan_pwm_pin`. Connect the fan's tachometer pin directly to the ESP32's `fan_speed_pin` (the ESP32 will handle the pull-up internally).

### 4. Door Switch
Monitors if the fridge door is open.
* **Circuit**: Use a PC817 optocoupler. Connect the switch across the optocoupler's LED side (with appropriate resistor). The output side connects to the ESP32 `door_pin` (using `INPUT_PULLUP`) and Ground.
* **Sensing Logic**: To sense the door status, solder a +12V lead to the switched side of the stock door switch (the one that controls the fridge light). Ensure the "input" side of the switch has constant +12V. This allows the optocoupler to detect the 12V signal when the door is open/closed, isolating the 12V circuit from the ESP32.

### 5. Fridge Fan (Internal)
Controls the circulation fan inside the fridge compartment.
* **Circuit**: Use a 2N2222 NPN transistor to switch the fan's ground connection. Connect the ESP32 `fridge_fan_pin` to the transistor's base via a resistor (e.g., 1kΩ). Connect the fan's negative lead to the transistor's collector, and the emitter to Ground. 
* **Critical**: Place a flyback diode (e.g., 1N4007) across the fan's positive and negative terminals (cathode to positive, anode to negative) to protect the transistor from voltage spikes when the motor turns off.

## Physical Installation & Wiring

### Component Mounting

A successful FridgeTroll installation requires precise placement of sensors and fans for optimal thermal control:

*   **Fridge Sensor**: Mounted inside the main refrigerator compartment to monitor food-safe temperatures.
*   **Freezer Sensor**: Mounted inside the freezer compartment.
*   **Compressor Sensor**: Mounted directly against the compressor body (using thermal tape or a clip) to monitor for overheating.
*   **Ambient Sensor**: Mounted in the air intake path, directly in front of the compressor cooling fan, to measure the temperature of the air being used to cool the coils.
*   **Compressor Fan**: The stock fan should be replaced with the high-performance Noctua PWM fan to allow the ESP32 to ramp cooling based on compressor temperature.
*   **Fridge Fan**: This fan should be mounted to facilitate air exchange between the freezer and fridge. A great location is in one of the existing baffle holes in the drip tray between the two compartments.

### Parasitic Wiring for DS18B20

To simplify wiring, the DS18B20 sensors are intended to be used in **Parasitic Power Mode**. This allows all sensors to share a single data line and ground.

*   **Wiring**: For each sensor, tie the **VDD (+)** and **GND (-)** pins together and connect them to the system Ground.
*   **Data Bus**: All **DQ (Data)** wires from the four sensors are connected together and run to the ESP32's `one_wire_bus_pin`.
*   **Pull-up**: Ensure there is a ~4.7kΩ pull-up resistor between the Data line and the ESP32's 3.3V rail.

### Cable Routing Tips

Reusing existing paths can save significant time and avoid drilling into the fridge chassis:

*   **Thermostat Rewiring**: You can often reuse the wires from the original mechanical thermostat to carry the 1-Wire data bus into the fridge compartment, as well as the door sensor +12V.
*   **Freezer Access**: Use the existing temperature probe tube (that originally housed the mechanical sensing bulb) as a conduit for the freezer DS18B20 wiring.
*   **Fan Cables**: Fridge fan wires can often be sneaked through the same hole used by the refrigerant tubes to get from the compressor area into the internal compartments.

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

*   **Syslog IP / Port**: Enter the destination IP address and port (default 514) for your Syslog collector.
*   **InfluxDB IP / Port**: Enter the destination IP address and port (default 8089) for your InfluxDB instance. FridgeTroll uses the InfluxDB UDP listener (Line Protocol).

> [!IMPORTANT]
> Changes to UDP settings require a device restart to take effect. Use the **Restart** button in the **Host** section after saving your changes.

## 3. Tuning Performance

The **Settings** section allows you to fine-tune the cooling algorithm without reflashing:

*   **Target Recovery Time**: Sets how aggressively the compressor should ramp up to return to target temperature. 
*   **Thermal Prediction Lead**: Adjusts the "look-ahead" time for predictive braking to prevent overshoot.
*   **Recovery Tolerance**: Defines the allowed deviation from the target cooling rate before adjusting compressor speed.
*   **Recovery Evaluation Window**: The duration of history used to calculate the real-time cooling delta.
*   **Compressor Speed Min Interval**: Prevents excessive oscillation by enforcing a minimum time between speed changes.
