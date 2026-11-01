# Agile Firmware Changelog

Format: `vMAJOR.MINOR.PATCH - YYYY-MM-DD`. Tag each release in git as `fw-vX.Y.Z`.

## v0.19.0 - unreleased (not compiled into a release, not run on hardware)
- Unit orientation NORMAL / INVERTED, saved in NVS `lt_imu` key `orient`. It sets the display rotation (3 / 1) and the IMU signs together: INVERTED negates left, up and yaw rate (enclosure rolled 180° about forward).
- Level Now learns it. If gravity reads below −0.5 g on the vertical axis, the unit is installed the other way up: the trim is computed in the flipped frame, then orientation and trim are saved and the unit reboots. Only with no lap in progress and at 5 km/h or less; otherwise the toast says `ORIENT FLIP / BLOCKED - STOP CAR` and nothing is saved.
- No automatic flip at boot. The boot check shows `IMU FLIPPED? HOLD OK 10s TO RELEARN` instead.
- `GET /imu` adds `orient`, `flip` and `level`. New `POST /imu/orient` sets it by hand (bench fallback). The web IMU card shows and sets the orientation.
- v0.18.0 stays the stable release with the orientation fixed.

## v0.18.0 - unreleased (provisional, not run on hardware)
- TFT backlight brightness 10–100 % from the web Config tab: slider with live preview, Day/Dusk/Night presets, saved to NVS `lt_disp`, applied before the TFT starts. Endpoints `GET /brightness`, `POST /brightness`.
- Needs the v1.1 rework: TFT LED pin cut from GND and switched low-side by an MGSF1N03LT1G, gate on GPIO48 via 100 Ω with 10 kΩ to 3V3. The mod is detected at boot; a stock board reports "not fitted". See `Documentation/TechnicalSpec.md` §7.4.1.

## v0.17.x
- Current series as consumed by PWSM (`lib/adapters/pltd.js`). Import source and history here.
