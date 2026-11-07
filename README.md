# SMA Inverter &amp; Battery Control (Node-RED Flow)

An advanced Node-RED automation flow designed to intelligently monitor and control an **SMA Tripower inverter** and a **BYD battery storage system** using **Modbus TCP** and **MQTT**.

> **Inspiration:** This project was inspired by and built upon concepts from [condacum/SMA-Modbus-energy-management-system](https://github.com/condacum/SMA-Modbus-energy-management-system).

I've removed my system from SMA portal to secure no interference from them. I've also blocket all attempt from the inverter, and Home Manager 2, to call home. If you remove your system from SMA portal be aware that you will lose all your data and you'll not receive any alarms generated from the inverter, this includes alarm of residual current! Make sure you're following your local regulations. In my case I pick up these alarms with Home Assistant.

I've only tested this with SMA Tripower Smart Engergy 8 SE and BYD HVS batteries. Please read the MIT license prior to use this code!

## Features

- **Intelligent State Machine:** Automatically manages operational modes based on solar generation, household consumption, and target State of Charge (SoC). In my case all decision making is done in Home Assistant, this code just carries out the command.
- **Real-Time Energy Monitoring:** Tracks PV generation (Strings A &amp; B), battery power/SoC, and grid import/export via SMA Modbus registers.
- **Household Load Calculation:** House consumption is calculated from the inverter AC power plus the net grid power from the SMA Home Manager 2. No external energy meter is needed.
- **Peak Shaving:** Always active. The battery covers load above a power limit (`sma/peak`) and keeps each phase below the main fuse (20 A).
- **Fail-Safe:** All measurements are time-stamped. If a mandatory value is missing or older than 10 s, the inverter is handed back to its own control.
- **Dynamic Operational Modes:**
  - `AUTO`**:** Balanced operation optimizing self-consumption and battery usage.
  - `CHARGE`**:** Forces battery charging up to a user-defined power limit.
  - `ZERO_EXPORT`**:** Restricts power export to the grid depending on household load requirements.
  - `LOCKED`**:** Holds the system steady.
  - `PEAK_SHAVING`**:** The battery discharges to keep grid import below the peak limit (`sma/peak`, default 13 kW) and each phase below the fuse limit (19 A trigger on a 20 A fuse). Never exports to the grid. Power peak does not start below 5 % SoC; phase peak has no SoC floor. Overrides all modes except `FAILSAFE`.
  - `DISCHARGE_MANUAL`**:** The battery covers the house load and exports the power set with `sma/discharge` to the grid. Only active while SoC is above the `sma/soc` target.
  - `FAILSAFE`**:** Entered when mandatory data (SoC, grid, PV A/B, inverter AC power) is missing or stale. The inverter is returned to its own default control. Highest priority.

  Priority order: `FAILSAFE` > `PEAK_SHAVING` > `DISCHARGE_MANUAL` > `ZERO_EXPORT` > `CHARGE` > `AUTO` / `LOCKED`. A mode is held for at least 20 s before switching (except `PEAK_SHAVING` and `FAILSAFE`). At SoC = target, `LOCKED` -> `AUTO` requires 10 s of continuous solar surplus and `AUTO` -> `LOCKED` requires 60 s without surplus.
- **MQTT Broadcasting:** Publishes state changes and metrics to an MQTT broker for seamless integration into home automation platforms like Home Assistant.

## External Sensor &amp; Hardware Requirements

To run this flow effectively, your setup requires the following hardware and external sensors:

- **SMA Solar Inverter:** An SMA Tripower inverter with Modbus TCP enabled (default Unit ID `3`) to read PV generation, battery status, and grid interactions. You also need a SMA Home Manager 2: it manages the Auto mode and provides the grid power and phase currents that v2 reads via the inverter.
- **BYD Battery Storage System:** Connected to the SMA inverter for energy storage.
- **A 247 server with Node-Red installed**. I run it on a Debian LXC container operating on a Proxmox server. 

## MQTT Commands &amp; Topics

You can control and configure the system remotely by publishing messages to the following MQTT topics:

- `sma/soc`
  - *Function:* Sets the target State of Charge (SoC) percentage for the battery (default: `50`). If target SoC is > current Soc, charging will start with chargecurrent power. If target Soc < current SoC, the inverter will use batteri as energy and go into AUTO state. If target SoC === current Soc, the system goes into LOCKED state and only use grid power.
- `sma/zeroexport`
  - *Function:* Toggles zero-export behavior. Send `1` to enable zero-export mode or `0` to disable it. If enabled the export of energy and charging of batteri will end and only grid power will be used. This is how the inverter is programmed. I only use this when the price is negativ and my batteries are at 100% SoC.
- `sma/chargepower`
  - *Function:* Sets the maximum allowed battery charge power limit in kW. Although the inverter can handle 10kW charge power, I've limited it to maximum 9kW not the stress the electronics too much.
- `sma/discharge` *(v2)*
  - *Function:* Target export to the grid in W (values below 2500 W = off, maximum 8000 W). The battery covers the house load plus this export (limited by the inverter AC output). Only applies while SoC is above the `sma/soc` target; otherwise it is ignored. Send `0` to stop.
- `sma/peak` *(v2)*
  - *Function:* Power peak limit for grid import in W. `0` or unset uses the default reserve of 13 kW. The battery covers all load above the limit. Does not start below 5 % SoC.
- `sma/fas/l1`, `sma/fas/l2`, `sma/fas/l3` *(v2, Broadcast)*
  - *Function:* Phase currents in A, published every second.
- `sma/state` *(Read/Broadcast)*
  - *Function:* Broadcasts the current active operational state (`AUTO`, `CHARGE`, `ZERO_EXPORT`, `LOCKED`, `PEAK_SHAVING`, `DISCHARGE_MANUAL`, `FAILSAFE`).

## Prerequisites &amp; Dependencies

- **Node-RED** with the following plugin installed:
  - `node-red-contrib-modbus`
- An active **MQTT Broker** (e.g., Eclipse Mosquitto).

## Setup &amp; Configuration

1. **Modbus Client:** Update the `SMA tripower` Modbus configuration node with your inverter's local IP address and port (`502`).
2. **MQTT Broker:** Update the `MQTT` broker configuration node with your local broker's IP address and port (`1883`).
3. **Import:** Import `SMA-Control-v2.json` into your Node-RED workspace via **Import -&gt; Clipboard** (or **Select a file to import**). `Node Red Flows 1.0 Beta.json` is kept for reference only. Never run v1 and v2 at the same time.
4. Change the IP addresses to match your inverter and MQTT broker.



## Versions

**2.0.0** (current, `SMA-Control-v2.json`)

- Peak shaving (phase current and power), manual discharge, fail-safe on stale data, phase currents on MQTT.
- See [CHANGELOG.md](CHANGELOG.md) for details.
- To go back to the old flow: disable the v2 tab, enable the old one and do a **full** deploy. Never have both active.

**1.0 Beta1** (`Node Red Flows 1.0 Beta.json`)

This release ran for some months without any issues.

**Roadmap**

- ~~Peak shaving~~ (done in 2.0.0).

<img width="722" height="849" alt="image" src="https://github.com/user-attachments/assets/8ff7a6b0-1d16-4bc6-90dd-388925883fdc" />



&nbsp;
