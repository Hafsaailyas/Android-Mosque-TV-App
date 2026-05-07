# UI State Management — MasjidWala TV

## Overview

The UI layer is driven by a single singleton class, `MainUI`, which owns the complete list of `DisplayObject` instances that back every visible row on the prayer time screen. All state mutations — whether triggered by a timer tick, a sync completion, or a prayer transition — flow through this singleton.

---

## The DisplayObject Model

`DisplayObject` is the fundamental UI binding unit. Each visible row on the main screen (Fajr, Sunrise, Ishraq, Zuhr, Asr, Maghrib, Isha, Jummah) corresponds to exactly one `DisplayObject` instance.

```
DisplayObject {
    id                  : Int        // Stable ordering index (1 = Fajr, 2 = Sunrise, ...)
    object_name         : String     // Internal identifier ("fajr", "zuhr", etc.)
    display_label       : String     // Translated label shown in the UI
    display_value       : String     // The time value shown (Iqamah time or prayer time)
    time_start          : String     // When this prayer period begins
    time_end            : String     // When this prayer period ends
    object_container_id : Int        // Android View resource ID to bind this object's data to
    is_display          : Boolean    // Whether this row is visible at all
    is_prayer           : Boolean    // Whether this object represents a namaz (true) or info row (false)
    is_current_prayer   : Boolean    // Whether this is the currently active prayer period
    prayer              : Prayer?    // Nested prayer metadata (iqamah, ending label, next prayer)
}
```

### View Binding Strategy

Rather than holding direct `TextView` references (which would leak if the Activity is recreated), `MainUI` stores only `object_container_id` — the Android `R.id` integer for the corresponding layout view. At render time, `MainActivity` calls `Activity.findViewById(displayObject.object_container_id)` to resolve the live view reference. This decouples `MainUI` from the Activity lifecycle.

---

## Building the DisplayObject List

`MainUI.init(context)` is called once after each successful sync and on each timer cycle. It:

1. Reads `Settings` from `DbFunc`
2. Reads today's `PerpetualCalendar` from `DbFunc`
3. Constructs one `DisplayObject` per configured prayer/info row
4. Applies visibility flags from `Settings` (`showFajr`, `showIshraq`, etc.)
5. Calls `GenFunc` time utilities to compute `time_start`, `time_end`, and Iqamah display values
6. Stores the complete list in `objDisplayObjects`

The list is built in a fixed order (Fajr → Sunrise → Ishraq → Zawal → Zuhr → Jummah → Asr → Sunset → Maghrib → Isha), and the display respects the `is_display` flag on each object to show or hide rows dynamically.

---

## Detecting the Current Prayer

At each timer tick, `MainUI` iterates `objDisplayObjects` and compares each object's `time_start` / `time_end` window against the current device time. The algorithm is:

```
Pseudocode:

currentTime = now()

for each displayObject in objDisplayObjects:
    if currentTime >= displayObject.time_start
    AND currentTime <  displayObject.time_end:
        displayObject.is_current_prayer = true
        currentPrayerObject = displayObject
    else:
        displayObject.is_current_prayer = false

if no prayer window matched:
    // We are between Isha end and Fajr start (late night period)
    currentPrayerObject = fajrObject  // Next upcoming prayer
```

The object identified as `is_current_prayer = true` receives visual emphasis in the layout (e.g., highlighted row, larger font, accent colour).

---

## The Prayer Timer (`prayerTimer`)

A `fixedRateTimer` (Kotlin `Timer.scheduleAtFixedRate`) fires every second. On each tick it performs two jobs:

### Job 1 — Countdown Label and Value

The timer determines which countdown mode is active and updates the central countdown display:

```
Pseudocode:

timeUntilIqamah = currentPrayer.prayer.iqamah_time - currentTime

if timeUntilIqamah > 0 AND timeUntilIqamah <= (settings.qiyamahCountMin * 60):
    // --- IQAMAH COUNTDOWN MODE ---
    countdownLabel.text = translations.countdown_iqmah_label   // "اقامت"
    countdownValue.text = formatAsMMSS(timeUntilIqamah)

    // Show "get ready" message overlay
    messageOverlay.visibility = VISIBLE
    messageLine1.text = translations.message_line1_iqamah_nearby
    messageLine2.text = translations.message_line2_iqamah_nearby

else if currentTime >= currentPrayer.prayer.iqamah_time
     AND currentTime <  currentPrayer.prayer.iqamah_time + (settings.afterQiyamahTimeMin * 60):
    // --- POST-IQAMAH / JAMAAT MODE ---
    countdownLabel.text = translations.message_line1_jamah    // "جماعت"
    messageLine1.text   = translations.message_line1_jamah
    messageLine2.text   = translations.message_line2_jamah
    // Trigger Iqamah slides if iqamah_screen slides exist

else:
    // --- NEXT PRAYER COUNTDOWN MODE ---
    nextPrayerTime = getNextPrayerTime(currentPrayerObject)
    timeUntilNext  = nextPrayerTime - currentTime
    countdownLabel.text = translations.countdown_next_prayer_label  // "اگلی نماز"
    countdownValue.text = formatAsHHMMSS(timeUntilNext)
    messageOverlay.visibility = GONE
```

### Job 2 — Date Display Refresh

On each tick, the date displays (Gregorian and Hijri) are refreshed. The Gregorian date is derived from the system clock. The Hijri date is sourced from the priority chain described in the [Content Sync Strategy](content-sync-strategy.md).

---

## RTL / LTR Layout Direction

The layout direction is determined at login time when the server returns the locale setting alongside the Bearer Token. It is persisted via `AuthPreferences.savePreference(..., layoutDir = "rtl"|"ltr", ...)`.

`MainActivity` applies it at the root `ConstraintLayout` level on `onCreate()`:

```kotlin
// Pseudocode

val dir = AuthPreferences.getLayoutDirection(context)
rootLayout.layoutDirection = when (dir) {
    "rtl" -> View.LAYOUT_DIRECTION_RTL
    else  -> View.LAYOUT_DIRECTION_LTR
}
```

All `TextView` alignments are defined as `START` / `END` (not `LEFT` / `RIGHT`) in XML, so they automatically mirror under RTL without code changes.

---

## State Transitions Summary

```
App Start
    │
    ▼
MainUI.init()  ──►  Build DisplayObject list  ──►  Bind to TextViews
    │
    ▼
prayerTimer.start() (1-second tick)
    │
    ├──► Evaluate current prayer window
    ├──► Update countdown label/value
    ├──► Update message overlay
    └──► Refresh date displays
    │
    ▼  [on sync interval]
syncRequest() completes
    │
    ▼
MainUI.destroy()  ──►  MainUI.init()   (full UI rebuild with fresh data)
    │
    ▼
prayerTimer restarts
```

---

## Thread Safety

All UI mutations are dispatched to the main thread via `Handler(Looper.getMainLooper()).post { ... }` from within the timer callback. The `MainUI` singleton is initialised under a `synchronized` block to prevent race conditions on the first access from the background sync coroutine.
