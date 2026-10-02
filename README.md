# RAM/CPU + Wi-Fi indicators and day-of-week clock

A small Ambxst bar mod, packaged the same way as
[ambxst-mod-i18n](https://github.com/flathead/ambxst-mod-i18n): a single
`ambxst.mod.json` manifest plus one patch against the tested Ambxst base
revision recorded in that manifest.

## What it adds

- **RAM/CPU indicator** (`modules/bar/RamIndicator.qml`) — two small ring
  meters (CPU + RAM) added to the bar, next to the battery indicator.
- **Wi-Fi indicator** (`modules/bar/WifiIndicator.qml`) — a bar button that
  mirrors the existing Wi-Fi/network state, click to rescan.
- **3-stop battery colour ramp** — `BatteryIndicator.qml` now goes
  red (≤15%) → orange (≤45%) → green (≥45%) instead of a red→green
  gradient.
- **Day-of-week in the clock** — `Clock.qml` shows the day abbreviation
  (Mon, Tue, ...) as its own label, with the weather glyph next to it
  instead of replacing it. As of v1.1.0 this also repoints `updateDay()`
  at the existing 3-letter `weather.day.*` translation keys instead of
  the 2-letter `calendar.day.*` keys that Ambxst's native i18n merge
  introduced, so the label reads "Mon" rather than "Mo" (see
  "Why the day label uses `weather.day.*` keys" below).
- **Optional: thermometer-style temperature gauge**
  (`modules/bar/TemperatureIndicator.qml`) is included but **not** wired
  into the bar by this patch, since the source `BarContent.qml` you sent
  didn't instantiate it either. See "Adding the temperature gauge" below
  if you want it too.

## Install

Open **Settings → Mods**, paste this repository's URL into
**Package source**, and select **Install**. Enable the package and
restart Ambxst (or run `ambxst reload`) when prompted.

## Package contents

- `ambxst.mod.json` — manifest.
- `patches/feature.patch` — a single `git diff`-format patch against the
  Ambxst base commit recorded in `compatibility.testedBaseCommits`. It
  touches:
  - `modules/bar/BarContent.qml` — wires `WifiIndicator` and
    `RamIndicator` into the bar layout.
  - `modules/bar/BatteryIndicator.qml` — the 3-stop colour ramp.
  - `modules/bar/clock/Clock.qml` — the day-of-week label.
  - `modules/bar/RamIndicator.qml`, `modules/bar/WifiIndicator.qml`,
    `modules/bar/TemperatureIndicator.qml` — new files.
  - `modules/services/SystemResources.qml` — see "Why SystemResources is
    patched" below.

## Why `SystemResources.qml` is patched

Both `RamIndicator.qml` and `TemperatureIndicator.qml` call
`SystemResources.activeBarWidgets++` / `--` in `Component.onCompleted` /
`onDestruction`, expecting that property to exist and to keep the CPU/RAM
poller alive while a bar widget needs live data. That property didn't
exist in stock Ambxst — polling there is normally gated only by whether
the dashboard's Metrics tab is open. Without this small addition the ring
meters would sit frozen at 0%. The patch adds:

```qml
property int activeBarWidgets: 0
readonly property bool monitoringActive: (GlobalStates.dashboardOpen && GlobalStates.dashboardCurrentTab === 2 && root.validDisks.length > 0) || root.activeBarWidgets > 0
```

so polling turns on automatically whenever one of these bar widgets is
on screen, and turns back off when it isn't, same resource-saving intent
as the existing dashboard-only gate.

## Other fixes made while packaging

`TemperatureIndicator.qml` had two references that don't resolve against
stock Ambxst and were fixed before patching:
- `iconText` was declared as a commented-out line but used in the
  component body — restored as `property string iconText: Icons.temperature`.
- The hover-highlight rectangle referenced `root.popupOpen`, which this
  component never declares (that property exists on `Clock.qml`, not
  here) — replaced with the same `root.isHovered` check `RamIndicator`
  uses.

## Adding the temperature gauge

If you also want the CPU thermometer in the bar, add this alongside the
`RamIndicator` block that the patch inserts into `BarContent.qml`:

```qml
Bar.TemperatureIndicator {
    id: temperatureIndicator
    bar: root
    layerEnabled: root.shadowsEnabled
    startRadius: root.innerRadius
    endRadius: root.innerRadius
}
```

Swap in `value: SystemResources.gpuTemp` (and adjust `label`) for a GPU
reading instead of CPU — see the comment block at the top of
`TemperatureIndicator.qml` for the rest of the configurable properties.

## Why the day label uses `weather.day.*` keys

[#why-the-day-label-uses-weatherday-keys](#why-the-day-label-uses-weatherday-keys)

Ambxst's `updateDay()` builds the bar's day-of-week label from an
`I18n.t()` lookup over a `dayKeys` array. Since Ambxst absorbed
`ambxst-mod-i18n` natively, that array points at `calendar.day.*`,
which is a deliberately short, 2-letter form (`"calendar.day.mon": "Mo"`
in `translations/en.json`) meant for tight calendar-grid cells. The bar
isn't that tight, so this patch swaps `dayKeys` over to the
`weather.day.*` keys the weather popup already uses for its forecast
strip — those are 3 letters in every shipped locale (`"weather.day.mon":
"Mon"`, `"Lun"`, etc.), except Russian, where 2 letters (`"Пн"`) is
already the correct native abbreviation in both key sets, so nothing
changes there. No new translation strings are added; this just points
at ones that already exist.

## Compatibility

Tested against Ambxst commit `2a704c438ddc0b94a11f4e4cf32a16220b62141f`
(post-1.3.0, native mod manager, includes the native-i18n merge that the
day-label fix above depends on). If your installed Ambxst has drifted
far from that revision, the mod manager's compatibility check may flag it
— use `ambxst mods bypass on` to force it through if you've confirmed the
patch still applies, or update `compatibility.testedBaseCommits`
yourself after re-generating the patch against your current tree. If
you're on an older Ambxst that predates the native-i18n merge (before
`calendar.day.*`/`weather.day.*` existed), the old `d6a3b7207` revision
of this mod already gives you a 3-letter day label via a different
mechanism (`Qt.formatDateTime(..., "ddd")`), so you don't need this
patch hunk at all in that case.

## License

GPL-3.0-only — see `LICENSE`.
