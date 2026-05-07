# Presentation Layer — MasjidWala TV

## Overview

The presentation layer is responsible for the complete user-facing experience: device registration, main prayer time display, media carousel, and all real-time timer-driven UI updates. It is composed of two Activities (`LoginActivity`, `MainActivity`), a UI singleton (`MainUI`), and an orchestration class (`GenFunc`) that bridges the domain and data layers with the UI.

---

## Boot-to-Display Lifecycle

```
DEVICE POWER ON
      │
      ▼
BootReceiver.onReceive(BOOT_COMPLETED)
   Supported intents:
   - android.intent.action.BOOT_COMPLETED
   - android.intent.action.QUICKBOOT_POWERON
   - com.htc.intent.action.QUICKBOOT_POWERON
      │
      ▼
ContextCompat.startForegroundService(AppStartService)
      │
      ▼  [5-second delay, up to 3 retries on failure]
AppStartService.tryLaunchLoginActivity()
   Flags: FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_SINGLE_TOP
      │
      ▼
LoginActivity.onCreate()
   ├── AuthPreferences.hasPreferences()?
   │       YES → startActivity(MainActivity) — skip login
   │       NO  → show OTP login UI
      │
      ▼  [on successful login]
MainActivity.onCreate()
```

**Resilience:** If `AppStartService` fails to launch `LoginActivity` after 3 retries, it posts a tap-to-open fallback notification. The service declares itself as `START_NOT_STICKY` so it is not auto-restarted by the OS if killed — avoiding retry loops on crash.

---

## LoginActivity

Manages device registration and session persistence.

### Flow

```
1. Display mobile number input field
2. User submits mobile number
   → GenFunc: POST /otp  [NO_AUTH]
   → Server sends OTP to mobile

3. Display OTP input field
4. User submits OTP
   → GenFunc: POST /login  [BASIC_AUTH: mobile:otp]
   → Server responds: { token, masjid_id, layout_direction }

5. AuthPreferences.savePreference(token, masjidId, layoutDir)
6. startActivity(MainActivity)
   finish()  // remove LoginActivity from back stack
```

### Error Handling

All API errors are mapped to translated error strings from the `Translations` object:

| Condition | Displayed Message |
|---|---|
| Invalid mobile format | `translations.error_wrong_mobile` |
| Wrong or expired OTP | `translations.error_wrong_pin` |
| Mobile not associated with any masjid | `translations.error_no_masjid` |
| Device not authorised to register a screen | `translations.error_no_screen` |
| Any other API error | `translations.error_unknown` |

---

## MainActivity

The primary activity. Orchestrates the complete display lifecycle after login.

### Responsibilities

1. **Credential check** — On `onCreate()`, verifies `AuthPreferences.hasPreferences()`. If false (session expired or cleared), redirects to `LoginActivity`.
2. **Layout direction** — Applies RTL or LTR to the root layout based on `AuthPreferences.getLayoutDirection()`.
3. **Translation loading** — Deserialises the stored `Translations` JSON from `AuthPreferences` and exposes it as `companion object translationsObj` for access by `MainUI`.
4. **Sync orchestration** — Calls `GenFunc.syncRequest()` in a coroutine scope. On completion, calls `MainUI.init(context)` to rebuild the UI.
5. **Sync timer** — Schedules a `fixedRateTimer` to repeat `syncRequest()` at `nextRunIn` second intervals.
6. **Prayer timer** — Starts the 1-second `prayerTimer` in `MainUI` after initial UI build.
7. **Slideshow** — Initialises `CarouselAdapter` and attaches it to `ViewPager2`. Starts the auto-advance scheduler.
8. **Lifecycle cleanup** — On `onDestroy()`, cancels all timers and calls `MainUI.destroy()`.

### Key Companion Object Properties

```kotlin
// Pseudocode — structural reference only

companion object {
    var translationsObj: Translations = Translations()   // Default Urdu strings
    var globalDeviceID: String = ""                      // Resolved masjid/device ID
    var layoutDirection: String = "ltr"                  // "ltr" or "rtl"
}
```

These are declared in `companion object` to allow access from `MainUI` and `GenFunc` without passing `Activity` references (which would create lifecycle coupling).

### Sync Timer Pattern

```kotlin
// Pseudocode — structural reference only

var syncTimer: Timer? = null

fun startSyncTimer(intervalSeconds: Int) {
    syncTimer?.cancel()
    syncTimer = fixedRateTimer(
        name = "sync-timer",
        initialDelay = intervalSeconds * 1000L,
        period = intervalSeconds * 1000L
    ) {
        CoroutineScope(Dispatchers.IO).launch {
            val nextRun = genFunc.syncRequest()
            withContext(Dispatchers.Main) {
                MainUI.destroy()
                MainUI.getInstance().init(context)
                startSyncTimer(nextRun)  // reschedule with server-provided interval
            }
        }
    }
}
```

---

## MainUI (Singleton)

Manages the complete prayer time display state. Implemented as a thread-safe singleton using double-checked locking.

### Initialisation

`MainUI.init(context)` is called after every sync and builds the full `List<DisplayObject>` from scratch:

```
1. Read Settings from DbFunc
2. Read today's PerpetualCalendar from DbFunc
3. Resolve Translations from MainActivity.translationsObj
4. For each configured prayer/info row:
   a. Create DisplayObject
   b. Set display_label from Translations
   c. Set is_display from Settings visibility flags
   d. Set time_start / time_end from PerpetualCalendar
   e. Compute display_value via GenFunc.getDisplayValue()
   f. Construct nested Prayer object (for prayer rows)
5. Store completed list in objDisplayObjects
6. Bind each DisplayObject to its TextView via object_container_id
7. Start prayerTimer
```

### Display Row Order

| ID | object_name | is_prayer | Visibility Flag |
|---|---|---|---|
| 1 | `fajr` | true | `settings.showFajr` |
| 2 | `sunrise` | false | `settings.showSunrise` |
| 3 | `ishraq` | false | `settings.showIshraq` |
| 4 | `zawal` | false | `settings.showZawal` |
| 5 | `zuhr` / `jummah` | true | `settings.showZuhr` / `showJummah` |
| 6 | `asr` | true | `settings.showAsr` |
| 7 | `sunset` | false | `settings.showSunset` |
| 8 | `maghrib` | true | `settings.showMaghrib` |
| 9 | `isha` | true | `settings.showIsha` |

### Prayer Timer

A Kotlin `fixedRateTimer` with a 1-second period drives all live updates:

```
Every 1 second:
  1. Evaluate is_current_prayer for each DisplayObject
  2. Apply/remove visual highlight on the current prayer row
  3. Determine countdown mode (Next Prayer / Iqamah / Post-Iqamah)
  4. Update countdown label and value TextViews
  5. Update message overlay (show/hide and set text)
  6. Refresh Gregorian and Hijri date displays
  7. Check for Iqamah window transitions (triggers slideshow swap)

All UI mutations dispatched via Handler(Looper.getMainLooper()).post { }
```

---

## GenFunc

The application's general-purpose orchestration and utility class. It is instantiated with a `Context` and provides two categories of functionality:

### Category 1 — Sync & Network

All methods are `suspend` functions. They use `suspendCancellableCoroutine` to bridge Volley's callback-based API into Kotlin coroutines.

| Method | Description |
|---|---|
| `syncRequest()` | Top-level sync orchestrator. Calls handshake, conditionally calls the four data syncs, returns `nextRunIn` in seconds |
| `syncSettings(url, defaults)` | Fetches and persists screen settings |
| `syncCalendar(url)` | Fetches and persists the 30-day prayer calendar |
| `syncSlides(url)` | Fetches and persists the media slide list |
| `syncTranslation(url)` | Fetches and persists UI label translations |
| `sendAPIRequest(url, authType)` | Generic suspend HTTP GET that returns the raw JSON string |

### Category 2 — Time Utilities

Pure functions with no side effects. See [Prayer Time Algorithms](../docs/prayer-time-algorithms.md) for detailed pseudocode.

| Method | Description |
|---|---|
| `getDisplayValue(context, displayObject, settings)` | Returns the Iqamah time string to display for a given prayer row |
| `adjustTimeByMinutes(baseTime, minutes, add)` | Adds or subtracts minutes from a time string; returns new time string |
| `getMillisUntil(targetTimeStr)` | Returns milliseconds from now until the target time (handles midnight crossing) |
| `isCurrentPrayer(displayObject)` | Returns `true` if `now` falls within the object's time window |
| `isInIqamahCountdownWindow(...)` | Returns `true` if the Iqamah countdown should be showing |
| `isInPostIqamahWindow(...)` | Returns `true` if the post-Iqamah Jamaat screen should be showing |
| `getHijriDateDisplay(context)` | Returns the correct Hijri date string using the priority chain |
| `getGregorianDateDisplay()` | Returns the formatted Gregorian date string |
| `isInternetAvailable(context)` | Returns `true` if the device has a validated internet connection |

### Additional Utilities

| Method | Description |
|---|---|
| `generateQRCode(content, width, height)` | Generates a `Bitmap` QR code for device registration display |
| `getDeviceID(context)` | Returns the persisted masjid ID from `AuthPreferences` |
| `getApiURL(endpoint)` | Constructs the full URL for a given `UrlEndpoints` enum value |
| `parseSyncResponseJSON(json)` | Parses the `/sync` response into a `Settings` object (control flags only) |
| `parseSettingsJSON(json)` | Parses the `/screen_settings` full response into a `Settings` object |
| `parseCalendarJSON(json)` | Parses the `/calendar` response into a `List<PerpetualCalendar>` |
| `parseSlidesJSON(json)` | Parses the `/slideshow` response, decoding Base64 BLOBs, into a `List<Slides>` |
| `parseTranslationJSON(json)` | Parses the `/translation` response into a `Translations` object |
