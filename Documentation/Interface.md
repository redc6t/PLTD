# Agile Pit Wall Interface

Taken from the firmware source (v0.17.0, `setupWebServer()`, `onWsEvent()`, `wsPushLive()`, `buildDataJson()`). The firmware is the source of truth: update this file when the API changes. PWSM consumes it through `PWSM_Pit_Wall_Stategy_Manager/lib/adapters/pltd.js`.

Agile is the server. Clients connect to `ws://<ip>/ws` (port 80) and poll HTTP on the same host. Station mode only; DHCP by default (`WIFI_USE_STATIC_IP 0`), trackside static address `192.168.0.41` when enabled.

## WebSocket, device to client

| `t` | Rate | Fields |
|---|---|---|
| `live` | 10 Hz (only while a client is connected) | `d` live delta ms, `v` delta valid, `tl` timing live (false in pits / stationary), `bs` bar state (0 OK, 1 NOREF, 2 GRAY, 3 OFF), `sec` current sector, `lap`, `el` lap elapsed ms, `est` predicted lap ms, `spd` km/h, `cs` car state string, `pit` lap touched the pits, `br` bar full scale ms |
| `flagstate` | on change, and in reply to `pitstate` | `yellow`, `sc`, `red` (bool) |
| `pitshown` | when a `pitmsg` is accepted | `id` |
| `pitack` | when a request message expires | `id`, `by` (always `"timeout"`; the driver ack was removed in v0.12.0) |

Car states (`cs`, `/data car_state`): `RUNNING`, `PIT ENTRY`, `PITS`, `PIT EXIT`, `FUELLING`. PWSM treats `PIT ENTRY`, `PITS`, `FUELLING` as in-pit.

## WebSocket, client to device

Single-frame text messages only.

| `t` | Fields | Notes |
|---|---|---|
| `pitmsg` | `text`, `pri` (0 or 1) | Shown white on blue, expires after 5 s. `text` is stored in 16 bytes, so 15 characters. `pri` above 1 is clamped to 1. A message with the same text replaces the old one. |
| `pitflag` | `name`, `on` (bool) | Latched, priority 2, cleared only by `on:false`. Names used by the UI: `YELLOW FLAG` (black on yellow), `SAFETY CAR` (black on orange), `RED FLAG` (white on red). Any other name is accepted and drawn like yellow. |
| `pitstate` | – | Device broadcasts `flagstate`. |

The queue holds 4 messages. When it is full the oldest unlatched message is replaced; if all 4 are latched, a new message is dropped.

## HTTP

### Read

| Endpoint | Returns |
|---|---|
| `GET /` | The web app (HTML + JS) |
| `GET /data` | Full state, see below |
| `GET /refpath` | `[{lat,lon},…]` reference lap outline, decimated to ~800 points; `[]` before a reference exists |
| `GET /geozones` | `{zones:[{type,label,colour,vertices:[{lat,lon}]}]}` armed zones |
| `GET /sf/get` | `{valid,lat,lon}` legacy point S/F (midpoint of the S/F line when a circuit is armed) |
| `GET /pitstate` | `{"t":"flagstate",yellow,sc,red}` |
| `GET /geo/list` | `{"<circuit>":[zones…],…}` every stored circuit |
| `GET /wifi` | `{ssid,connected,ip,ip_mode,rssi,from_nvs}` |
| `GET /imu` | `{roll,pitch,yaw,busy,mount,orient,flip,level,suggest,conf,n}` degrees. v0.19.0: `orient` is `NORMAL` or `INVERTED`; `flip` is true while an orientation reboot is pending; `level` is the last Level Now outcome (0 none or running, 1 ok, 2 failed, 3 flipped and rebooting, 4 flip blocked) |
| `GET /thr` | `{brake,corner,accel,mindur}` |
| `GET /brightness` | `{available,level,min,pin}`. v0.18.0. `available` is false on a board without the backlight mod. `?level=N` sets N (clamped 10–100) and saves it |
| `GET /logs` | `{cur,full,used_pages,total_pages,sessions:[{id,bytes}]}` |
| `GET /log?id=N` | `LOG_nnnn.CSV` as a streamed `text/csv` attachment |
| `GET /update`, `GET /ota` | OTA status: `{version,fw,user,auth,sketch_size,free_sketch,free_space,running,running_addr,running_size,target,target_size,ota_capable,ready,why,active,pct}` (no-store) |

### Write

All JSON bodies; replies are `{"ok":true}` or `{"ok":false,"error":…}`.

| Endpoint | Body | Effect |
|---|---|---|
| `POST /geo/save` | `{name,zones:[…]}` | Store a circuit (name without commas, ≤ ~3.9 KB) |
| `POST /geo/load` | `{name}` | Arm the circuit (zones + S/F line), mark it active for boot |
| `POST /geo/delete` | `{name}` | Delete a circuit |
| `POST /sf/set` | `{lat,lon}` | Legacy point S/F; also erases the stored sector map |
| `POST /nuke` | `{confirm:"ERASE",sf:bool}` | Learned-state reset |
| `POST /thr` | `{brake,corner,accel,mindur}` | Thresholds (0.05–2.0 g, 50–3000 ms) |
| `POST /brightness` | `{level,save}` | v0.18.0. Backlight 10–100 % (clamped). `save:false` is a live preview with no NVS write; `save` defaults to true. Replies with the `GET /brightness` JSON, or 400 `{"ok":false,"error":"parse"}` |
| `POST /imu/level` | – | Capture roll/pitch from gravity (409 if busy). v0.19.0: if the unit reads upside down it flips the orientation, saves the trim and reboots (only in `LAP_IDLE` and at ≤ 5 km/h) |
| `POST /imu/orient` | `{orient:"normal"\|"inverted"}` | v0.19.0. Set the orientation by hand and reboot. Keeps the trim; run Level Now afterwards. 409 `lap in progress`, `vehicle moving` or `rebooting` |
| `POST /imu/yaw` | `{yaw}` | Yaw offset, −180…180° |
| `POST /wifi` | `{ssid,pass}` | Save credentials to NVS and reconnect |
| `POST /wifi/clear` | – | Revert to compiled credentials and reconnect |
| `POST /update` | multipart, field `update` | OTA flash. Basic auth; refused unless `LAP_IDLE` and ≤ 5 km/h |
| `POST /reboot` | – | Basic auth; commits the NAND page, then reboots |

### `/data` fields

| Group | Fields |
|---|---|
| Lap | `lap_number`, `lap_elapsed_ms`, `lap_state` (0 IDLE, 1 RECON, 2 RACING), `best_lap_ms`, `estimated_lap_ms`, `ref_lap_ms`, `ref_ready`, `lap_pitted`, `timing_live` |
| Delta | `live_delta_ms`, `sector_live_delta_ms`, `live_delta_valid`, `bar_state`, `bar_range_ms`, `cumulative_delta_ms` |
| Sectors | `sector_count`, `current_sector`, `sectors[]{completed,delta_ms,cur_time_ms,colour}` |
| Completed lap (latched at the line) | `last_lap{n,ms,delta,promoted,sc}`, `last_sectors[]{t,d}` |
| Position | `lat`, `lon`, `speed_kph`, `car_state` |
| GNSS | `gnss_sats`, `gnss_pdop`, `gnss_hz`, `gnss{baud,req,act,nacks,auto,epochs}`, `gnss_meas_hz`, `gnss_worst_gap_ms` |
| Circuit | `geo_active`, `geo_zones`, `sf_line`, `sf_len`, `sf_pts` |
| Recon | `style`, `style_lat`, `gate_g`, `synth{cand,gate,dgate,dsf,clust,dcap,ddur,lap_m,sp_m,sp_ms,final,fb}` |
| IMU | `level_stale`, `yaw_agreed` |
| System | `log_dropped`, `wifi_rssi`, `heap_free`, `lap_us_worst`, `disp_us_worst` |

Sector colour: 0 gray (within ±10 ms or not measured), 1 green (faster), 2 red (slower), 3 purple (reserved; not produced by v0.17.0).

The completed lap time is not in the `live` frame. Clients fetch `/data` on every lap-number change and read `last_lap`.

## CAN (compiled out)

`CAN_ENABLED 0` in v0.17.0. When enabled: 500 kbit/s on GPIO1/2, `0x600` live (100 ms), `0x601` sector event, `0x602` status (1 s). See [Pseudocode §2](Pseudocode.md#2-freertos-task-map).
