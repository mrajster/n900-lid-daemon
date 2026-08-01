# n900-lid — slider-driven screen, lock & keyboard backlight for the Nokia N900 on postmarketOS

Close the keyboard slider → screen off, touch off, session locked.
Open it → one clean wake transition (no backlight flash), unlocked, keyboard
LEDs lit for 10 s. Any keypress re-lights the keyboard for 10 s. Optionally,
a closed slider suspends the device (s2idle) after a timeout.

Plain POSIX `sh` + `evtest`. No systemd, no dbus daemons, no compositor hooks —
works on the stock pmOS i3 image and survives package upgrades.

Tested on: Nokia N900, postmarketOS (edge, mainline kernel), i3 + tinydm + X11.

## How it works

The N900 keyboard slider is a plain evdev switch: `EV_SW / SW_KEYPAD_SLIDE`
on the `gpio_keys` input device. Two quirks make this less trivial than it
sounds, and they're the reason this daemon exists:

1. **`evtest --query` exit codes**: `10` = slider OPEN, `0` = closed
   (verified on hardware — the manpage doesn't tell you this).
2. **pmOS's own screenlock fights you.** The stock i3 config runs `xautolock`
   with a `screenlock.sh` that disables the touchscreen and sets a 3-second
   DPMS timeout. Left alone, it re-blanks and re-deadens the screen behind
   your back. The `local.d/n900-lid-overrides.start` script neuters exactly
   those two pieces, idempotently, on every boot (so a package upgrade can't
   silently bring them back).

The daemon (`bin/n900-lid`):

- follows slider events with a persistent `evtest` pipe (auto-restarts if
  evtest dies);
- **open** → restore WiFi if the closed-timer disabled it, kill `i3lock`,
  raise the framebuffer + DPMS + touchscreen *first* and the backlight *last*
  — one clean transition instead of the classic backlight-then-content double
  flash;
- **close** → `i3lock`, touch off, DPMS off, framebuffer blank, backlight 0,
  keyboard LEDs off, and start the closed-timer;
- **closed-timer** (each minute, while closed):
  - holds off while an SSH session is open or `apk` is running (never
    suspend/degrade a device someone is working on),
  - after `SUSPEND_AFTER` s → s2idle suspend (`echo freeze >
    /sys/power/state`) — **disabled by default**, see below,
  - if suspend is disabled/guarded: after `WIFI_OFF_AFTER` s → WiFi radio off
    (the single biggest idle consumer on this device); reopening restores it.

Keyboard backlight: the six `lp5523:kb*` LEDs are driven on demand. A
watcher inside the daemon lights them on any keypress of the TWL4030 keypad
and stamps `/run/n900-lid/kbd.last`; the one-shot `bin/n900-kbd-dim` helper
turns them off 10 s after the last press. Cheap, no polling while dark.

## Install

Dependencies (all in pmOS repos):

```sh
apk add evtest xinput i3lock
```

Files → places:

```sh
install -m755 bin/n900-lid bin/n900-kbd-dim /usr/local/bin/
install -m755 openrc/n900-lid /etc/init.d/
install -m755 local.d/backlight.start local.d/n900-lid-overrides.start /etc/local.d/
rc-update add n900-lid default
rc-update add local default      # if not already enabled
rc-service n900-lid start
```

### Things you must check for your setup

| Variable (top of `bin/n900-lid`) | Default | How to verify yours |
|---|---|---|
| `EVDEV` | `/dev/input/event4` | `evtest` — pick the device listing `SW_KEYPAD_SLIDE` |
| `BL` | `acx565akm` backlight | `ls /sys/class/backlight/` |
| `FB` | `omapdrm.0 … fb0/blank` | `ls /sys/devices/platform/omapdrm.0/graphics/` |
| `TOUCH` | `TSC2005 touchscreen` | `xinput list` |
| `XUSER` | `user` | your desktop login (pmOS default is `user`) |
| `WIFI_OFF_AFTER` | 600 s | taste |
| `SUSPEND_AFTER` | **0 (off)** | see warning below |

### ⚠️ Before enabling suspend (`SUSPEND_AFTER` > 0)

The stock mainline kernel **crashes during suspend entry on OMAP3** (an
i2c-omap noirq bug) — it looks like "suspends but never wakes" and needs a
battery pull. Fix your kernel first:
**[mrajster/n900-pmos-suspend-fix](https://github.com/mrajster/n900-pmos-suspend-fix)**.

With the patched kernel, set `SUSPEND_AFTER=300` (or whatever you like) and a
closed slider suspends after that many seconds; slider-open, power button, or
an RTC alarm wake it. The daemon deliberately refuses to suspend while an SSH
session is open or `apk` is running.

For the slider to wake the device from suspend, its `gpio_keys` node needs
`wakeup-source` in the device tree — check with:

```sh
ls /sys/firmware/devicetree/base/gpio_keys/keypad_slide/ | grep wakeup-source
```

(If missing: `fdtput yourdtb /gpio_keys/keypad_slide wakeup-source -t s ''`
and re-append the DTB the way your boot chain expects.)

## Files

| Path | What |
|---|---|
| `bin/n900-lid` | The daemon: slider → screen/lock/suspend + keypress → kbd LEDs |
| `bin/n900-kbd-dim` | One-shot LED off helper (spawned by the daemon) |
| `openrc/n900-lid` | OpenRC service |
| `local.d/backlight.start` | Panel backlight to max at boot (comes up near-zero otherwise) |
| `local.d/n900-lid-overrides.start` | Neuters pmOS xautolock/screenlock conflicts, idempotently |

## Design notes / gotchas collected the hard way

- **Order on wake matters**: framebuffer + DPMS + touch first, backlight
  last. The reverse gives a bright flash of stale content.
- **Kill timers by process group.** The closed-timer runs under `setsid` and
  is killed with `kill -TERM -- -PGID`; busybox `sh` never delivers a plain
  `kill` to a subshell's `sleep` children, which leaks orphaned sleeps that
  fire minutes later and turn your WiFi off mid-use.
- **`pkill -x`, never `pkill -f`**, on busybox: `pkill -f` matches its own
  command line and kills your SSH session's shell. (Exit code 255 right after
  a pkill over SSH = you did this.)
- **The lp5523 LED engines** are disabled at boot (see pmOS `n900` device
  package docs); driving `brightness` directly is all you need.
- `su -s /bin/sh user -c "DISPLAY=:0 XAUTHORITY=… xset …"` is the whole
  "talk to X from a root daemon" story — no dbus/logind needed.

## License

MIT
