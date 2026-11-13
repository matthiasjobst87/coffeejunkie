# coffeejunkie

Counts the espresso shots of a simple espresso machine from the power readings of smart plugs. It tells espresso
from cappuccino, keeps statistics, reminds you of backflushing, descaling and grinder cleaning, and tracks your bean stock.

It is a Home Assistant **package** (one YAML file) built only from Home Assistant's own building blocks: template sensors, helpers, scripts,
automations and utility meters. No custom integration, no Python, no `shell_command`. An optional **blueprint**
sends quiet reminders to your phone, and an **example dashboard** uses built-in cards only.

*Deutsche Anleitung: [README.de.md](README.de.md)*

## How it works

| What happens | What the plugs see |
|---|---|
| Grinding | Grinder above 100 W for 15 to 40 s |
| Brew | Within a few minutes after the grinder, the machine pump runs (60 to 130 W, heater pulses in between) for 25 to 50 s |
| Milk frothing | Shortly after a brew, the machine heats without pause for more than 90 s (steam) |

From this, the package detects:

- **Brew:** with a grinder plug, a grind followed by the pump within 5 minutes. A cappuccino counts as a brew too.
- **Cappuccino:** a brew followed by long continuous heating within about 5 minutes.
- **Grinder cleaning:** a long grind (45 s or more) without a brew afterwards books the grinder cleaning automatically. This only makes sense if the grinder is used for espresso alone; set the threshold to 0 to turn it off.

All thresholds are helpers that you can change in the dashboard. The defaults were measured on an Ascaso thermoblock machine.

### Two modes

| Mode | Plugs | Reliability |
|---|---|---|
| `with_grinder` (default) | machine and grinder | Very good: in a 9-day test, 22 of 22 brews and 4 of 4 cappuccinos, no false counts. |
| `machine_only` | machine only | Good, with a known weakness: anything that runs the pump as long as a brew counts as one (backflushing, descaling, flushing the group). In the test, 10 of 10 brews plus one flush counted as a brew. |

## Requirements

- Home Assistant 2024.10 or newer.
- A smart plug with power measurement on the espresso machine, and ideally one on the grinder.
- **The plugs must report often.** A brew lasts about 30 seconds, so the plug has to report at least every few seconds. This is the most important requirement:
  - Shelly: newer devices often report every second (the defaults were measured with a Gen4). Some report only on larger changes. Check the history of the power sensor during a brew.
  - Tasmota: the default is every 5 minutes. Set `TelePeriod 10` and `PowerDelta 101` (report on any change of 1 W or more).
  - Zigbee plugs: depends on the device; many report only above a minimum change. Configure the reporting interval if your integration allows it.
  - Tuya, Kasa/Tapo and other cloud plugs: often only every 30 to 60 seconds, too slow.
- Plugs and machine must handle the load. An espresso machine draws over 1000 W. Use a plug rated for it.

## Installation

1. **Enable packages** in `configuration.yaml`, if you have not already:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy [`packages/coffeejunkie.yaml`](packages/coffeejunkie.yaml) to `<config>/packages/coffeejunkie.yaml`.
3. In section 1 of the file, replace `sensor.espresso_machine_power` and `sensor.grinder_power` with the power sensors of your plugs (in W). These are the only lines that name your own entities.
4. **Check the configuration** (Developer tools > YAML > Check configuration) and **restart Home Assistant**. Utility meters and new helpers need a restart. On the first start all settings get their defaults, the maintenance clocks start and the bean stock is set to a full bag.
5. Without a grinder plug: set **Coffeejunkie mode** to `machine_only`.
6. Optional: add the [example dashboard](dashboards/coffeejunkie_view.yaml) and the [reminder blueprint](#reminders-on-your-phone).
7. Make one espresso and check the counter. If nothing is counted, see [Calibration](#calibration).

## Calibration

The defaults fit many simple single-boiler and thermoblock machines, but not all. Make one espresso (and one cappuccino if you like), then look at the
**Live and calibration** section of the example dashboard or at the history of `sensor.coffeejunkie_machine_power`.

| Setting | How to find the value |
|---|---|
| Grinder runs above | Power of the grinder while grinding, minus a margin. Must be clearly above its standby. |
| Pump band from / up to | Lowest and highest power while **only the pump** runs (in the first seconds of the brew). Must stay below the heater. |
| Heating above | Below the heater power (often 900 to 1500 W) and above the pump. |
| Heating must last | Shorter than steaming milk, longer than the heater pulses while the machine keeps warm. |
| Portafilter active from / after | Power and time that only a brew reaches. Second way to recognise a brew if the heater runs during the whole brew. |
| Machine only, brew at least / at most | Shortest and longest brew, measured from the start to the end of the pump. Keep rinsing (short) and descaling (long) outside. |

The detection sensors in the dashboard (grinder running, pump running, pump activity, portafilter active, heating) show live
what the package sees. A brew should switch on *grinder running*, then *pump running* or *portafilter active*.

If you only want to try values, change them in the dashboard. **Coffeejunkie: reset settings to defaults** restores them; counters are not touched.

## What you get

| Entity | Content |
|---|---|
| `sensor.coffeejunkie_total`, `sensor.coffeejunkie_cappuccino_total` | Brews and cappuccinos since the start (a cappuccino counts as a brew too) |
| `sensor.coffeejunkie_today` / `_week` / `_month` / `_year` | Brews per period, also for cappuccino (`sensor.coffeejunkie_cappuccino_today` …) |
| `sensor.coffeejunkie_distribution` | Attributes `hours`, `hours_cappuccino` (24 values), `weekdays` (Mon to Sun), `days`, `days_cappuccino` (last 35 days), `recent` (last 10 brews with time and type), `since` |
| `sensor.coffeejunkie_care_backflush`, `_descale`, `_grinder` | `ok` or `due`; attribute `text` (for example "in 18 brews or 5 days") and the details |
| `sensor.coffeejunkie_beans_left` | Brews the beans last for; attributes `stock_g`, `bag_g`, `percent` |
| `binary_sensor.coffeejunkie_*` | Detection: grinder running, pump running, pump activity, portafilter active, heating |
| `script.coffeejunkie_backflush_done`, `_descale_done`, `_grinder_done` | Book a maintenance as done (date and brew count) |
| `script.coffeejunkie_new_bag` | Bean stock back to the bag size |
| `script.coffeejunkie_record_brew` | Add a missed brew by hand (optional `time`) |
| `script.coffeejunkie_reset_settings` | All settings back to the defaults |
| event `coffeejunkie_brew`, `coffeejunkie_milk` | Fired on every brew and cappuccino, for your own automations |

The counters are trigger-based template sensors that count the events themselves. Home Assistant restores their state and attributes after a restart or reload. There is no file or database behind them.

**Maintenance** is due after a number of brews *or* days, whichever comes first (0 = that limit is off). Defaults: backflush every 40 brews or 14 days, descale every 90 days, clean grinder every 100 brews or 56 days. Follow the manual of your machine.

## Reminders on your phone

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/create-link/?redirect=blueprint_import&blueprint_url=https%3A%2F%2Fgithub.com%2Fmatthiasjobst87%2Fcoffeejunkie%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fcoffeejunkie%2Fcoffeejunkie_reminders.yaml)

The blueprint [`coffeejunkie_reminders.yaml`](blueprints/automation/coffeejunkie/coffeejunkie_reminders.yaml) sends one quiet notification (Companion app) when a maintenance becomes due or the beans run low,
reminds you once a day while something is open, and removes the notification when everything is done. You choose the phones, the time window,
the daily reminder time, the bean limit and optionally the people that must be home. On Android it uses its own channel with low importance; on iOS it is passive (no sound).

## Example dashboard

[`dashboards/coffeejunkie_view.yaml`](dashboards/coffeejunkie_view.yaml) is a sections view with built-in cards: counters, brews per day (30 days), latest brews, by hour and weekday,
maintenance and beans (tap to book as done, with confirmation), live power and detection for calibration, and all settings. Instructions are at the top of the file.

## Limits

- Tested on one machine (Ascaso thermoblock, vibration pump) with one plug type (Shelly Power Strip 4 Gen4, about one reading per second). Machines with large boilers, heat exchangers or rotary pumps will need different values. Curves of other machines are welcome in the issues.
- Mode `machine_only` cannot tell a brew from backflushing, descaling or flushing the group when they run as long as a brew.
- A brew without grinding (pre-ground coffee, pods) is not counted in mode `with_grinder`.
- Two brews in a row from one grind count as one.
- The plug only measures. coffeejunkie never switches it.

## Removing

Delete `packages/coffeejunkie.yaml`, restart, then remove leftover entities under Settings > Entities (search "coffeejunkie").

## License

MIT, see [LICENSE](LICENSE). By matthiasj.
