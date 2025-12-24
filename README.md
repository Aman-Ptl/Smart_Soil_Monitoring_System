# Smart Soil Monitoring System

Smart Soil Monitoring System is an IoT prototype for monitoring soil moisture, temperature and humidity with wireless sensor nodes (nRF24L01) and an ESP-based receiver that forwards data to Blynk (or any other dashboard). The system helps producers and hobbyists make data-driven irrigation decisions, reduce water waste, and monitor field conditions remotely.

This README explains:
- What the project is and how it works
- Hardware and software required
- Wiring & calibration
- How to build and flash transmitter (node) and receiver (gateway)
- How to configure Blynk (or alternative) and secure your secrets
- Troubleshooting, suggestions and next steps

---

## Project overview

- Wireless sensor node(s) measure:
  - Soil moisture (analogue sensor)
  - Temperature & humidity (DHT11/DHT22)
  - Optional PIR presence sensor
- Wireless transport: nRF24L01 (RF24 library)
- Receiver: ESP8266 (or similar) connects to Wi‑Fi and forwards sensor values to Blynk (or an MQTT broker)
- Simple control: receiver can drive a motor/relay (pump/valve) remotely from dashboard or automation logic

---

## Features

- Real-time soil moisture monitoring
- Temperature & humidity monitoring
- Remote viewing via Blynk
- Simple local automation (turn motor on/off)
- Lightweight, low cost hardware

---

## Hardware BOM (suggested)

- ESP8266 module (NodeMCU / Wemos D1 mini) — receiver
- Arduino Pro Mini / NodeMCU / Wemos D1 mini — transmitter (or use ESP8266 for both)
- nRF24L01 transceiver modules (one per node + gateway)
- Soil moisture sensor (analogue)
- DHT11 or DHT22 temperature/humidity sensor
- PIR sensor (optional)
- Relay module / motor driver (for irrigation valve or pump)
- Battery / solar + regulator (for field node), jumper wires, breadboard, enclosure

Add exact part numbers to the BOM before procurement.

---

## Software & libraries

- Arduino IDE (or PlatformIO)
- Libraries:
  - RF24 (manages nRF24L01)
  - DHT (DHT sensor library)
  - nRF24L01 (driver header)
  - BlynkSimpleEsp8266 (if using Blynk on ESP8266)
  - ESP8266WiFi (for ESP8266 builds)

Install libraries via Library Manager or PlatformIO.

---

## How it works (data flow)

1. Node reads analogue soil sensor (averaged), DHT and PIR.
2. Node fills a small struct and transmits using nRF24L01.
3. Gateway (ESP) listens for packets and parses the struct.
4. Gateway publishes the values to Blynk virtual pins (or publishes to MQTT/influxdb).
5. Dashboard shows graphs and allows manual control of relay/motor.

Important: When sending structs over RF, both sides must be compiled with compatible packing and types. Consider a version field, sequence number and basic checksum for robustness.

---

## Pin mapping (example)

Transmitter (examples in sketches use these symbolic pins):
- soilMoisture: A0
- DHT data: D3
- PIR: D0
- nRF24L01 CE / CSN: D1 / D2 (adjust to your board)

Receiver (ESP8266):
- nRF24L01 CE / CSN: D1 / D2 (adjust)
- Motor/Relay control: D0 (example)
- Buzzer: D3

Note: Pin labels (Dx) depend on the board (NodeMCU vs Wemos). Verify the physical GPIO numbers for your board.

---

## Calibration (soil moisture)

Soil sensors vary. Calibrate each sensor node:

1. Dry calibration:
   - Leave the sensor probe in open air (dry) and record analog value (ADC). Save as DRY.
2. Wet calibration:
   - Insert probe in water (or thoroughly wet soil), record analog value. Save as WET.
3. Use these values in code:
   - Percent = (raw - WET) / (DRY - WET) * 100
   - Clamp percent to 0–100%
4. Update constants in the transmitter sketch (MOISTURE_WET / MOISTURE_DRY).

This gives more meaningful percent values than a fixed magic mapping.

---

## Setup & build instructions

1. Clone the repo:
```bash
git clone https://github.com/Aman-Ptl/Smart_Soil_Monitoring_System.git
cd Smart_Soil_Monitoring_System
```

2. Open the provided .ino sketches in Arduino IDE:
- `mmmut_soil_Tx.ino` (sensor node)
- `projectsoilmmmut_rx.ino` (receiver / gateway)

3. Install Arduino libraries:
- RF24
- DHT sensor library
- Blynk (if using Blynk)
- ESP8266 core for Arduino (for NodeMCU / Wemos)

4. Update configuration placeholders:
- In receiver sketch, replace:
```c
char auth[] = "YOUR_BLYNK_TOKEN";
char wifi[] = "YOUR_WIFI_SSID";
char wifipass[] = "YOUR_WIFI_PASSWORD";
```
- In transmitter, set MOISTURE_WET / MOISTURE_DRY to your calibration values.

Important: DO NOT commit real credentials to the repo. See "Security & best practices".

5. Select board and port and upload:
- Upload the transmitter to the node microcontroller.
- Upload the receiver to the ESP8266.

6. Power devices and open Serial Monitor (9600 baud) for debugging.

---

## Blynk dashboard setup (quick)

1. Create a Blynk account and create a new Template or Blank Project.
2. Get your Blynk auth token (or Template ID + Device Name if using new templates).
3. Add widgets mapped to virtual pins:
   - V1: Soil moisture (Value display / Gauge)
   - V2: Temperature (C)
   - V3: Humidity
   - V4: PIR (LED or value)
   - V5: Temperature Fahrenheit (optional)
   - V0: Button to control relay/motor (switch)
4. Use the token in the receiver sketch. Restart the ESP and ensure Blynk connects.

If you prefer open-source alternatives, use MQTT to publish to a broker and visualize with Node-RED / Grafana.

---

## Security & best practices

- DO NOT commit secrets (Wi‑Fi credentials, Blynk tokens, API keys) into version control.
- If you accidentally committed secrets:
  - Revoke/regenerate the secret (e.g., Blynk token, Wi‑Fi password if necessary).
  - Remove secrets from git history:
    - Use BFG Repo Cleaner (recommended) or git filter-branch:
      - BFG: https://rtyley.github.io/bfg-repo-cleaner/
    - Example (BFG):
```bash
# remove a file called secrets.txt from history
bfg --delete-files secrets.txt
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```
- Better pattern: keep secrets in a local `secrets.h` or `config.h` that is gitignored. Example `.gitignore`:
```
/secrets.h
```
- In code use placeholders and load real secrets from the local file not tracked in git.

---

## Troubleshooting

- No data arriving on receiver:
  - Check nRF24L01 power (3.3V regulator) and wiring; nRF modules are sensitive to power.
  - Verify CE and CSN pins match in both sketches.
  - Ensure both use same RF channel and data rate.
- Strange / corrupted values:
  - Struct packing mismatch. Try building both sketches for same architecture or use a compact byte payload.
  - Add sequence number and checksum to the packet.
- DHT returns NaN:
  - DHT sensors are timing-sensitive. Check wiring and pull-up resistor if needed.
  - DHT11 is slower and lower precision than DHT22.
- Blynk not connecting:
  - Validate token and Wi‑Fi credentials.
  - Check if router requires captive portal or extra login.
- ADC reads not changing:
  - Check soil sensor wiring and voltage reference. Some boards map A0 differently.

---

## Coding notes & recommendations

- Avoid blocking delays in the transmitter and receiver. Use non-blocking timers (millis()) to keep radio responsive.
- Add packet fields:
  - uint8_t version
  - uint32_t seq_number
  - uint16_t crc/checksum
- Consider using JSON or small TLV payload if you move to MQTT / TCP.
- For battery powered nodes, implement deep sleep between readings and add battery voltage measurement.
- Add OTA updates (ESP8266 OTA / PlatformIO) for remote firmware upgrades.

---

## Roadmap / Suggested features

- Reliability: add ack/retry, sequence numbers, checksum
- Cloud: MQTT publishing to broker, persistent storage (InfluxDB), dashboards (Grafana)
- Power: battery meter, deep-sleep, solar charging support
- Scale: multi-node addressing and device registry
- Security: TLS for MQTT, authentication, encrypted payloads
- Sensors: pH probe, EC (electrical conductivity) for nutrient monitoring
- Control: automated irrigation with safety interlocks and scheduling
- UI: Mobile web app or native app, multi-user permissions

---

## Contributing

Contributions welcome. Suggested workflow:
1. Fork repository
2. Create a branch (feature/bugfix)
3. Make changes, avoid committing secrets
4. Open a Pull Request with description and testing notes

Please include:
- Hardware used for testing (board model)
- Calibration values
- Any Blynk or dashboard configuration (redacted)

---

## License

Choose a license for your project. If you want a permissive license, MIT is a common choice. Example badge and short note:

This project is available under the MIT License — see LICENSE file for details.

---

## A few final notes

- I removed any hard-coded credentials from the suggested examples here. If your repo currently contains a Blynk token or Wi‑Fi credentials, revoke them now in the provider console and replace them in code using a local `secrets.h` that is listed in `.gitignore`.
- If you'd like, I can:
  - Create an improved README PR directly against this repo.
  - Prepare example `secrets.h` and `.gitignore`.
  - Create improved, non-blocking sketches with sequence numbers and a small checksum for reliability.
  - Add an example MQTT collector (Node-RED + InfluxDB + Grafana) and a step-by-step deployment guide.

If you want me to add or change any section (for example a wiring image, Fritzing diagram, or exact BOM with links), tell me what you prefer and I will update the README and prepare a PR.  
