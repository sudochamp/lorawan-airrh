# Abstract

## Why

Monitoring temperature and humidity matters for farms, HVAC, cold storage, and industrial processes. Cheap IoT sensors are unreliable and cloud-dependent. Industrial sensors cost \$200+ and do more than needed. This device uses LoRa for long-range wireless, e-ink for ultra-low power, and SD card for local logging. Battery lasts years.

## Use Case

**Users:**
- Farmers (greenhouses, storage)
- Storage facilities
- Cold storage operators
- HVAC techs
- Researchers in remote areas

**How it works:**
Deploy sensor. E-ink shows live data. No phone needed. SD card logs locally. LoRa sends readings to gateway. QR code for initial WiFi/BLE setup.

Works standalone: display + logging without gateway. Years on batteries.

## Market

**What exists:**
- Consumer IoT (Xiaomi, Govee): Cheap but short range, cloud-dependent, months of battery
- Industrial (Omega, Vaisala): \$200-\$1000+, accurate but overkill
- DIY LoRa nodes: Long range but no display, poor enclosures

**Gap:**
Sub-\$50 LoRa sensor with display and SD logging. IP65+. 5+ years on batteries. No app needed.

**Who needs this:**
- Small farms and hobbyists
- Places without WiFi
- Users avoiding cloud dependency

## Initial Component Ideas

**Sensors:**
- SHT40/SHT45 (Sensirion): High-accuracy temp/humidity, I2C, low power
- BME680 if air quality (VOC) needed

**Display:**
- 4.2" 300x400 e-ink (SSD1683 controller, Adafruit #6381)
- Ultra-low power (only draws current during refresh)
- Skeuomorphic analog thermometer/hygrometer UI

**Wireless:**
- LoRa transceiver: RFM95W (SX1276) or SX1262 (newer, lower power)
- Optional: ESP32-C3 or nRF52 for WiFi/BLE setup (disable after config to save power)

**MCU:**
- STM32L4 or nRF52840: Ultra-low-power, sufficient flash/RAM for graphics
- Deep sleep between measurements (wake every 5-15 min)

**Storage:**
- microSD card for local datalogging (CSV/JSON)
- RTC (real-time clock) for timestamping: DS3231 or MCU internal RTC with backup battery

**Power:**
- 2x AA batteries or CR2032 coin cells
- Ultra-low quiescent current boost converter (TPS61291 or similar)
- Target: 5+ years battery life @ 15-min logging intervals

**Enclosure:**
- Weatherproof (IP65), vented for accurate humidity sensing
- Radiation shield for outdoor use (prevents solar heating from skewing temp readings)
- Slim form factor (stick-like, easy to mount)

**Setup:**
- QR code on e-ink display for WiFi provisioning (if WiFi radio included)
- Or: Bluetooth Low Energy for phone-based config

## Open Questions

- LoRa-only or add WiFi/BLE for setup?
- Power: AA, coin cell, or Li-Ion?
- Log interval: 5 min or 15 min?
- Display: button refresh or auto?
- LoRaWAN or custom protocol?
- Factory cal or user calibration?
- Add soil moisture or wired sensor input?
- Temp range: -10°C to 60°C or -40°C to 85°C?
- Program via USB-C SWD or wireless?

