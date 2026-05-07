# Database Schema — MasjidWala TV

## Overview

The application uses a single SQLite database (`masjid.wala`) managed via a subclass of `SQLiteOpenHelper`. Three tables are created on first install. All writes go through `DbFunc` which provides a typed CRUD interface; direct database access outside `DbFunc` is not permitted.

**Database name:** `masjid.wala`  
**Current schema version:** `1`

---

## Table: `settings`

Stores all masjid configuration. This table is treated as a single-row store — all writes first delete the existing row, then insert a fresh one. The primary key is always `1`.

| Column | Type | Constraint | Description |
|---|---|---|---|
| `primary_key` | INTEGER | PRIMARY KEY | Always 1 (single-row table) |
| `masjid_name` | STRING | | Display name of the mosque |
| `town_name` | STRING | | Town or locality name |
| `address` | STRING | | Full street address |
| `city` | STRING | | City (used for calendar queries) |
| `country` | STRING | | Country (used for calendar queries) |
| `maslak` | INTEGER | | Prayer calc method: `1` = Hanafi, `0` = Shafi |
| `manual_time` | BOOLEAN | | If `true`, Iqamah times are manually set; if `false`, calculated automatically |
| `show_ishraq` | BOOLEAN | | Whether to display the Ishraq prayer row |
| `show_zawal` | BOOLEAN | | Whether to display the Zawal time row |
| `show_fajr` | BOOLEAN | | Whether to display the Fajr prayer row |
| `show_zuhr` | BOOLEAN | | Whether to display the Zuhr prayer row |
| `show_asr` | BOOLEAN | | Whether to display the Asr prayer row |
| `show_maghrib` | BOOLEAN | | Whether to display the Maghrib prayer row |
| `show_isha` | BOOLEAN | | Whether to display the Isha prayer row |
| `show_jummah` | BOOLEAN | | Whether to display the Jummah row on Fridays |
| `qiyamah_count_down` | BOOLEAN | | Whether the Iqamah countdown timer is enabled |
| `date_qiyamah` | STRING | | Date used as reference for Iqamah time calculations (`YYYY-MM-DD`) |
| `fajr_qiyamah` | STRING | | Manual Fajr Iqamah time (`HH:MM AM/PM`) |
| `zuhr_qiyamah` | STRING | | Manual Zuhr Iqamah time |
| `asr_qiyamah` | STRING | | Manual Asr Iqamah time |
| `maghrib_qiyamah` | STRING | | Manual Maghrib Iqamah time |
| `isha_qiyamah` | STRING | | Manual Isha Iqamah time |
| `jummah_qiyamah` | STRING | | Manual Jummah Iqamah time (Fridays only) |
| `ishraq_time` | STRING | | Manual override for Ishraq display time |
| `zawal_time` | STRING | | Manual override for Zawal display time |
| `sunrise_time` | STRING | | Manual override for Sunrise display time |
| `sunset_time` | STRING | | Manual override for Sunset display time |
| `qiyamah_count_min` | INTEGER | | Minutes before Iqamah when countdown begins (default: `20`) |
| `hijri_date` | STRING | | Hijri date override string (used when API correction needed) |
| `next_request_run_in` | INTEGER | | Sync interval in minutes (server-controlled) |
| `run_calendar_req` | BOOLEAN | | Server flag: force-refresh the calendar on next sync |
| `run_settings_req` | BOOLEAN | | Server flag: force-refresh settings on next sync |
| `run_slideShow_req` | BOOLEAN | | Server flag: force-refresh slides on next sync |
| `play_azan` | BOOLEAN | | Whether to play the Azan audio cue |
| `play_iqamah` | BOOLEAN | | Whether to play the Iqamah audio cue |
| `show_bg_image` | BOOLEAN | | Whether to show a fullscreen background image |
| `bg_image_content` | BLOB | | Raw bytes of the background image |
| `after_qiyamah_time_min` | INTEGER | | Minutes after Iqamah during which the Iqamah screen remains active |

---

## Table: `namaz_times`

Stores the 30-day prayer time calendar. Each row represents one calendar day. The primary key is the ISO date string.

| Column | Type | Constraint | Description |
|---|---|---|---|
| `date` | STRING | PRIMARY KEY | ISO date `YYYY-MM-DD` — uniquely identifies the day |
| `city` | STRING | | City associated with these prayer times |
| `country` | STRING | | Country associated with these prayer times |
| `timezone` | STRING | | IANA timezone string (e.g., `Asia/Karachi`) |
| `gregorian_day` | STRING | | Day of week in English (e.g., `Tuesday`) |
| `gregorian_month` | STRING | | Month name in English (e.g., `April`) |
| `hijri_date` | STRING | | Hijri date in `DD-MM-YYYY` format |
| `hijri_month_en` | STRING | | Hijri month name in English (e.g., `Shawwal`) |
| `hijri_month_ar` | STRING | | Hijri month name in Arabic script |
| `fajr` | STRING | | Fajr prayer time (`HH:MM AM`) |
| `sunrise` | STRING | | Sunrise time (`HH:MM AM`) |
| `zuhr` | STRING | | Zuhr prayer time (`HH:MM PM`) |
| `asr` | STRING | | Asr prayer time (`HH:MM PM`) |
| `sunset` | STRING | | Sunset time (`HH:MM PM`) |
| `maghrib` | STRING | | Maghrib prayer time (`HH:MM PM`) |
| `isha` | STRING | | Isha prayer time (`HH:MM PM`) |
| `last_updated` | STRING | | Timestamp of last API fetch (`YYYY-MM-DD`) |
| `hijri_date_full_en` | STRING | | Full formatted Hijri date (e.g., `02, Shawwal 1446`) |

---

## Table: `slides`

Stores media slide content as binary BLOBs alongside metadata. Slides are displayed in ascending `sequence_no` order.

| Column | Type | Constraint | Description |
|---|---|---|---|
| `id` | STRING | PRIMARY KEY | Unique slide ID (timestamp-based string from the server) |
| `objectContent` | BLOB | | Raw image bytes (PNG, JPG, or GIF) |
| `masjid_id` | INTEGER | | Foreign reference to the owning masjid |
| `name` | STRING | | Human-readable slide name for logging/debugging |
| `type` | STRING | | Media type hint (`png`, `jpg`, `gif`) |
| `file_path` | STRING | | Optional external file path (not used when BLOB is present) |
| `sequence_no` | INTEGER | | Display order (ascending); determines carousel sequence |
| `second_to_show` | INTEGER | | Duration in seconds this slide is shown before advancing |
| `full_screen` | INTEGER | | `1` = fullscreen display; `0` = standard frame |
| `status` | INTEGER | | `1` = active (display); `0` = inactive (skip) |
| `iqamah_screen` | INTEGER | | `1` = only show during Iqamah window; `0` = always show |
| `updated_at` | STRING | | Last update timestamp from the server (`YYYY-MM-DD HH:MM:SS`) |

---

## Schema Upgrade Policy

Database version upgrades are handled in `onUpgrade()`. The current pattern executes version-specific migration SQL strings. For minor additions (new columns), `ALTER TABLE` statements are used. Destructive migrations (drop and recreate) are avoided; data preservation is the priority. Each migration block is gated by an `oldVersion` check to ensure idempotency.

---

## Index Recommendations

For future performance at scale, the following indices are recommended:

```sql
-- Optimise today's prayer time lookup (called on every UI refresh)
CREATE INDEX IF NOT EXISTS idx_namaz_date ON namaz_times (date);

-- Optimise slide ordering queries
CREATE INDEX IF NOT EXISTS idx_slides_sequence ON slides (sequence_no, status);
```

These are not created in the current version (v1) due to the expected small dataset size (≤ 30 calendar rows, ≤ 20 slides).
