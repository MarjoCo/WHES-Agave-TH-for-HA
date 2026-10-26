# WHES Agave TH for Home Assistant

Home Assistant packages for the **WHES Agave TH** three-phase hybrid inverter and battery (formerly sold as **Ecos eCactus**). They read the inverter locally over Modbus, without the cloud, and can optionally control it.

There are two packages, so that reading and controlling stay clearly apart:

| Package | What it does | Writes to the inverter |
|---|---|---|
| `packages/whes_agave_th_monitor.yaml` | Readings, status, energy totals for the Energy dashboard, fault texts and fault notifications | **Never** |
| `packages/whes_agave_th_control.yaml` | Work mode, battery power (PowerRef), power limits, battery minimum SOC and the VPP watchdog | Yes, registers 60001 and 60003–60012 |

Start with the monitoring package. Add the control package only if you want Home Assistant to control the battery, and read [Control package](#control-package) first.

**Tested on my own system.** Both the monitoring and the control part have been in production use on my system since November 2025. The VPP watchdog automations were added in October 2026. My unit:

- Agave TH, model WH-TIA133: 13 kW three-phase hybrid inverter, 6 battery modules (29.9 kWh, LFP)
- Firmware: DSP 510-00004-09, EMS 511-00009-08, BMS T511-00014-10
- Register map: *WH eCactus Three Phase Inverter Modbus Protocol V1.2.2*

Other models and firmware versions may differ.

### What has and has not been tested

- **One inverter only.** The packages are built for a system with one Agave TH inverter and its battery modules. Several inverters in a parallel system (master and slaves) have not been tested. In WHES's parallel wiring diagram the units talk to each other over the same VPP-485A/B terminals that this package uses. The register map has no separate registers for parallel units, so in principle nothing should change, but I do not know what a parallel system does in practice.
- **Three-phase only.** Tested on a three-phase system. Single-phase systems have not been tested.

## What you need

- Home Assistant 2024.10 or newer (the syntax needs it). In use on my system on versions 2025.12 to 2026.9.
- An **RS485 → Ethernet converter** with a Modbus TCP gateway, wired to the inverter's **VPP-485A** and **VPP-485B** terminals (the communication terminal block, marked "Only applicable for VPP" in WHES's wiring diagram). I use a Waveshare RS485 TO ETH (B) set up as a **TCP server**, **Modbus TCP ↔ RTU**, **115200 bps, 8N1**. The inverter's Modbus slave address is `1`.
- A notification service for the fault messages, for example the Home Assistant Companion app on your phone.

The inverter's own WiFi dongle and the ECOS app keep working alongside Modbus. The dongle uses the same Modbus bus, and now and then the two collide, so a single read fails for a few seconds. That is normal, and the packages are written for it (see step 6).

## Installation

### 0. Download the files

You do not need git. Open the [latest release](https://github.com/MarjoCo/WHES-Agave-TH/releases/latest), download the **.zip** under *Assets* and unpack it on your computer.

### 1. Turn on packages in Home Assistant

Open `configuration.yaml`. If it does not already load packages, add:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

If you already have a `homeassistant:` section, add only the `packages:` line under it. Do not create a second `homeassistant:` section.

### 2. Copy the monitoring package and set your IP address

1. Create the folder `packages` next to `configuration.yaml`, if it does not exist yet.
2. Copy `packages/whes_agave_th_monitor.yaml` into it.
3. Open the file and find the lines marked `CHANGE`:

```yaml
modbus:
  - name: agave_inverter
    type: tcp
    host: 192.168.1.100  # <-- CHANGE: IP address of your RS485-to-Ethernet converter
    port: 502 # <-- CHANGE if your converter uses another port
```

Set `host` to your converter's IP address. Give the converter a fixed address (a DHCP reservation in your router, or a static IP). Do not change `name: agave_inverter`, because the control package writes to the inverter through that name.

If your `configuration.yaml` already has a `modbus:` section for other devices, you can keep it. Home Assistant combines it with the one in the package.

### 3. Choose where notifications go

Fault messages (DSP, EMS and BMS errors) are sent with `notify.notify`, which goes to the first notification service Home Assistant finds. To send them to a specific phone, replace `notify.notify` in the package with your phone's service, for example `notify.mobile_app_my_phone`. You can find the name in **Developer tools → Actions** by typing `notify.`. The control package sends one more message, when the VPP watchdog expires.

The fault messages use the iPhone critical-alert sound, which plays even when the phone is muted. On Android they arrive as normal notifications.

### 4. Add the dashboard

1. Go to **Settings → Dashboards → Add dashboard → New dashboard from scratch**, name it (for example "Agave") and open it.
2. Top right **⋮ → Edit dashboard → ⋮ → Raw configuration editor**. Replace everything with the contents of `dashboards/whes_agave_th_dashboard.yaml` and save.

The dashboard uses only Home Assistant's own cards. The **Agave** view shows readings, energy and faults. The **Control** view needs the control package; delete that view if you only use monitoring.

### 5. Check the configuration and restart

1. **Developer tools → YAML → Check configuration.** It must say the configuration is valid.
2. **Settings → System → Restart Home Assistant.** A restart is needed the first time, because Modbus cannot be reloaded on its own.

### 6. Check that it works

After the restart, wait a minute and check in **Developer tools → States**:

- `sensor.agave_battery_soc` shows the battery's charge in %.
- `sensor.agave_grid_total_power` changes every second.
- `sensor.agave_work_mode_text` shows `Normal Mode (ECOS control)`.
- `sensor.agave_dsp_error_text`, `sensor.agave_ems_error_text` and `sensor.agave_bms_error_text` show `OK – No Errors`.

If the values are `unavailable`:

- Check the IP address and port, and that Home Assistant can reach the converter.
- Check the converter's serial settings (115200 8N1) and gateway mode (Modbus TCP ↔ RTU).
- If nothing comes through at all, swap the A and B wires.
- Look for Modbus errors in **Settings → System → Logs**.

Short read breaks of a few seconds are normal: the inverter's WiFi dongle shares the Modbus bus, and their requests sometimes collide. The sensors are written so that they keep their last value or show `unavailable` during a break, never a false 0.

## Energy dashboard

Go to **Settings → Dashboards → Energy** and add these sensors. They are the inverter's own lifetime totals, so the Energy dashboard can work out every day, month and year by itself.

| Energy dashboard | Field | Sensor |
|---|---|---|
| Electricity grid | Grid consumption | `sensor.agave_total_consumption_from_grid_energy` |
| | Return to grid | `sensor.agave_total_feed_to_grid_energy` |
| | Power (optional) | `sensor.agave_grid_total_power` — positive = import |
| Solar panels | Solar production | `sensor.agave_total_pv_energy` |
| | Power (optional) | `sensor.agave_total_pv_power` |
| Home battery storage | Energy going in to the battery | `sensor.agave_battery_total_charge_energy` |
| | Energy coming out of the battery | `sensor.agave_battery_total_discharge_energy` |
| | Power (optional) | `sensor.agave_battery_power_pcs` — positive = discharge |
| | State of charge (optional) | `sensor.agave_battery_soc` |

Notes:

- **Solar** counts only the panels connected to the Agave's own PV inputs (PV1 and PV2). If you have panels on another inverter, add that inverter's energy sensor as a second solar source.
- **Battery:** use the two sensors above, not the `… RAW` sensors. The raw registers can drop for a moment on a read error; these two never go down, so a read break never shows up as a spike in the dashboard.
- Do not add the `… Daily` sensors (for example `sensor.agave_battery_charge_daily`). They are for your own dashboards; the Energy dashboard makes its own daily totals.
- The grid totals come from the inverter's grid meter. Your electricity company's meter is the one you are billed by, so small differences from your bill are normal.

## Control package

**Read this whole section before you install the control package.** It lets Home Assistant command the inverter, and a wrong command can drain the battery or overload your main fuse.

To install it, copy `packages/whes_agave_th_control.yaml` into the same `packages` folder and restart Home Assistant. It needs the monitoring package.

### What it adds

| Entity | Register | What it does |
|---|---|---|
| `input_select.agave_set_work_mode` | 60001 | Work mode: `Normal (ECOS control)`, `Full external control`, `PV charge only`, `Discharge only` |
| `number.agave_set_battery_power` | 60004 | Battery power (PowerRef). Positive = discharge, negative = charge |
| `number.agave_set_inverter_power_limit` | 60008 | Inverter AC output power limit |
| `number.agave_pv_power_limit` | 60009 | PV power limit |
| `number.agave_max_feed_in_limit` | 60003 | Feed-in limit, % |
| `number.agave_set_battery_minimum_soc` | 60012 | Battery reserve. **Never below 10 %** |
| `input_boolean.agave_vpp_watchdog_enabled` | 60010 / 60011 | Turns on the VPP watchdog automations |
| `number.agave_set_vpp_timer`, `switch.agave_vpp_timer_enable` | 60010 / 60011 | Manual watchdog settings, normally handled by the automations |

Every setting is written only when the inverter does not already have that value.

### Work modes

- **Normal (ECOS control)** (work mode 0) is the inverter's own mode and its default. The battery then runs on the settings and working mode you have chosen in the manufacturer's **ECOS app**, for example Self-powered, Load shifting (time of use) or Backup, with the app's own settings such as reserve SOC and feed-in percentage. Home Assistant only reads in this mode: it cannot set the battery power, and the app's working mode and its settings are not visible over Modbus (register 60001 only shows 0). Every time Home Assistant hands control back, or the VPP watchdog expires, the inverter returns to this mode and to your ECOS app settings.
- **Full external control** (VPP mode 1) is the only mode in which the battery power (PowerRef) does anything. Home Assistant decides the battery power; the inverter follows it.
- **PV charge only** (VPP mode 2): the battery charges only from surplus solar and does not discharge.
- **Discharge only** (VPP mode 3): the battery only discharges to cover the house and does not charge.

Order matters: **select Full external control first, then write the battery power.** A battery power written while ECOS is in control is overwritten by ECOS, and after the switch the battery keeps ECOS's last value until the next write.

Each work mode change keeps the Modbus bus busy for about 5 seconds. Do not build automations that switch the mode every few seconds.

### ⚠️ Your main fuse

**In Full external control the inverter does not limit grid import.** If Home Assistant tells the battery to charge at full power while the house also uses a lot, the inverter draws it all from the grid. My unit can draw up to 26 A per phase from the grid, which is more than a 3×25 A main fuse. When you charge from the grid in Full external control, keeping house load plus charging power under your main fuse is your automation's job. `sensor.agave_phase_r_current`, `_s_current` and `_t_current` are read every second for this.

### VPP watchdog

**Why it exists.** In Full external control the inverter does exactly what Home Assistant last told it. If Home Assistant then crashes, restarts or loses the Modbus connection, nothing on the inverter changes back by itself: a grid charge keeps drawing from the grid, and nobody watches the fuse. The inverter has a built-in dead-man's switch for this, the VPP timer, and the manufacturer recommends always using it:

- **60010 VPPTimer** is a countdown in seconds. When it reaches 0, the inverter switches its work mode back to 0, Normal (ECOS).
- **60011 VPPTimerEnable** turns the countdown on (1) or off (0).

Home Assistant keeps rewriting the countdown while it is in control. As long as Home Assistant is alive the countdown never reaches 0. When Home Assistant stops, the countdown runs out and ECOS takes over safely.

**Turn it on:** `input_boolean.agave_vpp_watchdog_enabled`. Do this before you use Full external control. The four automations below do nothing while it is off.

**The four automations:**

1. **Arm** (`agave_vpp_watchdog_arm`). Runs when the work mode changes to Full external control, when Home Assistant starts while in Full external control, and when you turn the master switch on. It writes the timeout to 60010 first and enables the timer (60011 = 1) one second later. The order matters: if 60011 were enabled first, the countdown could start from an old value left in 60010, for example 0 or the inverter's default.
2. **Refresh** (`agave_vpp_watchdog_refresh`). Every 5 seconds while in Full external control, it writes the timeout to 60010 again. If 60011 has dropped to 0 (the timer expired, or someone turned it off by hand), it sets it back to 1. The refresh writes the register directly, not through `number.agave_set_vpp_timer`, because the number skips a write when the value does not change, and the refresh always writes the same value.
3. **Disarm** (`agave_vpp_watchdog_disarm`). When Home Assistant hands control back to Normal (ECOS), or you turn the master switch off, it disables the timer (60011 = 0), so ECOS is never under a countdown in normal use.
4. **Expired alert** (`agave_vpp_watchdog_expired_alert`). If the inverter's work mode drops from 1 to 0 by itself while Home Assistant still had Full external control selected, the countdown ran out: the refresh did not get through. You get a notification. It changes nothing; the inverter is already back in ECOS. When Home Assistant itself selects Normal, it changes its own selection before the inverter does, so a normal release never sends this alert.

**Which timeout is used.** Every arm and refresh picks the timeout from the battery power Home Assistant has asked for (`number.agave_set_battery_power`):

| Battery power (PowerRef) | Timeout | Helper |
|---|---|---|
| Below 0 (charging) | 30 s | `input_number.agave_vpp_timeout_s` |
| 0 or above (holding or discharging) | 120 s | `input_number.agave_vpp_timeout_sell_s` |

Charging gets the short timeout because charging from the grid is what can overload a fuse if Home Assistant stops in the middle of it. Holding or discharging does not increase what the house takes from the grid, so it can wait longer. When the power changes from discharging to charging, the next refresh, within 5 seconds, already writes the short timeout. Both helpers start from these values after every restart; change `initial:` in the file if you want other values.

**How the numbers fit together.** The refresh runs every 5 seconds and the shortest timeout is 30 seconds, so Home Assistant can pause for about 25 seconds without the inverter dropping out of Full external control. On my system the timer jitter in Home Assistant was 11–18 seconds at worst, which is well inside that.

**What you will see:**

- **Home Assistant restart while in Full external control:** the countdown runs out during the restart and the inverter goes back to ECOS. This is intended: after a restart, ECOS is the safe state. Your own automations take control again when they next run.
- **The inverter returning to ECOS:** with a 120 s timeout I measured the return at 118 and 119 seconds after the refresh stopped.
- **After expiry:** ECOS takes control by itself: the work mode changes to Normal (ECOS) and the battery runs on your ECOS app settings again. Home Assistant's work mode selection follows it to `Normal (ECOS control)`, and the disarm automation turns the timer off (60011 = 0). The inverter resets 60010 to its own default, 300 s; the next arm writes your timeout again.
- **Short Modbus read breaks:** a failed read does not stop the refresh. It writes every 5 seconds regardless of what was last read.

`switch.agave_vpp_timer_enable` and `number.agave_set_vpp_timer` are for setting the timer by hand, for example while testing. Leave them alone while the automations are on.

### Battery minimum SOC

`number.agave_set_battery_minimum_soc` cannot go below 10 %, the manufacturer's recommended minimum. The dashboard slider stops at 10, and the write itself is limited to 10–100 %, so an automation or a service call cannot set a lower value either.

### What is deliberately left out

Register **60002 Remote on/off** stops the whole system. It is only read here (`sensor.agave_remote_on_off_status`), never written.

## Things I learned on a real unit

- **Signs differ between registers.** Grid power 36275: positive = import. Battery power on the inverter side (PCS, 36307): positive = discharge. Battery power on the BMS side (36303): positive = charge. Getting one backwards inverts every charge/discharge decision.
- **The feed-in limit (60003) is not effective in Full external control** (protocol V1.2.2).
- **Grid-to-battery (36417) and battery-to-grid (36419)** need EMS V06 and DSP V05 or newer. On my unit 36417 rose while the battery charged from solar, so check it on yours before you rely on it.
- **The ECOS app's own modes** (Self-powered, Load shifting, Backup) are not visible over Modbus. Register 60001 only tells Normal from the three VPP modes.

## Support and maintenance

These packages are shared as is, because they work on my own system. **They are not actively maintained, and I cannot promise to answer questions or fix problems.** If something needs changing, you are welcome to fork the repository and change it yourself. The MIT license allows that.

You use the packages at your own risk. Check that the settings suit your installation, and your main fuse, before you let Home Assistant control the inverter.

## License

[MIT](LICENSE)

## Files

| File | Purpose |
|---|---|
| `packages/whes_agave_th_monitor.yaml` | Modbus connection, sensors, energy totals, fault texts and fault notifications. Read only |
| `packages/whes_agave_th_control.yaml` | Optional: work mode, battery power, limits, minimum SOC and the VPP watchdog |
| `dashboards/whes_agave_th_dashboard.yaml` | Dashboard: Agave (readings, energy, faults) and Control views |
