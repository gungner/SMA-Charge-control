# Changelog

All notable changes to this project are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/).

## [2.0.0] - 2026-11-07

New flow: **SMA Control v2** (`SMA-Control-v2.json`). It replaces the 1.0 Beta flow and must **not** run at the same time as it.

### Added
- **Peak shaving, always on.**
  - Phase-current based (register 31435, for a 20 A fuse, triggers at 19 A).
  - Power based via the new MQTT topic `sma/peak` (W, default reserve 13 kW).
  - Both use integrating controllers. Power peak does not start below 5 % SoC (resumes at 7 %).
- **Manual discharge** (`DISCHARGE_MANUAL`) via the new MQTT topic `sma/discharge`: target export to the grid in W, with the battery covering the house load plus the export. Only applies while SoC is above the `sma/soc` target.
- New states `PEAK_SHAVING`, `DISCHARGE_MANUAL` and `FAILSAFE` (also broadcast on `sma/state`).
- **Fail-safe.** All inputs are time-stamped; old or missing data (older than 10 s) puts the inverter in a safe state (803, 100 %, 0 W).
- **Phase currents published to MQTT** on `sma/fas/l1`, `sma/fas/l2` and `sma/fas/l3` (A).
- Minimum dwell time (20 s) between modes, a configurable SoC hysteresis for CHARGE (`SOC_HYST`, off by default) and slower AUTO/LOCKED switching at SoC = target (fewer mode changes on cloudy days).
- AC limits: charge max 8000 W + PV, discharge max 8000 W - PV.

### Changed
- House power is calculated from the inverter AC power (30775) plus net grid power.
- Real battery power (31393/31395) is used instead of an estimate from current and SoC.
- Writes to the inverter only happen on change. Forced modes refresh every 2 s / 10 s, AUTO every 60 s.
- Input polling is split per register with separate intervals (1 s and 5 s).
- Peak shaving from the 1.0 roadmap is now implemented.

### Removed
- Shelly EM3 dependency. House load is now calculated from the inverter and the SMA Home Manager 2; no external energy meter or `shellies/...` MQTT topics are needed.

## [1.0 Beta] - initial public version

- State machine with `AUTO`, `CHARGE`, `ZERO_EXPORT` and `LOCKED` modes.
- Modbus TCP control of an SMA Tripower and BYD battery, MQTT control (`sma/soc`, `sma/zeroexport`, `sma/chargepower`) and state broadcast (`sma/state`).
- Household load from a Shelly EM3 over MQTT.
