# Dimlight

**Stock AOD for the Galaxy A17 5G: a KernelSU module that brings Samsung's native Always On Display to the SM-A176B on One UI 9.**

Dimlight overlays Samsung's own AODService on the stock system and fixes the conflicts that causes, so you get the original AOD with a basic clock and an AOD wallpaper, and no third-party fullscreen AOD app.

> **Status: beta (r8).** The lock screen clock, the AOD settings page, AOD with wallpaper and the clock all work on the tested phone. A few limitations remain (see [Known issues](#known-issues)).

---

## Tested on

| | |
|---|---|
| Device | Galaxy A17 5G (SM-A176B) |
| Android / UI | Android 17, One UI 9 |
| Firmware | `A176BXXU8DZJ5` |
| Root | KernelSU with the [`meta-overlayfsx`](https://github.com/RipperHybrid/Meta-Overlayfsx) metamodule |
| Module version | `8.9.66.1-r8` (versionCode `896601006`) |

Other Samsung phone models are untested, and the outcome is unexpected. Proceed with caution. Note that the installer's model check (in `customize.sh`) currently refuses anything that isn't `SM-A176*`, so you would have to edit it out first.

## What it does

1. **Adds Samsung's AODService** (version 8.9.66.1) as a system_ext priv-app, with its permission XMLs.
2. **Fixes the missing lock screen clock.** Stock firmware already includes `com.samsung.android.app.clockpack`, which registers four SystemUI plugins. AODService registers the same four with identical class names, so SystemUI sees duplicates and drops the clock. `service.sh` disables AODService's four copies after boot:
   - `PluginKeyguardStatusViewImpl`
   - `UfClockProvider`
   - `LockItemPluginService`
   - `PluginViewCoverImpl`

   The AOD plugin (`PluginAODImpl`) stays enabled.
3. **Fixes the AOD settings crash and enables the AOD wallpaper.** AODService's database code adds its settings table only when the `CLOCKPACK_ITEM` floating feature does not contain `clockpackversion`. On this firmware it does, so the table was skipped and the app crashed on start. The module ships its own copy of `/system/etc/floating_feature.xml` (your stock file plus edits): it adds an `AOD_ITEM` line, removes the `CLOCKPACK_ITEM` line, and sets `SEC_FLOATING_FEATURE_LCD_CONFIG_AOD_FULLSCREEN` to `1`, which turns on the AOD wallpaper.
4. **Turns AOD on once.** On the first boot after flashing, `service.sh` sets `aod_mode=1`. After that your own setting is left alone.
5. **Compiles AODService (r8).** About 45 seconds after boot, `service.sh` compiles the AODService package with `speed` if it isn't already. Otherwise the package runs uncompiled (`run-from-apk`) and its plugin loads slower when the screen turns off. This is done once and survives reboots.

## Requirements

- Galaxy A17 5G (SM-A176B), rooted with KernelSU
- A metamodule that mounts module files (tested with [`meta-overlayfsx`](https://github.com/RipperHybrid/Meta-Overlayfsx))
- adb (optional, for logs and rollback)
- AOD set to **Show always** in the AOD settings (see [Known issues](#known-issues))

## Download

The completed, ready-to-flash module zip (`DimLight V1.1.zip`, release **v1.1 beta**) is in the [Releases](../../releases) section. There's nothing to build. It already includes Samsung's AODService 8.9.66.1 APK, the permission XMLs, `service.sh`, and the modified `floating_feature.xml`.

## Installing

1. Flash the zip in KernelSU Manager.
2. Do a **full reboot** (not a soft reboot), so the module files get mounted.
3. Make sure no older AODService module is installed alongside this one. The module id is unchanged between revisions, so a new flash updates in place.
4. Do not use this together with the [GueRapii AODFullScreen-Enabler](https://github.com/GueRapii/AODFullScreen-Enabler). Both overlay `floating_feature.xml` and will conflict.
5. Open **Settings > Lock screen > Always On Display** and choose **Show always**.

## Rolling back

Disable the module in KernelSU Manager and reboot, or from a shell:

```
adb shell "su -c 'touch /data/adb/modules/aodservice_a176b_v2/disable'"
```

Then do a full reboot. This turns off all of the module's changes.

## Options

Create an empty file in `/data/adb/modules/aodservice_a176b_v2/` and reboot to change behavior:

| File | Effect |
|---|---|
| `no_force_aod` | Don't set `aod_mode=1` on first boot |
| `no_compile` | Skip the `speed` compile of AODService |
| `allow_lock_plugins` | Test switch: re-enable AODService's four lock-screen plugins (the lock clock is expected to vanish again) |

Delete the file and reboot to go back. Example:

```
adb shell "su -c 'touch /data/adb/modules/aodservice_a176b_v2/no_compile'"
```

## Logs

- `/data/local/tmp/aodservice_a176b.log` (and `.log.prev` from the previous boot): what `service.sh` did, the compile result, and which plugins SystemUI can still discover.

  ```
  adb shell "su -c 'cat /data/local/tmp/aodservice_a176b.log'"
  ```
- Compile status of AODService (look for the `aodservice` entry and `status=speed`):

  ```
  adb shell "su -c 'dumpsys package dexopt'" > dexopt.txt
  ```
- To capture an AOD session: clear the log, turn the screen off for about 15 seconds, wake it, then save `logcat`.

On Windows `cmd`, avoid `|` inside nested quotes. Redirect output to a file instead.

## Known issues

These are observed behavior on one phone. Explanations marked *(interpretation)* are not proven.

- **The AOD clock may not show if AOD is switched off.** In an earlier capture where it was missing, SystemUI treated AOD as disabled:

  ```
  AODPreventBurnInMonitor: start skipped by not AODEnabled
  AlwaysOnDisplayEx: startAOD: ... mIsEnabled = false, mReason = AOD OFF
  ```

  The plugin loaded and the clock data was read, but nothing rendered. `aod_mode` was `0` at the time. The module now sets it to `1` once (r7 and later), which is the intended fix. On one boot `aod_mode` and `aod_show_lockscreen_wallpaper` were `0` again before the script ran, and the cause is unknown (a manual change, power saving mode, or something else).
- **The clock does not move for burn-in protection.** Only the charging info moves, about 2×3 px per minute. The clock stayed at the same position in 96 log readings. *(Interpretation: in the fullscreen layout the AOD draws the lock screen clock, which has no self-move.)* No setting changes this, and it does not look fixable from a module.
- **Show always is required.** In **Auto** mode the burn-in monitor logs `start skipped by not AOD_DISPLAY_MODE_SHOW_ALWAYS` and doesn't start.
- **Black gap of about 0.8 s when pressing the power button.** Panel off, DozeService start, AOD plugin load, AOD UI build, panel in doze. About 220 ms of the system part and about 200 ms of the panel switch can't be changed by a module. The r8 compile step shortened plugin loading in one sample (about 0.67 s total). That is a single measurement, not a guarantee.
- **AOD to lock screen: the clock and "now bar" don't animate**, only the wallpaper does. Same in r7 and r8. *(Interpretation: SystemUI behavior.)*
- **"Erase background" stays grayed out** in the AOD wallpaper settings.
- **Reinstalling re-forces `aod_mode=1` once**, because the marker file lives inside the module folder. A planned fix moves it out.
- **`AOD_ITEM=aodversion=7` is a guess.** It was added so AODService creates its settings table. The correct value is unverified.

## Help wanted

If you can test, the most useful information is:

```
adb shell "settings list system" > settings_system.txt
adb shell "settings list secure" > settings_secure.txt
```

plus whether the AOD settings page has an on/off switch, and whether it's on. Logs from a boot where the AOD clock is missing, and from one where it works, are especially helpful. Also useful:

- The `AOD_ITEM` lines from `/system/etc/floating_feature.xml` on a Samsung phone that ships native AOD, to verify the `aodversion=7` guess.
- Ideas for moving the AOD clock for burn-in, or for shortening the black gap on screen-off.

## Licensing

The scripts and documentation written for this module (`service.sh`, `customize.sh`, `uninstall.sh` and this README) are released under the MIT License. See `LICENSE`.

Samsung's AODService APK and the stock `floating_feature.xml` (which this module edits) remain Samsung's software and are **not** covered by that license. This project is not affiliated with Samsung.

## Credits

Made with love by **Liff** <3

- **Liff**: project owner, device testing and logs.
- **Codex**: the original AODService module (`AODService_A176B_v2`) this is based on.
- **Claude** (Anthropic): debugging, module fixes and documentation. Credited as a co-author on commits.

## Disclaimer

This modifies system files and package state on a rooted device. Use it at your own risk. Keep a way to disable the module (KernelSU Manager, or the rollback command above) before you flash.
