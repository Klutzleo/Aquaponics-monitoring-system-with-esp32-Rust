# Fish Around and Find Out — Aquaponics Monitor
## Design Document v0.1

---

## Project Overview

An ESP32-based aquaponics monitoring and control system written in Rust. Designed for a single IBC tote backyard aquaponics system running channel catfish and leafy greens/herbs in Atwater, CA (Central Valley). Extreme climate conditions — summers up to 110°F, winters near freezing — are a primary design constraint.

---

## Hardware Target

- **Microcontroller:** ESP32 (dual-core, WiFi + Bluetooth built in)
- **Framework:** esp-idf via esp-rs (std enabled)
- **Language:** Rust
- **Power:** Mains powered via outdoor GFCI outlet (solar expansion possible later)
- **Connectivity:** WiFi to home network

---

## System Architecture

```
Sensors → ESP32 → Data Processing → Alerts / Logging / Control
```

- Sensor readings collected on a configurable interval
- Values compared against safe thresholds
- Out-of-range values trigger alerts (push notification / email)
- Optional: relay control for pump, heater, dosing pumps
- Data logged over time for trend analysis

---

## Sensors & Parameters

### Water Chemistry
| Parameter | Sensor | Target Range | Alert Threshold |
|-----------|--------|-------------|-----------------|
| pH | Atlas Scientific EZO-pH | 6.8 – 7.2 | < 6.5 or > 7.8 |
| Ammonia — un-ionized (NH₃) | Atlas Scientific EZO-NH3 | 0 ppm | > 0.05 ppm (chronic) / toxicity onset ~0.02 ppm |
| Ammonia — total (TAN) | Atlas Scientific EZO-NH3 | 0 ppm | > 1 ppm (reference only — see note) |
| Nitrite (NO₂) | Atlas Scientific EZO-NO2 | 0 ppm | > 0.5 ppm |
| Nitrate (NO₃) | Atlas Scientific EZO-NO3 | 5 – 150 ppm | > 200 ppm |
| Dissolved Oxygen | Atlas Scientific EZO-DO | > 5 ppm | < 4 ppm |

> **Ammonia threshold note:** Un-ionized ammonia (NH₃) is the toxic species to channel catfish — literature consensus puts chronic-safe exposure below ~0.05 ppm, with toxicity effects starting around 0.02 ppm, and toxicity increasing at higher pH/temperature. This is very different from a flat 1 ppm total ammonia threshold. **TODO: verify which ammonia species the Atlas Scientific EZO-NH3 probe actually reports (un-ionized NH₃ vs. total ammonia nitrogen/TAN)** before trusting either threshold in production alerting — the probe spec should confirm this, but do not assume.

### Physical
| Parameter | Sensor | Notes |
|-----------|--------|-------|
| Water Temperature | DS18B20 | Waterproof, 1-Wire protocol, ~$3. Use the ESP32 RMT peripheral for 1-Wire timing rather than bit-banging — naive bit-banged implementations see checksum failures at roughly 1-in-50 transactions. |
| Water Level | Ultrasonic (HC-SR04) | Detects low water / evaporation loss |
| Flow Rate | YF-S201 Hall effect sensor | Confirms pump is actually running |

### Environment
| Parameter | Sensor | Notes |
|-----------|--------|-------|
| Air Temperature | DHT22 | Combined temp/humidity sensor. Timing-sensitive: build in release mode, poll no more than every ~500ms, and configure the GPIO as open-drain with no internal pull — otherwise reads spuriously time out. |
| Humidity | DHT22 | Same sensor as above |

---

## System State Monitoring

- **Pump status** — on/off, confirm via flow sensor
- **Heater status** — on/off (winter use)
- **Flood/drain cycle** — grow bed timing via configurable schedule
- **Power status** — detect outage condition

---

## Alert System

### Critical Alerts (immediate notification)
- Ammonia > 1 ppm
- pH out of range
- Dissolved oxygen < 4 ppm
- Water temp > 90°F or < 55°F (channel catfish thresholds)
- Water level low
- Pump failure / no flow detected
- Power outage detected

### Warning Alerts (non-urgent)
- Nitrate > 150 ppm
- pH drifting (trending toward threshold)
- Air temp extreme (> 105°F or < 35°F)

### Delivery Methods (TBD)
- Push notification via MQTT + phone app
- Email alert
- On-device buzzer / LED indicator

---

## Control Outputs (Phase 2)

> Phase 1 is monitoring only. Control features added after system is proven stable.

- **Relay 1** — Main water pump (on/off)
- **Relay 2** — Air pump / backup aeration
- **Relay 3** — Water heater (winter)
- **Relay 4** — pH dosing pump (auto pH correction — advanced)

---

## Data Logging

- Readings logged locally to flash storage
- Periodic sync to home server or cloud (TBD)
- CSV or JSON format for later analysis
- Goal: identify trends before they become problems

---

## Connectivity

- **WiFi** — primary connection to home network
- **MQTT** — lightweight messaging protocol for alerts and data
- **OTA updates** — update firmware without physical access to device

---

## Rust Crates (Likely)

| Crate | Purpose |
|-------|---------|
| `esp-idf-hal` | ESP32 hardware abstraction |
| `esp-idf-svc` | WiFi, MQTT, NTP services |
| `embedded-hal` | Sensor trait abstractions |
| `ds18b20` | DS18B20 temperature sensor driver |
| `dht-sensor` | DHT22 driver |
| `heapless` | No-heap data structures |
| `serde` + `serde_json` | Data serialization |

---

## Phased Build Plan

### Phase 1 — Sensors & Monitoring (Start Here)
- [ ] Get ESP32 dev board
- [ ] Read water temp via DS18B20
- [ ] Read air temp/humidity via DHT22
- [ ] Read water level via ultrasonic sensor
- [ ] WiFi connection
- [ ] Log readings to serial output
- [ ] Basic threshold alerting

### Phase 2 — Chemistry Sensors
- [ ] Integrate Atlas Scientific pH probe
- [ ] Integrate dissolved oxygen probe
- [ ] Ammonia / nitrite / nitrate (if budget allows)
- [ ] Write Rust driver for Atlas Scientific EZO UART/I2C protocol (no existing crate — protocol is plain ASCII commands, straightforward to implement)
- [ ] Send alerts via MQTT

### Phase 3 — Control & Automation
- [ ] Relay board integration
- [ ] Pump control and scheduling
- [ ] Flood/drain cycle automation
- [ ] Auto pH dosing (stretch goal)

### Phase 4 — Dashboard
- [ ] Web dashboard for live readings
- [ ] Historical data charts
- [ ] Mobile-friendly interface

---

## Physical System Reference

| Component | Spec |
|-----------|------|
| Fish tank | 275–330 gal IBC tote |
| Fish species | Channel catfish (Ictalurus punctatus) |
| Fish count | 25–30 to start |
| Grow beds | 2x IBC tote tops |
| Grow media | Expanded clay pebbles |
| Flood cycle | 15 min on / 45 min off |
| Target pH | 6.8 – 7.2 |
| Target water temp | 65 – 85°F |
| Location | Atwater, CA — Merced County |
| Climate | Extreme heat summers (110°F), near-freeze winters |

---

## Notes

- Physical aquaponics system build comes FIRST — ESP32 project starts after system is running
- Start with cheap sensors (DS18B20, DHT22, ultrasonic) before investing in Atlas Scientific probes
- Atlas Scientific probes are gold standard but $50-80 each — add in Phase 2
- Battery backup air pump is critical hardware regardless of this project
- Year 1: personal/non-commercial use. Year 2: farmers market sales under "Fish Around and Find Out" brand

---

*v0.2 — revised ammonia threshold (un-ionized vs. total), added firmware implementation notes (DS18B20 RMT, DHT22 timing, EZO driver gap)*
