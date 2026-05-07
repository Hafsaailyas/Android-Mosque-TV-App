# Domain Layer — MasjidWala TV

## Overview

The domain layer consists of pure Kotlin data classes with no Android framework imports. Each class represents a discrete concept in the mosque display domain. They are constructed by `GenFunc`, persisted by `DbFunc`, and consumed by `MainUI`. The absence of Android dependencies makes these classes independently unit-testable.

---

## Settings

**File:** `Settings.kt`  
**Purpose:** Represents the complete configuration state for one deployed TV screen.

Settings is the most complex domain object. It is the authoritative source for:
- Which prayer rows are visible
- Whether times are manually set or auto-calculated
- Iqamah timing parameters (countdown window, post-Iqamah window, auto-advance interval)
- Audio playback flags
- Sync control flags (server-set, determines which data categories refresh next cycle)
- Background image BLOB

See [Settings Sample Reference](../samples/Settings.MD) for the complete field listing with types, defaults, and constraints.

**Lifecycle:**
```
API Response JSON
    → GenFunc.parseSettingsJSON()
        → Settings (populated)
            → DbFunc.insertSettings()
                → SQLite settings table
                    ← DbFunc.getSettings()
                        → MainUI.init()
```

---

## PerpetualCalendar

**File:** `PerpetualCalendar.kt`  
**Purpose:** Represents one calendar day's complete prayer time schedule, including both Gregorian and Hijri date metadata.

The name "Perpetual Calendar" reflects the design intent: the server delivers a 30-day rolling window, ensuring the device always has upcoming dates pre-cached even without connectivity.

### Fields

| Field | Type | Example | Description |
|---|---|---|---|
| `date` | String | `"2025-04-01"` | ISO date (primary key in SQLite) |
| `city` | String | `"Lahore"` | Associated city |
| `country` | String | `"Pakistan"` | Associated country |
| `timezone` | String | `"Asia/Karachi"` | IANA timezone string |
| `gregorian_day` | Int | `2` | Day of month (numeric) |
| `gregorian_month` | Int | `4` | Month (numeric) |
| `hijri_date` | String | `"02-10-1446"` | Hijri date in `DD-MM-YYYY` |
| `hijri_month_en` | String | `"Shawwal"` | Hijri month in English |
| `hijri_month_ar` | String | `"شوال"` | Hijri month in Arabic script |
| `fajr` | String | `"04:32 AM"` | Fajr prayer time |
| `sunrise` | String | `"05:55 AM"` | Sunrise time |
| `zuhr` | String | `"12:15 PM"` | Zuhr prayer time |
| `asr` | String | `"04:02 PM"` | Asr prayer time |
| `sunset` | String | `"06:35 PM"` | Sunset time |
| `maghrib` | String | `"06:35 PM"` | Maghrib prayer time |
| `isha` | String | `"07:52 PM"` | Isha prayer time |
| `hijri_date_full_en` | String | `"02, Shawwal 1446"` | Formatted Hijri display string |
| `gregorian_date_full` | String | `"Tuesday, 1 April 2025"` | Formatted Gregorian display string |
| `ishraaq` | String | `"06:10 AM"` | Derived Ishraq time (Sunrise + offset) |
| `zawal` | String | `"12:00 PM"` | Derived Zawal time |
| `last_updated` | String | `"2025-04-01"` | Date this record was last fetched |

**Note:** `gregorian_date_full`, `ishraaq`, and `zawal` are computed fields populated by `GenFunc` after the calendar data is received from the API. They are not stored in SQLite; they are recalculated on each `MainUI.init()` call from the stored raw values.

---

## Prayer

**File:** `Prayer.kt`  
**Purpose:** A sub-object attached to each prayer-type `DisplayObject`. Holds the timing boundaries and labels needed by the countdown timer logic.

| Field | Type | Description |
|---|---|---|
| `starting_time` | String | Azan time for this prayer (sourced from `PerpetualCalendar`) |
| `iqamah_time` | String | Iqamah time (manual or calculated, sourced from `Settings`) |
| `ending_time` | String | When this prayer period ends (typically the next prayer's Azan time) |
| `ending_label` | String | Translated label for the end marker (e.g., `"طلوع آفتاب"` for Fajr's end) |
| `next_prayer` | String | Internal identifier of the next prayer slot, or `"no_prayer"` for the last slot |

`Prayer` is used exclusively by the `prayerTimer` in `MainUI` to calculate:
- Time until Iqamah (`iqamah_time - now`)
- Whether the post-Iqamah window is active (`now - iqamah_time < afterQiyamahTimeMin`)
- The ending label to show when the prayer window closes

---

## DisplayObject

**File:** `DisplayObject.kt`  
**Purpose:** The UI binding model. Each row on the main prayer display screen corresponds to exactly one `DisplayObject`. It bridges the domain data and the Android View layer.

| Field | Type | Description |
|---|---|---|
| `id` | Int | Stable ordering index (1=Fajr, 2=Sunrise, 3=Ishraq, ...) |
| `object_name` | String | Internal name key (`"fajr"`, `"zuhr"`, `"asr"`, etc.) |
| `display_label` | String | The translated row label shown in the UI |
| `display_value` | String | The primary value shown (Iqamah time or prayer time) |
| `time_start` | String | Start of this row's active window |
| `time_end` | String | End of this row's active window |
| `object_container_id` | Int | Android `R.id` of the layout view that displays this row |
| `is_display` | Boolean | If `false`, the corresponding view is set to `View.GONE` |
| `is_prayer` | Boolean | `true` for Fajr/Zuhr/Asr/Maghrib/Isha/Jummah; `false` for info rows |
| `is_current_prayer` | Boolean | Set to `true` by the timer when this row's window is currently active |
| `prayer` | Prayer? | Nested `Prayer` object (non-null for prayer rows, `null` for info rows) |

`DisplayObject` is intentionally shallow — it holds references by ID rather than by object, ensuring it can be serialised or passed between components without dragging in Android View references.

---

## Slides

**File:** `Slides.kt`  
**Purpose:** Represents a single media slide in the digital signage carousel.

| Field | Type | Description |
|---|---|---|
| `id` | String | Server-assigned unique ID (timestamp-based) |
| `objectContent` | ByteArray | Raw image bytes (PNG, JPG, GIF) — stored as SQLite BLOB |
| `masjid_id` | Int | Owning masjid's server ID |
| `name` | String | Human-readable label for logging |
| `type` | String | Media type hint: `"png"`, `"jpg"`, `"gif"` |
| `sequence_no` | Int | Ascending display order |
| `second_to_show` | Int | Seconds this slide displays before auto-advancing |
| `full_screen` | Int | `1` = fullscreen layout; `0` = framed layout |
| `status` | Int | `1` = active (display); `0` = inactive (skip) |
| `iqamah_screen` | Int | `1` = only show during post-Iqamah window; `0` = regular rotation |
| `updated_at` | String | Server-side last-modified timestamp |

---

## Translations

**File:** `Tranlsations.kt` *(note: filename contains a typo inherited from the original codebase)*  
**Purpose:** Holds all locale-specific UI label strings. Populated from the `/translation` API endpoint and persisted as a serialised JSON string in `SharedPreferences`.

The translations object decouples all displayed text from the layout XML, enabling full runtime language switching without an Activity restart. It supports both RTL (Urdu/Arabic) and LTR (English) locales.

### Field Categories

| Category | Fields |
|---|---|
| **Masjid Identity** | `masjid_name`, `town_city` |
| **Prayer Labels** | `fajr_label`, `ishraq_label`, `zawal_label`, `zuhr_label`, `juma_label`, `asr_label`, `maghrab_label`, `isha_label` |
| **Prayer Ending Labels** | `fajr_ending_label`, `zuhr_ending_label`, `asr_ending_label`, `maghrab_ending_label`, `isha_ending_label` |
| **Countdown Labels** | `countdown_iqmah_label`, `countdown_next_prayer_label` |
| **Iqamah Nearby Messages** | `message_line1_iqamah_nearby`, `message_line2_iqamah_nearby` |
| **Iqamah Messages** | `message_line1_iqamah`, `message_line2_iqamah` |
| **Jamaat Messages** | `message_line1_jamah`, `message_line2_jamah` |
| **Azan Messages** | `message_line1_azan`, `message_line2_azan` |
| **Login UI** | `mobile_label`, `pic_label`, error message strings |

All fields have Urdu-script default values (reflecting the primary deployment locale), with English translations provided by the server when `lang=en` is requested.
