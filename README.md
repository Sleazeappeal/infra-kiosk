# infra-kiosk: a Netdata dashboard on a touch screen

## Credits

This is a thin layer over other people's work:

- **[Netdata](https://github.com/netdata/netdata)**, which collects every number shown here and serves it
  over its REST API.
- **[cage](https://github.com/cage-kiosk/cage)**, the Wayland kiosk compositor by Jente Hidskes and contributors.
  `kiosk/infra-kiosk.service` follows the
  [Starting Cage on boot with systemd](https://github.com/cage-kiosk/cage/wiki/Starting-Cage-on-boot-with-systemd)
  wiki page.
- **[wlr-randr](https://gitlab.freedesktop.org/emersion/wlr-randr)** by Simon Ser, for the output rotation.
- **[Firefox](https://www.mozilla.org/firefox/)** in kiosk mode, and **[libinput](https://wayland.freedesktop.org/libinput/doc/latest/)**
  for the touch calibration matrix.
- Fonts **[IBM Plex Mono](https://github.com/IBM/plex)** (IBM) and **[VT323](https://fonts.google.com/specimen/VT323)**
  (Peter Hull), both SIL Open Font License, loaded from Google Fonts.

Thank you to all of them.

## Requirements

The kiosk setup (`kiosk/`) is built for one hardware combination, and that is the only one it was tested on:

- **Raspberry Pi 5**
- **Raspberry Pi Touch Display 2**: 7", 720x1280 portrait, on the Pi's DSI port, with a Goodix touch
  controller. It is used in landscape (`KIOSK_TRANSFORM=270`).
- **Raspberry Pi OS** based on Debian 13, with its `firefox` package, plus `cage` and `wlr-randr`
- one or more **Netdata** agents that the Pi can reach on port 19999

Several parts depend on this hardware:
- the DSI output lookup and the rotation;
- the Goodix module reload at start;
- the backlight control through `/sys/class/backlight`;
- the touch calibration matrices;
- the Firefox CPU tuning for a passively cooled Pi.

Other hardware will need changes in these places.

The dashboard on its own (`dashboard/infra-dashboard.html`) has no hardware requirement. Any current browser
that can reach the Netdata agents can show it.

## What it is

One HTML file, [`dashboard/infra-dashboard.html`](dashboard/infra-dashboard.html), that reads a few Netdata
agents straight from the browser and shows them in a green-on-black terminal style. There is no build step,
no framework and no backend for the data.

- **Desktop view** (open the file in any browser): one column per host with CPU, memory, temperatures,
  storage and network cards, each with a short history curve.
- **Kiosk view** (`?kiosk`): a no-scroll layout for a 1280x720 touch screen. The overview shows one tile per
  host plus a ping row. The pages rotate on their own, and a tap on a tile opens that host's full column.

The `kiosk/` folder turns a Raspberry Pi 5 with the Touch Display 2 into an appliance that shows the kiosk
view. It boots straight into the dashboard, needs no login, dims at night and wakes up on touch.

The UI labels are in Hungarian (`CPU kihasználtság` = CPU usage, `Hőmérsékletek` = temperatures,
`Tárhely` = storage, `Hálózat` = network, `minden rendben` = all good). Code and comments are in English.

## Where the data comes from

All metrics come from Netdata. The page only ever talks to the Netdata agents in `CONFIG.hosts` and, in the
kiosk, to the local helper.

| What you see | Source | Request |
|---|---|---|
| Which cards a host gets | Netdata chart list | `GET <host>/api/v1/charts`, once at page load (retried until the host answers) |
| Card values and curves | Netdata metrics | `GET <host>/api/v1/data?chart=<id>&after=-<s>&points=<n>&group=average&format=json`, one per card |
| Ping row | the go.d `ping` collector job of the **first** host in `CONFIG.hosts` | charts `ping_<job>.host_packet_loss_<addr>` and `…host_rtt_<addr>`, average of the last 15 s |
| Service LEDs | Netdata `systemdunits` collector on `CONFIG.serviceHost` | chart `systemdunits_service-units.unit_<unit>_service_state`, dimension `active` |
| Devices without Netdata | an agent that polls them over SNMP, see [below](#devices-without-netdata-snmp) | same as card values |
| Screen dark or lit (kiosk only) | `infra-kiosk-helper` on 127.0.0.1:8090 | `GET /api/state`; a tap sends `POST /api/wake` |
| Clock | the browser | none |

Cards are not configured one by one. After loading the chart list, the page matches the chart IDs against
the rules in `TEMPLATES`, and every rule that finds a chart becomes a card:

- **CPU:** `system.cpu`, `system.load`, and the CPU package temperature (`coretemp`, `k10temp` or the
  Raspberry Pi `cpu_thermal` sensor).
- **Memory:** `system.ram`, `mem.swap`.
- **Temperatures:** PECI or Tctl, the board sensor (SYSTIN, `acpitz` or the Pi 5 RP1), NVMe, and up to two
  `smartctl` disks.
- **Storage:** `disk_space./` plus `CONFIG.mounts`, and up to two LVM thin pools.
- **Network:** the main interface (`vmbr0`, `eth0` or the first physical one) and `system.net`.

Netdata's API allows cross-origin reads, so the browser fetches it directly and no proxy is needed. The page
must be able to reach every agent's port 19999.

## How often it refreshes

All intervals live in the `CONFIG` block.

| What | Interval | Notes |
|---|---|---|
| All cards, ping row | `refreshMs` = 2 s (desktop), `kioskRefreshMs` = 5 s (kiosk) | each tick asks every agent for the latest point of each card, averaged over the last 2 s |
| History curves | same tick | 90 points over the last 180 s. They are only fetched where a curve is visible: everywhere on the desktop; in the kiosk only on the open host page and for the CPU curve on the tiles |
| Service LEDs | `serviceMs` = 30 s | |
| Kiosk page rotation | `pageMs` = 20 s | paused for `idleMs` = 60 s after the last tap |
| Backlight state | 5 s | kiosk only, from the helper |
| Chart list (which cards exist) | once | reload the page after adding a collector |

The request load is about one small HTTP request per card per tick, a few dozen per agent.

## Repository layout

| File | Installed as |
|---|---|
| `dashboard/infra-dashboard.html` | `/opt/infra-kiosk/infra-dashboard.html` (or open it anywhere) |
| `kiosk/infra-kiosk.service` | `/etc/systemd/system/` (cage + Firefox on tty7) |
| `kiosk/infra-kiosk-helper.service` | `/etc/systemd/system/` |
| `kiosk/infra-kiosk-ctl.service`, `kiosk/infra-kiosk-ctl.timer` | `/etc/systemd/system/` (day/night schedule) |
| `kiosk/infra-kiosk-helper` | `/usr/local/bin/` (serves the page, drives the backlight) |
| `kiosk/infra-kiosk-session` | `/usr/local/bin/` (runs inside cage: rotation, Firefox profile, Firefox) |
| `kiosk/infra-kiosk-ctl` | `/usr/local/bin/` (schedule and pause) |
| `kiosk/infra-kiosk-rotate` | `/usr/local/bin/` (orientation + touch calibration) |
| `kiosk/infra-kiosk.default` | `/etc/default/infra-kiosk` |
| `kiosk/pam-infra-kiosk` | `/etc/pam.d/infra-kiosk` |
| `kiosk/90-backlight.rules` | `/etc/udev/rules.d/` (lets the `video` group set the backlight) |
| – | `/etc/udev/rules.d/91-touch.rules`, written by `infra-kiosk-rotate` |

## Configuration

Everything site-specific is in the `CONFIG` block at the top of the `<script>` in
`dashboard/infra-dashboard.html`:

- `title`: header text.
- `hosts`: the Netdata agents, `{key, base}`. `key` is the display name and also a URL parameter that
  overrides the address without editing the file (`?node1=http://10.0.0.5:19999`).
- `pings`: machines without Netdata that the first host's ping job watches. `host` must be the address
  exactly as written in that ping job.
- `mounts`: mount points shown besides `/`.
- `hostThresholds`: per-host `[warn, crit]` overrides by card label, for example a CPU that idles hot.
- `serviceHost`, `services`: which host's systemd units get LEDs, as `[label, unit]`.
- `refreshMs` … `idleMs`: see [How often it refreshes](#how-often-it-refreshes).

On the desktop the base URL inputs under the columns also switch a host (type the URL, press Enter).

## Kiosk setup

On the hardware and OS listed under [Requirements](#requirements), with the Netdata agents already running:

```bash
sudo apt install cage wlr-randr
sudo useradd --system --create-home --home-dir /var/lib/infra-kiosk --shell /usr/sbin/nologin \
  --groups video,render,input infra-kiosk
sudo install -m 0644 kiosk/90-backlight.rules /etc/udev/rules.d/
sudo udevadm trigger --subsystem-match=backlight --action=add
sudo install -m 0644 kiosk/infra-kiosk.service kiosk/infra-kiosk-helper.service \
  kiosk/infra-kiosk-ctl.service kiosk/infra-kiosk-ctl.timer /etc/systemd/system/
sudo install -m 0644 kiosk/pam-infra-kiosk /etc/pam.d/infra-kiosk
sudo install -m 0644 kiosk/infra-kiosk.default /etc/default/infra-kiosk
sudo install -m 0755 kiosk/infra-kiosk-helper kiosk/infra-kiosk-session kiosk/infra-kiosk-ctl \
  kiosk/infra-kiosk-rotate /usr/local/bin/
sudo install -d -m 0755 /opt/infra-kiosk
sudo install -m 0644 dashboard/infra-dashboard.html /opt/infra-kiosk/
sudoedit /etc/default/infra-kiosk          # KIOSK_TOUCH_MODULE: empty unless your touch driver needs a reload
sudo infra-kiosk-rotate 270                # writes the touch rule, see below
sudo systemctl daemon-reload
sudo systemctl start infra-kiosk-helper
curl -s http://127.0.0.1:8090/api/state; echo    # {"pct": 50} by day
sudo systemctl start infra-kiosk
```

If the picture, the touch and the page rotation look right, enable it and test a reboot:

```bash
sudo systemctl enable infra-kiosk-helper infra-kiosk infra-kiosk-ctl.timer
```

The login prompt stays on tty1 (Ctrl+Alt+F1 with a keyboard).

To update the dashboard, copy the new `infra-dashboard.html` to `/opt/infra-kiosk/` and run
`sudo systemctl restart infra-kiosk`. The helper sends `Cache-Control: no-store`, so Firefox never shows an
old copy.

## Orientation and touch calibration

cage does not rotate the screen on its own. Kernel cmdline rotation such as `video=DSI-1:…,rotation=270` only
turns the text console. And touch controllers report raw panel coordinates whatever way the picture is turned.
So two settings must always match:

- `KIOSK_TRANSFORM` in `/etc/default/infra-kiosk`: `infra-kiosk-session` passes it to
  `wlr-randr --transform` (`normal`, `90`, `180`, `270`).
- the libinput calibration matrix in `/etc/udev/rules.d/91-touch.rules`: it maps every touch to the
  rotated picture.

`infra-kiosk-rotate` sets both, reloads udev and restarts the kiosk:

```bash
sudo infra-kiosk-rotate 270                     # landscape on a portrait panel (the tested setup)
sudo infra-kiosk-rotate 90                      # the same panel the other way up
sudo infra-kiosk-rotate 270 "0 1 0 -1 0 1"      # with a measured matrix
```

| `KIOSK_TRANSFORM` | stock matrix |
|---|---|
| `normal` | `1 0 0 0 1 0` |
| `90` | `0 -1 1 1 0 0` |
| `180` | `-1 0 1 0 -1 1` |
| `270` | `0 1 0 -1 0 1` (measured) |

**Touch calibration.** If touches land in the wrong place, measure the panel:

1. `sudo systemctl stop infra-kiosk`, then `sudo libinput debug-events`. The touchscreen's `TOUCH_DOWN`
   lines show each touch as `x/y` in percent of the raw panel range (the `x/y` before the `(…mm)` part).
2. Touch the **top-left**, **top-right** and **bottom-left** corners of the picture as you want to see it.
   Write them down as fractions (98% = 0.98): (x0, y0), (x1, y1), (x2, y2).
3. If every value is close to 0 or 1, round them: one of the stock matrices above will fit. Otherwise compute
   the matrix `a b c d e f`:

   ```
   det = (x1-x0)*(y2-y0) - (x2-x0)*(y1-y0)
   a =  (y2-y0)/det     b = -(x2-x0)/det     c = -(a*x0 + b*y0)
   d = -(y1-y0)/det     e =  (x1-x0)/det     f = -(d*x0 + e*y0)
   ```

   Example (the tested panel at 270): top-left 0.98/0.02, top-right 0.98/0.99, bottom-left 0.01/0.02 round
   to (1,0), (1,1), (0,0), giving `0 1 0 -1 0 1`.
4. `sudo infra-kiosk-rotate <transform> "a b c d e f"`.

## Day/night schedule and backlight

- `infra-kiosk-helper` sets the backlight to 50% by day and turns it off between 22:00 and 08:00. Every tap
  gives 100% for 120 s. On a dark screen the first tap only wakes it and does not change the page. The values
  are at the top of the helper (`DAY_PCT`, `NIGHT_FROM`, `NIGHT_TO`, `BOOST_S`).
- `infra-kiosk-ctl` (timer every 5 min) runs cage + Firefox only between 08:00 and 22:00. At night it stops
  them and keeps the screen dark. Its hours (`DAY_FROM`, `NIGHT_FROM`) must match the helper's.
- Pause for any length of time, surviving reboots: `sudo infra-kiosk-ctl pause "tomorrow 18:00"`. Clear it
  with `resume`; `status` shows the state.

## Devices without Netdata: SNMP

A NAS, a router or a switch usually cannot run Netdata, but one of your agents can poll it with Netdata's
go.d `snmp` collector.
The charts then come through the same API, so the dashboard needs no other data source. Suggested setup:

1. On one agent, add a job to `/etc/netdata/go.d/snmp.conf` (`root:netdata`, mode `0640`, it holds the
   credentials). Prefer SNMPv3:

   ```yaml
   # without this, a job whose first check fails (device still booting) stays disabled
   autodetection_retry: 60
   jobs:
     - name: nas
       hostname: 192.0.2.20
       update_every: 60          # SNMP agents on small devices are slow; 30-60 s is plenty
       create_vnode: false       # keep the charts on this agent, where the dashboard looks
       options:
         version: "3"
         timeout: 10
       user:
         name: netdata
         level: authPriv
         auth_proto: sha
         auth_key: "<auth key>"
         priv_proto: aes
         priv_key: "<priv key>"
   ```

   Netdata picks a device profile by the device's sysObjectID. Its charts are named
   `snmp_<job>.snmp_device_prof_<metric>`. List them with
   `curl -s http://node1:19999/api/v1/charts | jq -r '.charts | keys[]' | grep '^snmp_nas\.'`.
2. Give the device its own column with an `only` filter. These charts then show only there:

   ```js
   {key:"nas", base:"http://node1.example.lan:19999", only:/^snmp_nas\./},
   ```
3. Add cards for its charts after the `TEMPLATES` array, and add the section name to `SECTION_ORDER`. A card
   labelled like one in `TILE_ROWS` (`Rendszer`, `RAM`, …) also shows on the kiosk tile. For example:

   ```js
   TEMPLATES.push(
     {sec:"SNMP", label:"Uptime", unit:"nap", kind:"gauge", val:first, fmt:v=>(v/86400).toFixed(1),
       resolve:ids=>find(ids,/^snmp_nas\.snmp_device_prof_systemUptime$/)},
     {sec:"SNMP", label:"Rendszer", unit:"°C", kind:"gauge", val:first, fmt:f0, th:[50,60],
       resolve:ids=>find(ids,/^snmp_nas\.snmp_device_prof_qnap_system_temperature$/)},
   );
   ```

   The chart IDs depend on the profile, so take them from the list in step 1.

## Keeping a passively cooled Pi cool

Firefox renders the page on the CPU when it falls back to software WebRender (check `about:support` →
*Graphics* → *Compositing*). A first version with animations ran a passively cooled Pi 5 into thermal
throttling. The kiosk view therefore:

- switches off the scanline overlay, the LED pulse, all glow (`text-shadow`/`box-shadow`) and the fixed
  background, so the page only repaints when data changes;
- polls every 5 s instead of 2 s, and fetches history only for visible curves;
- runs Firefox at `layout.frame_rate` = 10 (`user.js`, rewritten on every start).

Together this took Firefox from about 37% to about 14% CPU on the Pi.

## Troubleshooting

- Logs: `journalctl -u infra-kiosk -u infra-kiosk-helper -b --no-pager | tail -n 40`.
- A host tile says offline: check `curl http://<host>:19999/api/v1/info` from the kiosk.
- A card is missing: the chart ID differs on that host. Compare `/api/v1/charts` with the `resolve` rules
  in `TEMPLATES`.
- The ping row says `nincs ping chart` (no ping chart): the address in `CONFIG.pings` differs from the one in
  the ping job.
- Back to the console: `sudo systemctl disable --now infra-kiosk infra-kiosk-helper infra-kiosk-ctl.timer`.

## License

GPL-2.0, see [LICENSE](LICENSE).
