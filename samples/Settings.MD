# Settings — Domain Model Reference

## Overview

The `Settings` class is the central configuration object for a deployed MasjidWala TV screen. It is populated from the `/screen_settings` API endpoint and persisted in the `settings` SQLite table (single-row). All fields have safe default values so the app renders meaningfully even before the first successful sync.

---

## Field Reference

### Masjid Identity

| Field | Type | Default | Description |
|---|---|---|---|
| `masjidName` | `String` | `"Executive Cottages"` | Display name of the mosque, shown in the UI header |
| `townName` | `String` | `"Paragon City, Lahore"` | Town or locality label shown below the masjid name |
| `address` | `String` | `"Executive Cottages, Paragon City, Lahore, Pakistan"` | Full address (used internally; not typically displayed on screen) |
| `city` | `String` | `"Lahore"` | City used as the location parameter for calendar API queries |
| `country` | `String` | `"Pakistan"` | Country used as the location parameter for calendar API queries |

---

### Prayer Calculation

| Field | Type | Default | Description |
|---|---|---|---|
| `maslak` | `Int` | `1` | Juristic school for prayer time calculation. `1` = Hanafi (Asr at double shadow), `0` = Shafi (Asr at single shadow) |
| `manualTime` | `Boolean` | `true` | If `true`, Iqamah times are taken directly from the `*Qiyamah` fields. If `false`, they are calculated automatically by adding `iqamahCountDownIntervalMin` to the Azan time |

---

### Prayer Row Visibility

Each boolean below controls whether the corresponding row is rendered on the main display. Setting a value to `false` hides the row entirely.

| Field | Type | Default |
|---|---|---|
| `showFajr` | `Boolean` | `true` |
| `showIshraq` | `Boolean` | `true` |
| `showZawal` | `Boolean` | `true` |
| `showZuhr` | `Boolean` | `true` |
| `showAsr` | `Boolean` | `true` |
| `showMaghrib` | `Boolean` | `false` |
| `showIsha` | `Boolean` | `true` |
| `showJummah` | `Boolean` | `true` |
| `showSunrise` | `Boolean` | `true` |
| `showSunset` | `Boolean` | `true` |
| `showDakhul` | `Boolean` | `true` |

---

### Manual Iqamah Times

Used when `manualTime == true`. All values are in 12-hour `HH:MM AM/PM` format.

| Field | Type | Default | Description |
|---|---|---|---|
| `dateQiyamah` | `String` | `"2000-01-01"` | Reference date for the manual Iqamah schedule (`YYYY-MM-DD`) |
| `fajrQiyamah` | `String` | `"00:00 AM"` | Fajr Iqamah time |
| `zuhrQiyamah` | `String` | `"00:00 PM"` | Zuhr Iqamah time |
| `asrQiyamah` | `String` | `"00:00 PM"` | Asr Iqamah time |
| `maghribQiyamah` | `String` | `"00:00 PM"` | Maghrib Iqamah time |
| `ishaQiyamah` | `String` | `"00:00 PM"` | Isha Iqamah time |
| `jummahQiyamah` | `String` | `"00:00 PM"` | Jummah Iqamah time (Fridays only) |

---

### Manual Time Overrides

Used to override calculated times for auxiliary display rows when `manualTime == true`.

| Field | Type | Default | Description |
|---|---|---|---|
| `ishraqTime` | `String` | `"00:00 PM"` | Manual Ishraq display time |
| `zawalTime` | `String` | `"00:00 PM"` | Manual Zawal display time |
| `sunriseTime` | `String` | `"00:00 PM"` | Manual Sunrise display time |
| `sunsetTime` | `String` | `"00:00 PM"` | Manual Sunset display time |

---

### Iqamah Countdown Behaviour

| Field | Type | Default | Description |
|---|---|---|---|
| `qiyamahCountDown` | `Boolean` | `true` | Enables the Iqamah countdown timer feature |
| `qiyamahCountMin` | `Int` | `20` | Minutes before Iqamah when the countdown begins showing |
| `afterQiyamahTimeMin` | `Int` | `10` | Minutes after Iqamah during which the Jamaat screen remains active |
| `startCountDownAfterSec` | `Int` | `120` | Seconds after which the countdown overlay auto-starts |
| `jammatMessageAfterIqamahSec` | `Int` | `30` | Seconds for which the "صف سیدھی کریں" message displays |
| `iqamahCountDownIntervalMin` | `Int` | `15` | Auto-calculated Iqamah offset from Azan (used when `manualTime == false`) |
| `jammatIntervalBreakMin` | `Int` | `10` | Break interval between Jamaat session end and next cycle |
| `slidesStartAfterIqamahMin` | `Int` | `10` | Minutes after Iqamah before the regular slideshow resumes |

---

### Audio

| Field | Type | Default | Description |
|---|---|---|---|
| `playAzan` | `Boolean` | `true` | Whether to play the Azan audio cue at prayer time |
| `playIqamah` | `Boolean` | `true` | Whether to play the Iqamah audio cue before Iqamah |
| `azanBeforeIqamahMin` | `Int` | `15` | Minutes before Iqamah when the Azan audio is triggered |

---

### Background Image

| Field | Type | Default | Description |
|---|---|---|---|
| `showBackgroundImage` | `Boolean` | `true` | Whether a fullscreen background image is shown behind the prayer times grid |
| `backgroundImageContents` | `ByteArray` | `byteArrayOf()` | Raw bytes of the background image (PNG/JPG). Stored as a BLOB in SQLite |

---

### Sync Control Flags

These fields are populated from the `/sync` handshake response and persisted alongside settings. They direct which data categories are refreshed on the next sync cycle.

| Field | Type | Default | Description |
|---|---|---|---|
| `hijriDate` | `String` | `"2000-01-01"` | Server-override for the displayed Hijri date |
| `nextRequestRunIn` | `Int` | `10` | Minutes until the next sync cycle (server-controlled) |
| `runCalendarReq` | `Boolean` | `true` | Force calendar re-fetch on next sync |
| `runSettingsReq` | `Boolean` | `true` | Force settings re-fetch on next sync |
| `runSlideShowReq` | `Boolean` | `true` | Force slides re-fetch on next sync |
| `runLanguageReq` | `Boolean` | `true` | Force translation re-fetch on next sync |

---

## Constraints & Notes

- All `String` time fields must conform to the format `"HH:MM AM"` or `"HH:MM PM"` (12-hour, uppercase AM/PM, space-separated). Deviation causes a `DateTimeParseException` in `GenFunc`, which falls back to `LocalTime.MIDNIGHT`.
- `maslak` values outside `{0, 1}` are treated as `1` (Hanafi) by default.
- `backgroundImageContents` is not stored in the `settings` SQLite table in the current schema version (the column `bg_image_content` is defined but write is commented out pending a storage optimisation review). It is delivered via the slides mechanism instead.
- Default values are chosen to be "safe to display" — the app will show sensible placeholder times and labels on a fresh install before any sync completes.
