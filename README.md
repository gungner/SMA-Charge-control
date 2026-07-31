# SMA Inverter &amp; Battery Control (Node-RED Flow)

An advanced Node-RED automation flow designed to intelligently monitor and control an **SMA Tripower inverter** and a **BYD battery storage system** using **Modbus TCP** and **MQTT**.

> **Inspiration:** This project was inspired by and built upon concepts from [condacum/SMA-Modbus-energy-management-system](https://github.com/condacum/SMA-Modbus-energy-management-system).

I've removed my system from SMA portal to secure no interference from them. I've also blocket all attempt from the inverter, and Home Manager 2, to call home. If you remove your system from SMA portal be aware that you will lose all your data and you'll not receive any alarms generated from the inverter, this includes alarm of residual current! Make sure you're following your local regulations. In my case I pick up these alarms with Home Assistant.

I've only tested this with SMA Tripower Smart Engergy 8 SE and BYD HVS batteries. Please read the MIT license prior to use this code!

## Features

- **Intelligent State Machine:** Automatically manages operational modes based on solar generation, household consumption, and target State of Charge (SoC). In my case all decision making is done in Home Assistant, this code just carries out the command.
- **Real-Time Energy Monitoring:** Tracks PV generation (Strings A &amp; B), battery power/SoC, and grid import/export via SMA Modbus registers.
- **Household Load Integration:** Reads real-time power data from a **Shelly EM3** energy meter over MQTT with built-in rate-limiting to protect against excessive writes.
- **Dynamic Operational Modes:**
  - `AUTO`**:** Balanced operation optimizing self-consumption and battery usage.
  - `CHARGE`**:** Forces battery charging up to a user-defined power limit.
  - `ZERO_EXPORT`**:** Restricts power export to the grid depending on household load requirements.
  - `LOCKED`**:** Holds the system steady.
- **MQTT Broadcasting:** Publishes state changes and metrics to an MQTT broker for seamless integration into home automation platforms like Home Assistant.

## External Sensor &amp; Hardware Requirements

To run this flow effectively, your setup requires the following hardware and external sensors:

- **SMA Solar Inverter:** An SMA Tripower inverter with Modbus TCP enabled (default Unit ID `3`) to read PV generation, battery status, and grid interactions. You also need a SMA Home Manager 2 to manage the Auto mode.
- **BYD Battery Storage System:** Connected to the SMA inverter for energy storage.
- **External Energy Meter (Shelly EM3):** A 3-phase power meter installed at the main household feed, publishing real-time phase power data (`shellies/shellyem3/emeter/#`) via MQTT to track total household load.
- **A 247 server with Node-Red installed**. I run it on a Debian LXC container operating on a Proxmox server. 

## MQTT Commands &amp; Topics

You can control and configure the system remotely by publishing messages to the following MQTT topics:

- `sma/soc`
  - *Function:* Sets the target State of Charge (SoC) percentage for the battery (default: `50`). If target SoC is > current Soc, charging will start with chargecurrent power. If target Soc < current SoC, the inverter will use batteri as energy and go into AUTO state. If target SoC === current Soc, the system goes into LOCKED state and only use grid power.
- `sma/zeroexport`
  - *Function:* Toggles zero-export behavior. Send `1` to enable zero-export mode or `0` to disable it. If enabled the export of energy and charging of batteri will end and only grid power will be used. This is how the inverter is programmed. I only use this when the price is negativ and my batteries are at 100% SoC.
- `sma/chargepower`
  - *Function:* Sets the maximum allowed battery charge power limit in kW. Although the inverter can handle 10kW charge power, I've limited it to maximum 9kW not the stress the electronics too much.
- `sma/state` *(Read/Broadcast)*
  - *Function:* Broadcasts the current active operational state (`AUTO`, `CHARGE`, `ZERO_EXPORT`, `LOCKED`).

## Prerequisites &amp; Dependencies

- **Node-RED** with the following plugin installed:
  - `node-red-contrib-modbus`
- An active **MQTT Broker** (e.g., Eclipse Mosquitto).

## Setup &amp; Configuration

1. **Modbus Client:** Update the `SMA tripower` Modbus configuration node with your inverter's local IP address and port (`502`).
2. **MQTT Broker:** Update the `MQTT` broker configuration node with your local broker's IP address and port (`1883`).
3. **Import:** Copy the raw JSON flow data and import it directly into your Node-RED workspace via **Import -&gt; Clipboard**.
4. Change the IP addresses to match your inverter and MQTT broker.



**1.0 Beta1**

This release has been running now for some months without any issues.

**Roadmap**

- Peak Shaving, be able to to set a maximum power limit from the grid, all additional power shall be provided by the battery. 

<img width="722" height="849" alt="image" src="https://github.com/user-attachments/assets/8ff7a6b0-1d16-4bc6-90dd-388925883fdc" />



&nbsp;
