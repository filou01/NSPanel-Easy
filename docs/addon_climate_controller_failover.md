# Add-on: Climate Controller Failover

## Purpose

The **Climate Controller Failover** add-on is an advanced local-heating fallback for NSPanel Easy
installations where Home Assistant is normally responsible for the heating algorithm (PID, PWM, adaptive
control, schedules, window logic, or another external climate controller), while the NSPanel must remain able
to heat the room autonomously if Home Assistant, the network, or the Blueprint becomes unavailable.

Unlike the standard `nspanel_esphome_addon_climate_backup.yaml`, this add-on performs a complete **controller
ownership handover** rather than only mirroring the embedded thermostat action to a backup relay.

During normal operation Home Assistant owns the configured heating relay. The add-on continuously stores the
last valid climate mode and target temperature. If Home Assistant is no longer considered ready for longer
than the configured delay, the panel switches ownership to its embedded thermostat, restores the last known
target/mode and opens a fully local climate UI. When Home Assistant is healthy again, ownership is returned
only after all required controller values and the NSPanel Easy Blueprint are ready for a configurable
stabilization period.

> [!IMPORTANT]
> This add-on currently supports **heating only (`heat` / `off`)**. The standard Climate Backup add-on remains
> the better choice for simple heat/cool/dual relay fallback where full controller synchronization and an
> offline UI are not required.

## Why a second fallback add-on?

NSPanel Easy already includes a standard Climate Backup add-on in the 2026.10 line. It is
intentionally small and generic: after Home Assistant has been unavailable for `climate_backup_delay`, the
embedded thermostat mirrors its action to configured backup relays, and those relays are released again when
Home Assistant reconnects.

Controller Failover addresses a different use case: the same physical heating output is normally controlled by
a Home Assistant climate algorithm and must become a locally controlled thermostat only during an outage.

### Existing Climate Backup is better suited for

- Simple installations with minimal configuration.
- Heat, Cool, and Dual modes.
- Dedicated backup-relay configurations.

It runs the embedded thermostat and mirrors its action to dedicated backup
relays after the subscription-loss delay, releasing them when HA returns.
Its firmware and documentation remain unchanged by this add-on.

### Climate Controller Failover is better suited for

- External HA PID/PWM controllers, Adaptive Climate, and Smart Thermostat.
- Systems where HA normally performs the actual control algorithm.
- Retaining the last valid target and Heat/Off mode during an outage.
- Visible local climate adjustment and temperature while offline.
- A controlled, delayed return after all required HA data is ready.

Choose based on relay topology and controller requirements. Controller Failover
requires more configuration and depends on more NSPanel Easy internal hooks.

## Safety model

The central design rule is **exactly one owner of the physical heating relay**.

The add-on uses three owner states:

- `START` - neither controller may energize the relay while readiness is being established.
- `HA` - Home Assistant owns the relay. The output is allowed only when the controller climate is in `heat`,
  the heat-request entity is `on`, the API has an active state subscription, the Blueprint is ready, and all
  three HA values have been received.
- `LOCAL` - the embedded thermostat owns the relay. Home Assistant state can no longer energize it.

The sole runtime writer of the physical GPIO switch (`ccf_heater_output`) is
`ccf_apply_output`. The standard selected relay ID becomes an internal, read-only
template proxy: it reports the physical state for relay chips, while commands,
including hardware-button toggles, only re-evaluate the arbiter. They cannot
directly change the GPIO. The embedded thermostat is internal too, preventing HA
from changing its settings directly. Both ownership transfers first turn the
output off, and the physical GPIO restores Off on boot.

Other custom packages must not redefine the owned GPIO, output, or proxy.
The unselected relay retains its normal behavior. The transfer is ordered in
software; there is no relay-contact feedback or configurable physical dead-time.

All relay writes are centralized in one ESPHome script (`ccf_apply_output`). The transfer back to Home
Assistant is break-before-make: output ownership is first set to `START`, the embedded thermostat is turned
off, fresh HA state is stored, and only then is HA ownership enabled.

## Requirements

Include all of the following:

1. `nspanel_esphome.yaml`
2. `esphome/nspanel_esphome_addon_climate_heat.yaml`
3. `esphome/nspanel_esphome_addon_climate_controller_failover.yaml`

Do **not** include Climate Backup, Climate Cool, Climate Dual, or Cover at the
same time. These combinations are rejected during compilation. The Climate Heat
default `heater_relay: "0"` is invalid for this add-on: select Relay 1 or 2.

The Home Assistant controller must expose:

- a climate entity whose state is `heat` or `off` and whose `temperature` attribute contains a finite target within the local thermostat's `temp_min..temp_max`;
- a binary-like entity (`input_boolean`, `binary_sensor`, switch, etc.) that is `on` exactly when the external controller requests heat.

The HA climate's temperature unit must match the panel's `temp_units`; the HA
unit attribute is not auto-detected. Configure the Blueprint's main climate as
the external controller. Remove direct HA relay-control automations: the selected
relay entity becomes internal and HA drives the subscribed heat request instead.

The heat-request entity should represent the final output of the Home Assistant controller after
PID/PWM/window logic. Home Assistant must not also control the NSPanel relay through another automation once
this add-on is installed.

## Installation

Include the base, Climate Heat, and Controller Failover packages in that order.
Remove the earlier local fallback package. Configure the substitutions below,
compile, and install. Check the diagnostic state and exercise cold boot, outage,
local adjustment, and recovery on the actual panel before relying on it.

## Configuration

### Required substitutions

| Key | Description |
| --- | --- |
| `controller_climate_entity` | Primary Home Assistant climate entity, for example `climate.living_room_floor_heating`. |
| `controller_heat_request_entity` | Final external heating request, for example `input_boolean.living_room_heat_request`. |
| `heater_relay` | Physical heating relay: `"1"` (default) or `"2"`. Temperature units do not affect relay selection. |

### Optional substitutions

| Key | Default | Description |
| --- | ---: | --- |
| `climate_failover_boot_wait_ms` | `120000` | Minimum uptime before local takeover at boot. |
| `climate_failover_loss_wait_ms` | `30000` | How long HA may be unready during normal operation before local takeover. |
| `climate_failover_return_wait_ms` | `10000` | Continuous readiness before granting HA ownership, including at startup. |
| `climate_failover_initial_target` | `70°F` / `21°C` | Safe target used only if no stored/valid target exists. |
| `climate_failover_initial_heat` | `false` | Safe initial heat mode used only if no stored mode exists. |
| `climate_failover_local_label` | `Local heating` | Name displayed on the local climate page. |

Time values are integer milliseconds. Boot wait accepts `0..2147483647`; loss
and return waits accept `1..2147483647`. Compilation checks these ranges, relay
selection, initial target bounds, required Heat support, and conflicting add-ons.
Missing HA entity substitutions fail ESPHome configuration validation.

Normal Climate Heat settings such as `min_off_time`, `min_run_time`,
`min_idle_time`, `heat_deadband`, and `heat_overrun` apply during LOCAL control.
They do not reshape the external HA controller's heat request. Ownership changes
or invalid HA input can switch the output off immediately.

## Example YAML

```yaml
substitutions:
  device_name: "living-room-panel"
  friendly_name: "Living room NSPanel"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
  language: en

  heater_relay: "1"
  controller_climate_entity: climate.living_room_floor_heating
  controller_heat_request_entity: input_boolean.living_room_heat_request

  climate_failover_boot_wait_ms: "120000"
  climate_failover_loss_wait_ms: "30000"
  climate_failover_return_wait_ms: "15000"
  climate_failover_initial_target: "21.0"  # 70.0 for Fahrenheit panels
  climate_failover_initial_heat: "false"
  climate_failover_local_label: "Local heating"

packages:
  remote_package:
    url: https://github.com/edwardtfn/NSPanel-Easy
    ref: main
    refresh: 300s
    files:
      - nspanel_esphome.yaml
      - esphome/nspanel_esphome_addon_climate_heat.yaml
      - esphome/nspanel_esphome_addon_climate_controller_failover.yaml
```

## State machine

| Owner | Valid output source | Transition |
| --- | --- | --- |
| `START` | None; output Off | Stable readiness grants HA; otherwise boot wait permits LOCAL. |
| `HA` | HA heat request, gated by Heat mode and readiness | Continuous unreadiness for loss wait grants LOCAL. |
| `LOCAL` | Embedded thermostat, gated by finite internal temperature | Continuous readiness for return wait grants HA. |

The supervisor runs every second and on HA input updates. Invalid inputs and
subscription loss cancel the return timer. Brief recovery followed by failure
starts a new loss interval. Duration comparisons handle `millis()` rollover.

## Operation

### Normal HA operation

When the API has a state subscription, the Blueprint handshake is complete, and valid mode/target/demand
values have arrived, the panel enters `HA` ownership. The embedded thermostat is forced off. The physical
relay follows only the external heat-request entity, gated by the external climate being in `heat` mode.

While HA owns the relay, the add-on continuously updates its persistent copy of target temperature and
heat/off mode. Incoming targets and initial configuration use the panel's temperature unit.
The saved target always uses Celsius internally, so a display-unit change cannot
reinterpret persisted data. ESPHome's restoring globals use the normal preference
write interval. Sudden power loss can lose changes not yet flushed to flash;
network loss does not affect saved values in memory.

### Loss of Home Assistant or Blueprint readiness

A disconnect immediately prevents stale HA state from energizing the relay. If readiness does not recover within `climate_failover_loss_wait_ms`, the panel enters `LOCAL` mode.

The local thermostat then:

- restores the last valid HA target temperature;
- restores the last valid `heat` / `off` mode;
- uses the NSPanel internal temperature sensor;
- exclusively controls the configured physical relay;
- exposes a local Climate page that remains usable without HA/Wi-Fi;
- shows the internal temperature on Home;
- stores any locally changed target/mode for continued offline operation.

Offline changes are deliberately **not written back** to Home Assistant. On recovery, HA remains the authority.

### Return to Home Assistant

A simple TCP/API reconnect is not enough. The add-on waits until:

1. the ESPHome API has a client with state subscriptions;
2. the NSPanel Easy Blueprint is ready;
3. a valid HA climate mode has been received;
4. a finite HA target within `temp_min..temp_max`, in panel units, has been received;
5. a valid heat request has been received; and
6. this complete ready state remains continuous for `climate_failover_return_wait_ms`.

Losing the last state-subscribing connection clears the received-value mask and
Blueprint readiness. Reconnection requires replayed controller data and a new
Blueprint handshake. Logger-only clients do not satisfy the subscription check.
LOCAL keeps control throughout the return wait.

The handback then disables relay output first, turns the local thermostat off, stores the fresh HA values, and finally enables HA ownership.

## Diagnostics

The add-on creates `sensor`-style diagnostic text entity **Climate Controller Failover Status** with values such as:

- `START`
- `HA`
- `HA: checking readiness`
- `LOCAL: no HA connection`
- `LOCAL: Blueprint not ready`
- `LOCAL: HA values missing`
- `LOCAL: preparing handback`

Debug logging uses the tag `nspanel.addon.climate.controller_failover` and prints owner, API, Blueprint, value mask and display state every 30 seconds.

## Reboot behavior

Both `wifi.reboot_timeout` and `api.reboot_timeout` are set to `0s` by this add-on. This is deliberate: an
automatic ESPHome reboot during a long outage would restart the failover grace period and temporarily remove
local control.

A power loss still causes a normal reboot. After boot, the panel waits `climate_failover_boot_wait_ms`. If
Home Assistant is still unavailable after that interval, the locally stored target and mode are restored.

## UI behavior

When local ownership starts, the add-on switches the Climate detail page to `embedded_climate` and labels it
with `climate_failover_local_label`. The page remains adjustable without Home Assistant. The Home page uses
the NSPanel internal temperature rather than stale HA values.

The add-on also repairs the Climate page if a late HA/Blueprint update overwrites the detail target while
LOCAL still owns the heater. Display initialization uses the normal NSPanel Easy boot/recovery path; ESPHome's
`display.on_setup` fires once per ESPHome boot. The normal display-resync hook handles later display resets.

The indoor-temperature subscription renderer is suspended during LOCAL and
restored on return; incoming HA data remains cached. Climate state changes
repaint only when displayed values, mode, or action change. Home text/color is
cached, and the existing climate chip's change-only rendering remains in use.
Page entries and display resyncs can explicitly force a repaint.

## Limitations

- Heat-only (`heat` / `off`). Cooling and dual-mode installations should use the standard Climate Backup add-on unless this implementation is extended.
- The implementation intentionally integrates with NSPanel Easy internal scripts/flags to provide a seamless
  offline UI. It is therefore more tightly coupled to firmware internals than the standard backup add-on and
  needs CI coverage when those internals change.
- The external heat-request entity must represent the final controller output. Feeding an intermediate PID
  value instead of the final on/off request can cause incorrect relay operation.
- HA target units must match the panel. Automatic unit detection is not provided.
- A live HA instance which stops running its controller but leaves subscriptions,
  Blueprint readiness, and values valid is not detected; no controller heartbeat
  or expiry for unchanged values is provided.
- Readiness is event-driven for values and disconnects, with one-second polling
  for other changes. A non-finite internal sensor value prevents local heating.
- Thermostat startup/minimum-time protections can delay heating after takeover.
- Relay supervision pauses during OTA/TFT upload. Avoid these operations during
  an outage requiring uninterrupted heating. Manual, power-loss, OTA, and the
  project's subscription-configuration restarts remain possible.
- Custom packages must not access the owned GPIO or physical output directly.
- CI fixtures cover Celsius and Fahrenheit, both with relay 1 against latest/dev.
  Hardware tests must separately cover invalid inputs, Blueprint failure, local
  edits, recovery flapping, relay safety, and power-cycle persistence.

## Migration from a local custom package

Users of the earlier local `nspanel_fallback.yaml` package can migrate by renaming substitutions:

| Earlier local name | Add-on name |
| --- | --- |
| `ha_main_climate` | `controller_climate_entity` |
| `ha_heat_request` | `controller_heat_request_entity` |
| `fb_boot_wait_ms` | `climate_failover_boot_wait_ms` |
| `fb_loss_wait_ms` | `climate_failover_loss_wait_ms` |
| `fb_return_wait_ms` | `climate_failover_return_wait_ms` |
| `fb_initial_target` | `climate_failover_initial_target` |
| `fb_initial_heat` | `climate_failover_initial_heat` |
| `fb_local_label` | `climate_failover_local_label` |

The current package adds a GPIO-isolated relay proxy, fresh Blueprint handshake,
compile-time checks, and display-subscription handling to the original local design.
