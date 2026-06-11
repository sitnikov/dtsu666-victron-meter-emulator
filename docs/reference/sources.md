# Sources

Links the documentation is based on (collected: 2026-05-30). Grouped by topic.
The confidence level of individual facts is marked in the reference docs themselves (✅ / ⚠️).

## Primary sources reviewed line by line (adversarial verification)

These files were opened and read line by line during verification — the confirmed (✅) facts
rely on them:

- wlcrs/huawei-solar-lib `register_values.py` (enum `ActivePowerControlMode`:
  0/1/5/6/7) — https://raw.githubusercontent.com/wlcrs/huawei-solar-lib/master/src/huawei_solar/register_values.py
- wlcrs/huawei-solar-lib `registers.py` (40125 %/gain10; 40126 W/gain1; 47415 mode;
  47416 W/gain1; 47418 %/gain10; 37100–37138 meter block) —
  https://raw.githubusercontent.com/wlcrs/huawei-solar-lib/master/src/huawei_solar/registers.py
- salakrzy `main/DTSU666H_translator.cpp` (`#define HUAWEI_ID 0x0B`; `SERIAL_8N2`;
  FC03-only; handler `0x07D1 → 0x3B11`; **Huawei side: data block `0x0836` (80 reg.)
  + `0x08A6` (10 reg.)**; CHINT side 0x2000/0x401E; divisor table; captured real SUN2000
  frames in the comments) —
  https://github.com/salakrzy/DTSU666_CHINT_to_HUAWEI_translator/blob/main/main/DTSU666H_translator.cpp
- **Vuko791/sun2000-dtsu666h-sniffer** `flows-dtsu666h.json` — Node-RED decoder of LIVE
  inverter↔meter traffic; per-register layout of block 0x0836 (2102…2174), ground
  truth — https://github.com/Vuko791/sun2000-dtsu666h-sniffer
- **pazzero/esphome-dtsu666h-modbus-proxy** `sensors/em1_sensors.yaml` — a working ESPHome
  proxy; confirms the handshake (2001→15121=0x3B11), the data block and parameters 2214–2223 —
  https://github.com/pazzero/esphome-dtsu666h-modbus-proxy
- victronenergy/dbus_modbustcp `attributes.csv` (grid: 2600/2601/2602 int16 W power;
  2638/2640/2642 uint32 kWh Energy/Forward — NOT power; no total power in the grid service) —
  https://raw.githubusercontent.com/victronenergy/dbus_modbustcp/master/attributes.csv
- ESPHome `modbus_server` docs (value_type FP32/FP32_R; arbitrary addresses; FC03; read_lambda) —
  https://esphome.io/components/modbus_server/

## Official Huawei manuals (pinout, fail-safe, comms)

- SUN2000-(8KTL-20KTL)-M2 User Manual, Issue 12 — Table 5-3 COM pinout
  (485A2=pin7, 485B2=pin9, DIN1=pin8); §7.2.1 Energy Control (export-limitation modes);
  "Active power output limit for fail-safe" + "Communication disconnection fail-safe"
  (behavior on meter loss) — https://support.huawei.com/enterprise/en/doc/EDOC1100150830
  (Energy Control: …/EDOC1100150830/67e950f5/energy-control)
- DTSU666-H Smart Power Sensor Quick Guide, Issue 03 (official, solar.huawei.com) — 9600 8N1,
  address 11 (11–19), password 701, 3P4W (n.34) / 3P3W (n.33) —
  https://solar.huawei.com/admin/asset/v1/pro/view/425d013bf46d465db0255cc62db78e65.pdf
- DTSU666-H User Manual (official, Modbus-RTU; does NOT publish the register map) —
  https://support.huawei.com/enterprise/en/doc/EDOC1100020898/f00615c5/functions

> ⚠️ Huawei does NOT officially publish the DTSU666-H Modbus map — the addresses
> 0x07D1/0x0836/0x08A6 were established by reverse engineering (salakrzy + Vuko791 + pazzero),
> but they agree across the three sources and match live-traffic captures. Confirm the exact
> handshake code (0x3B11/0x3F80) and stop bits on your own hardware.

> ⚠️ The official Huawei "Solar Inverter Modbus Interface Definitions" PDF is behind a
> login/anti-scrape wall; the inverter addresses are confirmed via the wlcrs library (built
> from that PDF), but the **exact DTSU666 hex addresses and the 47416 scale (W vs kW) remain
> "verify on-device".** PDF mirror (unofficial):
> https://forum.iobroker.net/assets/uploads/files/1732790783983-sun2000ma-v100r001c00spc166-modbus-interface-definitions.pdf

## Huawei: DTSU666-H meter and emulation

- salakrzy/DTSU666_CHINT_to_HUAWEI_translator — https://github.com/salakrzy/DTSU666_CHINT_to_HUAWEI_translator
- MichielfromNL/DTSU666Emulator — https://github.com/MichielfromNL/DTSU666Emulator
- ArminJo/Arduino-DTSU666H_PowerMeter — https://github.com/ArminJo/Arduino-DTSU666H_PowerMeter
- emavap/dtsu666-emulator — https://github.com/emavap/dtsu666-emulator
- emavap/dtsu666-fe — https://github.com/emavap/dtsu666-fe
- xyphro/Sun2000MeterTransposer — https://github.com/xyphro/Sun2000MeterTransposer
- Egyras/Sungrow_DTSU666_emulator — https://github.com/Egyras/Sungrow_DTSU666_emulator
- ChrisSiedler/dtsu666-Emulator, issue #2 (Huawei) — https://github.com/ChrisSiedler/dtsu666-Emulator/issues/2
- Etyop/DTSU666-H-Arduino-Reader — https://github.com/Etyop/DTSU666-H-Arduino-Reader
- Chint float register table (aggsoft) — https://www.aggsoft.com/modbus-data-logging/chint-instrument.htm
- DTSU666-H comms defaults (Huawei docs) — https://support.huawei.com/enterprise/en/doc/EDOC1100020898/f00615c5/functions
- Sign on export (photovoltaikforum 227877) — https://www.photovoltaikforum.com/thread/227877-dtsu666-h-smartmeter-negative-werte-pa-pb-und-pc/
- Tasmota DDSU666-H/ESP32 (discussion 16715) — https://github.com/arendst/Tasmota/discussions/16715

## Huawei: inverter registers / control

- wlcrs/huawei-solar-lib (registers.py, mode enum) — https://github.com/wlcrs/huawei-solar-lib
- wlcrs/huawei_solar wiki "Connecting to the inverter" — https://github.com/wlcrs/huawei_solar/wiki/Connecting-to-the-inverter
- wlcrs/huawei_solar wiki "Changing Active Power Control" — https://github.com/wlcrs/huawei_solar/wiki/Changing-Active-Power-Control
- wlcrs/huawei_solar discussion #1231 (SDongle forwarding) — https://github.com/wlcrs/huawei_solar/discussions/1231
- EVCC discussion #19715 (§9 EEG export limit) — https://github.com/evcc-io/evcc/discussions/19715
- EVCC docs Huawei SUN2000 — https://docs.evcc.io/en/meters/huawei-sun2000-hybrid-inverter/
- photovoltaikforum 244275 (active power via Modbus/TCP) — https://www.photovoltaikforum.com/thread/244275-huawei-sun2000-begrenzung-der-wirkleistung-per-modbus-tcp-kein-dongle-wie/
- Huawei "Setting Export Limitation Parameters" — https://support.huawei.com/enterprise/en/doc/EDOC1100083281/67bed395/setting-export-limitation-parameters
- TapHome Huawei SUN2000 RTU (sign of 37113) — https://taphome.com/en/compatibility/huawei-sun2000-rtu/
- Huawei Modbus Interface Definitions V3.0 (mirror) — https://community.symcon.de/uploads/short-url/pqZXWOienoBK2AsEzGD2oH1bcPR.pdf

## Victron

- dbus_modbustcp / CCGX-Modbus-TCP-register-list — https://github.com/victronenergy/dbus_modbustcp
- GX Modbus-TCP FAQ — https://www.victronenergy.com/live/ccgx:modbustcp_faq
- dbus-flashmq (MQTT, keepalive) — https://github.com/victronenergy/dbus-flashmq
- Venus OS Large (Node-RED) — https://www.victronenergy.com/live/venus-os:large
- node-red-contrib-victron, node list — https://github.com/victronenergy/node-red-contrib-victron/wiki/Available-nodes
- ESS Mode 2 and 3 — https://www.victronenergy.com/live/ess:ess_mode_2_and_3
- Dynamic ESS (Node-RED) — https://flows.nodered.org/node/victron-dynamic-ess

## ESPHome: Modbus server (slave)

- ESPHome modbus_server (official) — https://esphome.io/components/modbus_server/
- ESPHome modbus_controller (master, for reference) — https://esphome.io/components/modbus_controller/
- epiclabs-uc/esphome-modbus-server — https://github.com/epiclabs-uc/esphome-modbus-server
- thomase1234/esphome-fake-xemex-csmb (slave example) — https://github.com/thomase1234/esphome-fake-xemex-csmb
- MSkjel/esphome-flexit-modbus-server (slave example) — https://github.com/MSkjel/esphome-flexit-modbus-server
- jesusrop/esphome_huawei_sun2000 (reading Huawei over RTU) — https://github.com/jesusrop/esphome_huawei_sun2000
- ESPHome feature-request "Modbus slave" #708 — https://github.com/esphome/feature-requests/issues/708
