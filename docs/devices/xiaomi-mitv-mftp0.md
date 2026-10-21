# Xiaomi MiTV-MFTP0: HDMI source switching

Tested on MiTV-MFTP0 (`alita`, MediaTek, 32-bit `armeabi-v7a`), Android 9 / API 28,
firmware `23.7.11.6589`. Stock Home is `com.mitv.tvhome/.MainActivityUserMode`.
Launcher-only record: no debloat list was applied and nothing was disabled.

Measured on a real TV on 2026-10-21 through Wi-Fi ADB (`192.168.11.88:5555`), in one session.

## The standard channel URI does not switch anything here

`ACTION_VIEW` on `content://android.media.tv/passthrough/<input id>` with MIME
`vnd.android.cursor.item/channel` does **not** raise `ActivityNotFoundException` on this box.
Four third-party players register a filter broad enough to answer it:

```
com.fongmi.android.tv        com.leospring.flutter_ant_video
com.player.bear              com.xunlei.downloadprovider.hd
```

`pm resolve-activity` returns the framework `ResolverActivity` (several matches), and one launch
put `com.clouddrive2/…ExternalPlayerActivity` in the foreground instead of a source panel. So the
standard path "succeeds" and shows an unrelated app. Any fallback that only runs after
`ActivityNotFoundException` is dead code on this firmware: the vendor activity has to be tried
first.

`tv_input` itself is healthy — `com.mediatek.tvinput` registers the services and a session exists
for the current source. The problem is purely which activity the implicit intent resolves to.

## Port table (measured by sweep)

`com.xiaomi.mitv.tvplayer/.ExternalSourceActivity` (tvplayer 1.3.1, versionCode 171102,
`/vendor/app/MiTVTVPlayer`) answers `com.xiaomi.mitv.tvplayer.EXTSRC_PLAY` with an INTEGER extra
`input`. `query-activities` reports `exported=true`, so a normal app may start it explicitly; every
start in the sweep came from the shell uid, so the app-side start still has to be confirmed by
reading the `START … from uid` line in logcat on the next install.

| Label  | TvInputInfo ID (HW)                       | `input` extra |
|--------|-------------------------------------------|---------------|
| HDMI1  | `com.mediatek.tvinput/.hdmi.HDMIInputService/HW5` | 23 |
| HDMI2  | `com.mediatek.tvinput/.hdmi.HDMIInputService/HW6` | 24 |
| HDMI3  | `com.mediatek.tvinput/.hdmi.HDMIInputService/HW7` | 25 |
| AV     | `com.mediatek.tvinput/.composite.CompositeInputService/HW2` | 2 |
| ATV    | `com.mediatek.tvinput/.tuner.TunerInputService/HW1` | 1 |
| DTMB   | `com.mediatek.tvinput/.tuner.TunerInputService/HW0` | 28 |

Every value below 1, 2, 23, 24, 25, 28 and outside them was rejected: **HW8 is not reachable**
through this activity, so the fourth `hdmi_port=4` entry in `TvInputHardwareManager` is not an
externally switchable source. Same conclusion as `xiaomi-mitv-mftr0-freeguy.txt`: HW8 must not be
read as evidence of a fourth physical connector. VGA (HW4) and component (HW3) are registered in
the framework but have no accepted extra value either.

`EXTSRC_PLAY` with **no** extra leaves no session at all, and a rejected value does not throw — the
activity opens and shows its own default source. Both cases look like "switched to the wrong input"
to the user, which is why an unmapped port must never reach the activity.

## How the table was read

`settings system current_source_for_tvplayer` holds the source the activity accepted, which makes
the sweep observable without watching the screen:

```sh
adb shell am force-stop com.xiaomi.mitv.tvplayer
adb shell am start -n com.xiaomi.mitv.tvplayer/.ExternalSourceActivity --ei input 24
adb shell settings get system current_source_for_tvplayer   # -> 24 accepted, 34 = rejected
adb shell dumpsys tv_input | sed -n '/sessionStateMap/,/callbackSet/p'   # -> inputId: …/HW6
```

`34` is the rejection sentinel, not a source. A value that is accepted always pairs with a session
for the HW listed above; check both rather than the settings key alone.

## Home identities

Both `com.gathin.tvhome` and `com.ts.launcher` are installed on this TV and both appear in the
HOME query, alongside `com.mitv.tvhome`, `com.huanxi.tvhome`, `com.example.xiaoyitvlauncher`,
`cn.qianx.tvdesk` and `com.xinjing.hotel.home`. The `com.ts.launcher` identity is needed here for
the same reason as on MiTV-AETP0 (see `xiaomi-mitv-aetp0-dofus.md`): Xiaomi's `chooseBestActivity`
keeps only system HOME candidates or packages on `MitvSettings.isThirdpartyHomeAllowed`.
