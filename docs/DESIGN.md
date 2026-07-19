# Fish Around and Find Out — Aquaponics Monitor
## Design Document v0.5

---

## Project Overview

An ESP32-based aquaponics monitoring and control system written in Rust. Designed for a single IBC tote backyard aquaponics system running channel catfish and leafy greens/herbs in Atwater, CA (Central Valley). Extreme climate conditions — summers up to 110°F, winters near freezing — are a primary design constraint.

---

## Hardware Target

- **Microcontroller:** ESP32 (dual-core, WiFi + Bluetooth built in)
- **Framework:** esp-idf via esp-rs (std enabled)
- **Language:** Rust
- **Power:** Mains powered via outdoor GFCI outlet (solar expansion possible later)
- **Power-loss backup:** Small battery or supercap dedicated to the ESP32 + radio, sized only to survive long enough to send one last-gasp MQTT alert before shutdown — see **System State Monitoring** below. Without this, the device meant to report a power outage goes dark at the exact moment power is lost, defeating the alert.
- **Connectivity:** WiFi to home network
- **Enclosure:** Weatherproof, IP65+ minimum. Atwater summers hit 110°F ambient in direct sun, and a sealed box can run meaningfully hotter inside — plan for ventilation or shading rather than a fully sealed sun-facing box, to keep the ESP32 and DHT22 within their rated operating temperature.

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

### Water Chemistry — Hybrid Sensing Plan

Chemistry monitoring splits into three tiers based on how dangerous and how fast-moving each parameter is, not a single sensor brand across the board:

1. **pH + Dissolved Oxygen — automated, cheap.** These drift slowly enough that a budget analog probe catches a problem in time.
2. **Ammonia — automated, worth the cost.** The fastest-acting fish killer of the group; a spike can turn toxic in hours, so this is the one probe where a real-time alert matters more than saving money.
3. **Nitrite + Nitrate + Alkalinity (KH) — manual, no sensor.** These build up or deplete over days/weeks, giving plenty of warning from a cheap test kit checked on a schedule. KH in particular is what silently depletes as nitrification produces acid — once it bottoms out, pH crashes suddenly instead of drifting predictably, which is one of the most common real-world aquaponics failure modes. All three come from the same test kit already budgeted for nitrite/nitrate, at no extra cost.

| Parameter | Sensor | Target Range | Alert Threshold |
|-----------|--------|-------------|-----------------|
| pH | DFRobot Gravity Analog pH (ESP32 ADC) | 6.8 – 7.2 | < 6.5 or > 7.8 |
| Dissolved Oxygen | DFRobot Gravity Analog DO (ESP32 ADC) | > 5 ppm | < 4 ppm |
| Ammonia — un-ionized (NH₃) | Atlas Scientific EZO-NH3 | 0 ppm | > 0.05 ppm (chronic) / toxicity onset ~0.02 ppm |
| Ammonia — total (TAN) | Atlas Scientific EZO-NH3 | 0 ppm | > 1 ppm (reference only — see note) |
| Nitrite (NO₂) | **Manual** — liquid test kit (e.g. API Freshwater Master Test Kit), not wired to the ESP32 | 0 ppm | > 0.5 ppm — check 2-3x/week, more often in summer heat |
| Nitrate (NO₃) | **Manual** — same test kit as nitrite | 5 – 150 ppm | > 200 ppm — check 2-3x/week, more often in summer heat |
| Alkalinity (KH) | **Manual** — same test kit as nitrite/nitrate | 100 – 180 ppm (as CaCO₃) | < 60 ppm — risk of sudden pH crash as buffering capacity depletes |

> **Ammonia threshold note:** Un-ionized ammonia (NH₃) is the toxic species to channel catfish — literature consensus puts chronic-safe exposure below ~0.05 ppm, with toxicity effects starting around 0.02 ppm, and toxicity increasing at higher pH/temperature. This is very different from a flat 1 ppm total ammonia threshold. **TODO: verify which ammonia species the Atlas Scientific EZO-NH3 probe actually reports (un-ionized NH₃ vs. total ammonia nitrogen/TAN)** before trusting either threshold in production alerting — the probe spec should confirm this, but do not assume.

> **Stretch goal:** if budget allows later, nitrite and nitrate can be automated the same way as ammonia (Atlas Scientific EZO-NO2 / EZO-NO3), reusing the same EZO driver already written for the ammonia probe. Not planned for the initial build.

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

## Hardware Alternatives — Probe Comparison

Atlas Scientific EZO circuits are the default plan (see Sensors table above), but they're not the only option for every parameter. Comparison researched for this project:

### pH
| | Atlas Scientific EZO-pH | DFRobot Gravity Analog pH |
|---|---|---|
| Cost | ~$70-90 | ~$15-25 |
| Accuracy | ±0.002 pH | ±0.1 pH |
| Calibration interval | Monthly (stable) to weekly (field/fluctuating temp) | Monthly, weekly if heavily used |
| Interface | UART/I2C, digital ASCII output | Analog voltage → ESP32 ADC |
| Rust work needed | Custom EZO driver (no crate exists) | None — `esp-idf-hal` ADC read |
| Continuous outdoor duty | Built for it | DFRobot markets it as "lab grade" — less proven for 24/7 outdoor temp swings |

Target range is 6.8–7.2 (a 0.4 pH window), so DFRobot's ±0.1 accuracy is still usable for threshold alerting, just noisier than Atlas's ±0.002.

### Dissolved Oxygen
| | Atlas Scientific EZO-DO | DFRobot Gravity Analog DO |
|---|---|---|
| Cost | ~$150+ | ~$50-60 |
| Type | Galvanic | Galvanic, no polarization wait |
| Maintenance | Periodic membrane/electrolyte refresh | Membrane cap: 1-2mo (muddy water) / 4-5mo (clean); electrolyte (0.5 mol/L NaOH) refill monthly |
| Noise handling | Good | Explicitly marketed as low-jitter even with pump/heater noise nearby — relevant since our pump runs continuously |
| Rust work needed | Custom driver | None — analog ADC |

DO is the parameter where DFRobot looks most competitive: purpose-built for continuous immersion near pump noise, a third of the cost, zero custom driver work.

### Ammonia / Nitrite / Nitrate — no real budget alternative for automated sensing
DFRobot does not sell consumer-affordable ISE probes for NH₃/NO₂/NO₃. Atlas Scientific EZO is effectively the only option at the hobbyist price point for automated sensing of these three. Also checked DFRobot's industrial RS485 ammonia sensor (SEN0711) as a possible cheaper automated alternative — it's actually *more* expensive (~$359) than Atlas EZO-NH3, so it's not a cost win. The real lever isn't a cheaper sensor, it's dropping automation for the parameters where manual testing is safe enough (see below).

### Decision: hybrid plan
Rather than choosing one brand across the board, the plan splits by how dangerous and fast-moving each parameter is:

- **pH + DO → DFRobot analog.** Cheap (~$65-85 combined), automated, and accurate enough for these slow-moving parameters. Only costs an `esp-idf-hal` ADC read, no custom driver.
- **Ammonia → Atlas Scientific EZO-NH3.** The one parameter that can turn toxic in hours, so it keeps the real-time automated sensor (~$200-230) despite the cost — this is where the EZO Rust driver actually gets written.
- **Nitrite + Nitrate → manual test kit, no sensor.** These move slowly enough that a $20-30 liquid test kit checked 2-3x/week (more in summer heat) catches a problem in time. Automating them would mostly buy convenience, not safety, at ~$400-500 combined — not worth it for a Year-1 hobby build.

Estimated chemistry cost under this plan: **~$300-340** (DFRobot pH+DO + Atlas EZO-NH3 + test kit), versus ~$916-1,140 for automating all five via Atlas EZO, or ~$85-115 for going fully manual and skipping ammonia automation too (not recommended — see above).

---

## System State Monitoring

- **Pump status** — on/off, confirm via flow sensor
- **Heater status** — on/off (winter use)
- **Flood/drain cycle** — grow bed timing via configurable schedule
- **Power status** — detect outage condition. Requires the battery/supercap backup noted under **Hardware Target**: on loss of mains power, the ESP32 runs briefly on backup power, immediately sends a "power lost" MQTT alert, then shuts down cleanly rather than browning out mid-operation. On restore, it reconnects and sends a "power restored" message so outage duration is known.

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
- Nitrate > 150 ppm *(manual test-kit reading in the hybrid plan — not an ESP32-triggered alert unless/until the stretch-tier EZO-NO3 sensor is added)*
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
- **MQTT** — lightweight messaging protocol for alerts and data. **Security:** use TLS plus username/password (or client cert) auth on the broker from the start, not just once Phase 3 ships. Phase 3 puts pump/heater/aeration relay control behind MQTT commands — an unauthenticated broker on the home network means anything that can reach it can shut off aeration. Cheaper to design in now than retrofit later.
- **OTA updates** — update firmware without physical access to device. Should be signed/authenticated once implemented — an unauthenticated OTA endpoint is a remote-code-execution risk on a device with relay control over pump and heater.

---

## Rust Crates (Likely)

| Crate | Purpose |
|-------|---------|
| `esp-idf-hal` | ESP32 hardware abstraction |
| `esp-idf-svc` | WiFi, MQTT (TLS-capable via mbedTLS bindings), NTP services |
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

### Phase 2 — Chemistry Sensors (Hybrid Plan)
- [ ] Integrate DFRobot Gravity analog pH sensor (ESP32 ADC, no custom driver)
- [ ] Integrate DFRobot Gravity analog dissolved oxygen sensor (ESP32 ADC, no custom driver)
- [ ] Write Rust driver for Atlas Scientific EZO UART/I2C protocol (no existing crate — protocol is plain ASCII commands, straightforward to implement)
- [ ] Integrate Atlas Scientific EZO-NH3 ammonia sensor using that driver
- [ ] Set up manual testing routine for nitrite/nitrate (liquid test kit, 2-3x/week) — not wired to the ESP32 in this phase
- [ ] Send alerts via MQTT

**Stretch / "baller" tier** (budget-permitting, not planned for initial build):
- [ ] Automate nitrite via Atlas Scientific EZO-NO2, reusing the ammonia driver
- [ ] Automate nitrate via Atlas Scientific EZO-NO3, reusing the ammonia driver

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
| Predator protection | Netting/cover over the tank — raccoons and herons are realistic backyard threats in this area, not an electronics problem but cheap to prevent |

---

## Notes

- Physical aquaponics system build comes FIRST — ESP32 project starts after system is running
- Start with cheap sensors (DS18B20, DHT22, ultrasonic) before investing in chemistry probes
- ~~Atlas Scientific probes are gold standard but $50-80 each~~ — corrected: that figure was the bare EZO circuit only, not the probe. All-in cost per parameter is actually $140-260. See the hybrid plan above for how this project splits cheap (DFRobot) vs. worth-it (Atlas EZO ammonia) vs. manual (nitrite/nitrate) to control cost.
- Battery backup air pump is critical hardware regardless of this project
- Year 1: personal/non-commercial use. Year 2: farmers market sales under "Fish Around and Find Out" brand
- Gap review (July 2026): added ESP32 power-loss backup design, alkalinity (KH) tracking, MQTT/OTA security requirements, enclosure thermal rating, and predator protection — see Hardware Target, Water Chemistry, Connectivity, and Physical System Reference above

---

*v0.5 — closed gap review: ESP32 power-loss backup for outage alerting, alkalinity (KH) tracking added to manual test routine, MQTT/OTA security requirements, enclosure thermal/IP rating, predator protection note*

*v0.4 — adopted hybrid chemistry-sensing plan (DFRobot pH/DO, Atlas EZO ammonia, manual nitrite/nitrate), restructured Phase 2 build plan, corrected probe cost note*

*v0.3 — added probe hardware comparison (Atlas Scientific vs. DFRobot) with recommendation*

*v0.2 — revised ammonia threshold (un-ionized vs. total), added firmware implementation notes (DS18B20 RMT, DHT22 timing, EZO driver gap)*
