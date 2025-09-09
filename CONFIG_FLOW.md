# AQMonitor1 YAML Configuration Analysis

## Overview
This repository contains `aqmonitor1.yaml`, an [ESPHome](https://esphome.io) configuration targeting an ESP32-S2 board (`esp32-s2-saola-1`) with an ILI9341 LCD. The YAML describes sensors, display logic, and connectivity, which ESPHome translates into C++ code compiled with the Arduino framework.

The configuration monitors multiple environmental sensors (temperature, humidity, CO₂, VOCs, particulate matter, etc.), renders readings on a TFT LCD, and reports data to Home Assistant via the native API and a local web server.

## System Configuration
- **Hardware:** ESP32-S2 with PSRAM (quad mode, 80 MHz). ILI9341 display on SPI, PMSX003 particulate sensor and ZE08 formaldehyde sensor on UART, multiple I²C sensors on two busses.
- **Firmware:** Built with ESPHome `min_version 2024.6.0` using the Arduino framework. Logging, OTA updates, improv serial provisioning, captive portal, and a simple web server are enabled.
- **Networking:** Wi-Fi credentials are provided with a fallback access point. Home Assistant API uses encryption and supports OTA updates.

## Backlight and Input Handling
- A `globals` variable `dbright` tracks display brightness state.
- An LEDC PWM output (`GPIO15`) drives the backlight through a `monochromatic` light component. 
- A GPIO button (`GPIO13`) cycles brightness levels via a `lambda` in `on_press`. The state machine toggles between four brightness levels, turning the backlight on/off as appropriate.

## Fonts and Palette
Fonts are fetched from Google Fonts (`Rubik` and `Orbitron`) and assigned to identifiers used later in drawing. A custom color palette defines IDs (white, grey, light_grey, green, yellow, orange, red, purple, blue) with hex values for use in display rendering.

## Display Rendering
An `ili9xxx` display component provides a custom `lambda` that implements the LCD UI directly in C++.

### Display Lambda Walkthrough
1. **Setup:** The lambda clears the screen (`it.fill(Color::BLACK)`) and caches fonts, dimensions, and palette IDs for reuse.
2. **State Gathering:** Local variables pull the latest readings from each sensor via `id(<sensor>).state`.
3. **Color Mapping:** For every measurement, a chain of `if` statements maps raw values to palette colors and stores the result (e.g., `co2_c`, `pm25_c`).
4. **Fallback Sensor Selection:** Arrays of temperature and humidity readings are iterated to choose the first non‑`NaN` value, ensuring a reading even if some sensors fail.
5. **Reusable Section Renderer:** Arrays for names, values, and colors are populated for each category (Particles, CO₂, VOC). A loop draws a separator line, heading, units, and up to three value columns while skipping any `NaN` entries.
6. **VOC Index Lines:** After the general VOC section, optional lines show SGP41 VOC and NOx indexes with range hints.

This inline C++ lambda orchestrates color logic, layout, and sensor presentation at each display refresh.

## Sensor Management
The configuration defines numerous sensors with update intervals and filters:
- **Template Sensor:** `ch2o_ze08` reads ZE08 formaldehyde data from a UART dump, publishing ppb values after filtering and discarding out-of-range readings.
- **ZE08 UART Parsing:** A `uart` debug `lambda` checks 9‑byte frames for the correct header (`0xFF 0x17`), combines bytes into a value, and publishes it to `ch2o_ze08`.
- **AHT20/BMP280/SHT40/SCD40:** Provide temperature, humidity, and pressure via I²C. The AHT20 temperature is converted from °C to °F.
- **ENS160:** Supplies eCO₂, TVOC, and AQI with moving-average filters and environmental compensation from a paired AHT20.
- **SGP30 and SGP4x:** Offer eCO₂/TVOC and VOC/NOx indexes respectively, with baseline storage and compensation.
- **PMSX003:** Delivers particulate matter counts for PM1, PM2.5, and PM10 via UART at 30‑second intervals.
- **Utility Sensors:** Uptime, heap free, and loop time diagnostics are included but disabled by default.

ESPHome generates C++ classes for each sensor component, handling initialization, periodic polling, and state publishing. The display lambda accesses these states via the `id()` function, ensuring the UI reflects the most recent data.

## Data Reporting
All sensors are named for Home Assistant and exposed through the ESPHome API (`api:` block). The device can also be monitored via the built-in web server. Update intervals dictate how often new data is polled and published. Filters (e.g., exponential moving averages) smooth readings before they are displayed or reported.

## Flow Summary
1. **Startup:** ESP32 initializes hardware, sets up Wi-Fi/OTA, and starts logging and API services. Sensors on I²C, UART, and SPI are configured.
2. **Polling:** Each sensor’s driver periodically polls hardware and updates internal state variables.
3. **Display Refresh:** On every display update cycle, the lambda fetches sensor values, determines colors/labels, and renders the UI onto the ILI9341 LCD.
4. **Reporting:** Updated sensor states are transmitted to Home Assistant through the encrypted API and optionally displayed via the web server.
5. **User Interaction:** Pressing the button adjusts `dbright` and backlight intensity, affecting the LCD brightness.

This configuration orchestrates a comprehensive air quality monitoring station, blending a rich set of sensors with custom LCD output and network reporting.

