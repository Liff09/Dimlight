# Dimlight

**Stock AOD for the Galaxy A17 5G: a KernelSU module that brings Samsung's native Always On Display to the SM-A176B on One UI 9.**

Dimlight overlays Samsung's own AODService on the stock system and fixes the conflicts that causes, so you get the original AOD with a basic clock and no third-party fullscreen AOD app.

> **Status: work in progress.** The lock screen clock and the AOD settings page work. The AOD clock sometimes doesn't show, and sometimes needs a soft restart to appear (see [Known issues](#known-issues)).

---

## Tested on

| | |
|---|---|
| Device | Galaxy A17 5G (SM-A176B) |
| Android / UI | Android 17, One UI 9 |
| Firmware | `A176BXXU8DZJ5` |
| Root | KernelSU with the `meta-overlayfsx` metamodule |

Other Samsung phone models are untested, and the outcome is unexpected. Proceed with caution. Note that the installer's model check (in `customize.sh`) currently refuses anything that isn't `SM-A176*`, so you would have to edit it out first.

## What it does

1. **Adds Samsung's AODService** (version 8.9.66.1) as a system_ext priv-app, with its permission XMLs.
2. **Fixes the missing lock screen clock.** Stock firmware already includes `com.samsung.android.app.clockpack`, which registers four SystemUI plugins. AODService registers the same four with identical class names, so SystemUI sees duplicates and drops the clock. `service.sh` disables AODService's four copies after boot:
   - `PluginKeyguardStatusViewImpl`
   - `UfClockProvider`
   - `LockItemPluginService`
   - `PluginViewCoverImpl`

   The AOD plugin (`PluginAODImpl`) stays enabled.
3. **Fixes the AOD settings crash.** AODService's database code adds its settings table only when the `CLOCKPACK_ITEM` floating feature does not contain `clockpackversion`. On this firmware it does, so the table was skipped and the app crashed on start. The module ships its own copy of `/system/etc/floating_feature.xml` (your stock file plus two edits): it adds an `AOD_ITEM` line and removes the `CLOCKPACK_ITEM` line.

## Requirements

- Galaxy A17 5G (SM-A176B), rooted with KernelSU
- A metamodule that mounts module files (tested with `meta-overlayfsx`)
- adb (optional, for logs and rollback)

## Download

The completed, ready-to-flash module zip (`DimLight_V1.zip`, release **v1.0 beta**) is in the [Releases](../../releases) section. There's nothing to build. It already includes Samsung's AODService 8.9.66.1 APK, the permission XMLs, `service.sh`, and the modified `floating_feature.xml`.

## Installing

1. Flash the zip in KernelSU Manager.
2. Do a **full reboot** (not a soft reboot), so the module files get mounted.
3. Make sure no older AODService module is installed alongside this one. The module id is unchanged between revisions, so a new flash updates in place.

## Rolling back

Disable the module in KernelSU Manager and reboot, or from a shell:

```
adb shell "su -c 'touch /data/adb/modules/aodservice_a176b_v2/disable'"
```

Then do a full reboot. This turns off all of the module's changes.

## Logs

- `/data/local/tmp/aodservice_a176b.log` (and `.log.prev` from the previous boot): what `service.sh` did, and which plugins SystemUI can still discover.
- To capture an AOD session: clear the log, turn the screen off for about 15 seconds, wake it, then save `logcat`.

## Test switch

To re-enable AODService's four lock-screen plugins (the lock clock is expected to vanish again), create this file and reboot:

```
/data/adb/modules/aodservice_a176b_v2/allow_lock_plugins
```

Delete the file and reboot to go back.

## Known issues

- **The AOD clock sometimes doesn't show, and sometimes needs a soft restart to appear.** In a captured session where it was missing, SystemUI treated AOD as switched off. The log showed:

  ```
  AODPreventBurnInMonitor: start skipped by not AODEnabled
  AlwaysOnDisplayEx: startAOD: ... mIsEnabled = false, mReason = AOD OFF
  ```

  The plugin loads and the clock data is read (`apply lock clock type 2`), but nothing renders while AOD is considered disabled. The cause is still being worked out. Likely candidates are a timing issue at boot, the AOD enable setting SystemUI reads, or the `AOD_ITEM` floating-feature value.
- **A small 3x1 widget shows** while AOD is enabled. Its source isn't identified yet. It may be a FaceWidget or complication widget.
- **`AOD_ITEM=aodversion=7` is a guess.** It was added so AODService creates its settings table. The correct value is unverified.
- The baseline settings before testing were `aod_mode=0`, `doze_always_on=0` and `ambient_enabled=1`.

## Help wanted

If you can test, the most useful information is:

```
adb shell "settings list system | grep -i aod"
adb shell "settings list secure | grep -i -E 'aod|doze'"
```

plus whether the AOD settings page has an on/off switch, and whether it's on. Logs from a boot where the AOD clock is missing, and from one where it works, are especially helpful.

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
