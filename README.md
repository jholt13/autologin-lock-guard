# autologin-lock-guard

Ever updated a Mac used in an appliance capacity and found it at the lock screen following an OS update? This ensures that following an OS update macOS reboots back to the desktop.

Originally built for our Zoom Rooms running macOS but nothing in the script is Zoom-specific.

## What it installs

| Path | Purpose |
|---|---|
| `/opt/autologin-lock-guard/bin/autologin-lock-guard` | The guard script |
| `/Library/LaunchDaemons/com.github.jholt13.autologin-lock-guard.plist` | Runs the script as root at every boot |

## The problem

A long-standing issue MacAdmins have observed is that macOS does not appear to
respect MDM-managed autologin after a macOS update. After the update, the Mac
shows what looks like the login screen instead of the desktop or the
full-screen app that should be running. Using MDM commands or DDM declarations to enforced an OS
update overnight results in a Mac stuck at the login window instead of signing in and showing the desktop.

This has been reported for years without a root cause — see the
[Jamf Nation thread](https://community.jamf.com/general-discussions-2/macos-update-not-auto-logging-in-after-update-26390),
whose only suggestion is checking "Disable automatic login", and whose reporter
notes the tell: *"the issue doesn't recur until the next update."*

Autologin is not broken. What actually happens:

1. The update installs and reboots.
2. macOS auto-logs the room in, then immediately locks the screen. loginwindow
   logs this as `kLWLockFromDirectLock (8)`. We cannot prevent this.
3. loginwindow persists that state as `autoLoginUserScreenLocked=true` in
   `/Library/Preferences/com.apple.loginwindow.plist`.
4. **Every** later boot re-locks due to that flag, logged as
   `kLWLockFromCookieFile (7)`.

So the room is not at a login window — it is at a *lock screen*, on top of a
live session with Zoom Rooms already running behind it. 

### Evidence

Five consecutive boots on one Zoom Room, same config throughout:

| Boot | Trigger | Lock reason | Result |
|---|---|---|---|
| 11:07:30 | plain reboot | — | unlocked |
| 16:09:55 | plain reboot | — | unlocked |
| 16:52:59 | **OS update** 26.5 → 26.7 | `kLWLockFromDirectLock (8)` | locked, **sets the flag** |
| 17:08:36 | plain reboot | `kLWLockFromCookieFile (7)` | locked (latched) |
| 17:12:25 | flag deleted first | — | unlocked, Zoom Rooms visible |

## When the lock lands

This is the part that matters, and it varies enormously by update type:

| Update | Type | Lock arrives |
|---|---|---|
| 26.5 → 26.7 | minor patch | **~95 seconds** after boot |
| 26.7 → 27.0.1 | major upgrade | **10m 34s** after boot |

A major upgrade runs `Template Migration` and then `Setup Assistant` behind an
installer shield window before the desktop is reachable. The lock lands three
seconds before Setup Assistant exits:

```
13:32:58  boot
13:33:04  Installer Progress shield window appears
13:34:40  Setup Assistant starts
13:43:32  *** kLWLockFromDirectLock (8) ***
13:43:35  Setup Assistant exits
13:43:36  shield window closes
```

An installer shield window is **not** a screen lock and reads as unlocked, so
any fixed wait short enough to be reasonable will conclude "all clear" while the
upgrade is still working. v1.0 waited a fixed 90s, checked once at 13:37:35 with
Setup Assistant 26% complete, and exited — six minutes before the lock arrived.

## What the guard does

There is **no supported way to unlock a locked screen from the CLI**, so this
removes the cause rather than the symptom.

At every boot it:

1. Deletes `autoLoginUserScreenLocked` once a second for 45 seconds.

   This does **not** stop the current boot locking, and is not meant to.
   loginwindow reads the flag \~8s into boot, while launchd has been measured
   starting this daemon **19-92s** in — the race is never winnable. What the
   window buys is cleanup: it removes the flag loginwindow just consumed, and
   re-removes it if loginwindow rewrites it (~19s in), so the **next** boot is
   clean even if the watch below never runs or is interrupted.

   This phase does not watch for a lock, so a lock landing inside it is not
   noticed until step 2 begins. On the two minor updates that cost ~60-90s of
   extra downtime.
2. Compares `sw_vers -buildVersion` against `/var/db/com.github.jholt13.autologin-lock-guard.build`.
   - **Unchanged** — the flag is cleared, nothing else happens.
   - **Changed** (a post-update boot) — *watches* for the lock rather than
     guessing when it will arrive. Every 10s it clears the flag and checks
     `ioreg` for `CGSSessionScreenIsLocked`. The watch runs for at least 5
     minutes, and that clock **restarts for as long as `Setup Assistant` or
     `Installer Progress` is running**, so a slow major upgrade extends it
     naturally. A 45-minute hard cap stops it watching forever.
   - On seeing a lock: clear the flag, stamp the build, reboot once. The next
     boot comes up unlocked with Zoom Rooms on screen.

Net effect on an update: update → boot (locks) → one automatic reboot → clean
boot into Zoom Rooms, unattended.

## Tuning

At the top of the script:

| Constant | Default | Meaning |
|---|---|---|
| `RACE_WINDOW` | `45` | Seconds spent clearing the flag at boot |
| `POLL_INTERVAL` | `10` | Seconds between checks during the post-update watch |
| `GRACE_AFTER_INSTALLER` | `300` | Minimum watch, and how long to keep watching after the installer UI goes away |
| `HARD_CAP` | `2700` | Absolute ceiling on the watch (45 min) |
| `LOG_MAX_BYTES` | `1048576` | Log truncated above this size |

## Logs

`/var/log/autologin-lock-guard.log`, plus `.out`/`.err` from launchd.

**Timestamps are UTC** (`2026-09-29T15:23:29Z`). The daemon can run before the
timezone is configured, so local stamps would render some boots `-0700` and
others `+0000` in the same file.

Beware the same effect when correlating against `log show`: entries written
before the timezone is set render in **UTC**, later ones in **local time**, so a
single boot's output mixes both. A lock at `15:32:02` and one at `08:32:02` may
be the same event. Convert before comparing — the guard's own log is UTC
throughout, which makes it the reliable anchor.

The script truncates its own log above 1 MB. It writes a few lines per boot, so
this should never trigger, but an unrotated daemon log that fills a disk is not
a risk worth taking on an unattended Mac.

## Finding affected devices

Any Mac returning `1` is latched right now and will be locked after every reboot
until cleared:

```bash
defaults read /Library/Preferences/com.apple.loginwindow autoLoginUserScreenLocked 2>/dev/null || echo "not set"
```

Manual remediation, if you are not installing the package:

```bash
sudo defaults delete /Library/Preferences/com.apple.loginwindow autoLoginUserScreenLocked && sudo reboot
```

Prefer this over the `rm com.apple.loginwindow.plist` advice found in forums —
deleting the whole plist discards `autoLoginUser`, `RecentUsers` and the rest,
and relies on the configuration profile to re-push autologin.

## Prerequisites

- Autologin configured (`autoLoginUser`, `/etc/kcpassword`, or an
  `AutologinUsername` configuration profile payload)
- FileVault **off** — autologin does not apply otherwise, and neither does this

The script is inert on any Mac where the flag is never set, so it is harmless
if it lands out of scope.

## Testing locally

```bash
sudo installer -pkg build/autologin-lock-guard-1.1.pkg -target /
```

Confirm it armed:

```bash
sudo launchctl print system/com.github.jholt13.autologin-lock-guard | head -20
cat /var/db/com.github.jholt13.autologin-lock-guard.build; echo
cat /var/log/autologin-lock-guard.log
```

To exercise the post-update path without an actual update, fake a build change
and reboot:

```bash
echo -n "FAKE123" | sudo tee /var/db/com.github.jholt13.autologin-lock-guard.build >/dev/null
sudo reboot
```

The guard should log a build change, and reboot once more only if the screen is
locked. Confirm a boot is clean with:

```bash
log show --start "$(date '+%Y-%m-%d %H:%M:%S' -r $(sysctl -n kern.boottime | sed 's/.*sec = \([0-9]*\).*/\1/'))" \
  --style compact | grep -c 'kLWLockFrom'
```

`0` means the room booted straight to the desktop.

