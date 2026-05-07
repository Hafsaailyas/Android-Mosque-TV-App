# Content Sync Strategy — MasjidWala TV

## Overview

MasjidWala TV maintains a continuous synchronisation loop between the device and the backend API. The strategy is designed to minimise unnecessary data transfer while guaranteeing that the display is never more than one sync cycle out of date.

---

## Sync Trigger Points

Content synchronisation is initiated at three points:

| Trigger | Description |
|---|---|
| **App launch / post-login** | The first sync runs immediately after `MainActivity` confirms valid credentials |
| **Periodic timer** | A `fixedRateTimer` fires every `nextRunIn` seconds (received from the server on each sync) |
| **Boot completion** | `BootReceiver` → `AppStartService` → `LoginActivity` → `MainActivity` re-runs the full sync chain |

---

## Phase 1: Handshake / Sync Control

Every sync cycle begins with a call to the `/sync` endpoint. The server responds with a lightweight JSON payload (no media content) containing boolean flags and the next-run interval.

```
GET /sync?masjid_id=<device_id>
Authorization: Bearer <token>

Response:
{
  "run_settings_req":   true|false,
  "run_calendar_req":   true|false,
  "run_slideshow_req":  true|false,
  "run_language_req":   true|false,
  "next_request_run_in": <integer minutes>
}
```

This handshake response acts as a **remote kill switch** for each data category. The administrator can force any device to refresh specific data on the next cycle without requiring a full resync of everything.

---

## Phase 2: Conditional Sync Logic

After the handshake, `GenFunc.syncRequest()` evaluates four independent conditions. Each condition follows the pattern: **"force flag from server OR data not present locally"**.

```
Pseudocode:

if (server_flag.run_settings_req == true  OR  local_db.settingsExist() == false)
    syncSettings()

if (server_flag.run_language_req == true)
    syncTranslations()

if (server_flag.run_calendar_req == true  OR  local_db.prayerTimesExist() == false)
    syncCalendar()

if (server_flag.run_slideshow_req == true  OR  local_db.slidesExist() == false)
    syncSlides()
```

This means on a fresh install (empty database), all four data fetches run regardless of the server flags. On subsequent syncs, only the categories flagged by the server are refreshed.

---

## Phase 3: Per-Category Merge Logic

### Settings Merge

Settings are treated as a **full replacement**, not a merge. The insert routine calls `deleteAllSettings()` before writing new values. This ensures no stale fields survive a server update.

**Special case — Hijri Date:**  
The `hijri_date` field in `settings` is used as a client-side correction override. When the server provides a corrected Hijri date (e.g., for moon sighting announcements), this value takes priority over the date derived from the `namaz_times` calendar.

**Priority order for displayed Hijri date:**
```
1. settings.hijriDate  (server override, if non-empty and non-default)
2. namaz_times[today].hijri_date  (calculated calendar value)
3. Hardcoded default string  (fallback if both are missing)
```

### Calendar Merge

Calendar records are **upserted by date** (insert-or-replace using the `date` primary key). This means partial syncs (e.g., receiving only the next 7 days) do not delete the rest of the month. Rows for past dates are retained until a full calendar refresh is received.

### Slides Merge

The slides sync performs a **full delete-and-replace** of the `slides` table. This ensures that slides removed from the admin panel are also removed from the device on the next cycle — no orphaned content persists.

### Translations Merge

Translation strings are stored in `SharedPreferences` as a serialised JSON string (not in SQLite). They are also treated as a full replacement — the key `TRANSLATIONS_OBJ` is overwritten entirely.

---

## Sync Interval Control

The `next_request_run_in` value from the server controls the timer period. It is expressed in minutes and converted to seconds internally:

```
nextRunInSeconds = next_request_run_in * 60
```

The default value when the API is unreachable is **10 minutes** (`600 seconds`). The server can set this as low as 1 minute for urgent updates or as high as 60 minutes for stable deployments.

---

## Fallback Behaviour (Offline Mode)

If any API call fails (network timeout, server error, parse exception), the following fallback chain applies:

```
API Call Fails
      │
      ▼
Catch Exception / VolleyError
      │
      ▼
Use DbFunc to read last-known value from SQLite
      │
      ├── Settings      → DbFunc.getSettings()         → Settings (last stored)
      ├── Prayer Times  → DbFunc.getPerpetualCalendar() → PerpetualCalendar (today's row)
      ├── Slides        → DbFunc.getSlides()            → List<Slides> (last stored)
      └── Translations  → AuthPreferences.getTranslations() → serialised JSON string
      │
      ▼
UI renders with cached data
(timer continues; next sync attempt runs at next interval)
```

**Key guarantee:** The display never goes blank due to a network failure. Even on extended offline periods (days), the `namaz_times` table covers the pre-fetched 30-day window.

---

## Sync Execution Context

All sync network calls are executed as Kotlin coroutines (`suspend` functions) within a `withContext(Dispatchers.IO)` scope. This prevents any blocking on the Android main thread. The `MainActivity` calls `syncRequest()` from a coroutine scope tied to the Activity lifecycle, ensuring cancellation on `onDestroy()`.

---

## Sequence Summary

```
Launch / Timer Fire
        │
        ▼
   /sync (handshake)
        │
   ┌────┴──────────────────┐
   │  run_settings_req?    │── YES ──► /screen_settings ──► DB insert
   └───────────────────────┘
   ┌────┴──────────────────┐
   │  run_language_req?    │── YES ──► /translation ──► SharedPreferences
   └───────────────────────┘
   ┌────┴──────────────────┐
   │  run_calendar_req?    │── YES ──► /calendar ──► DB upsert (by date)
   └───────────────────────┘
   ┌────┴──────────────────┐
   │  run_slideshow_req?   │── YES ──► /slideshow ──► DB delete + insert
   └───────────────────────┘
        │
        ▼
  return nextRunIn (seconds)
        │
        ▼
  MainUI.init() ──► UI refresh
        │
        ▼
  schedule next sync in nextRunIn seconds
```
