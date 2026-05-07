# Prayer Time Algorithms — MasjidWala TV

## Overview

`GenFunc` contains all time-calculation utilities used to determine prayer windows, Iqamah offsets, countdown durations, and display values. This document describes the algorithms as structured pseudocode. The proprietary implementation details are not disclosed.

---

## Core Utility: String-to-LocalTime Conversion

Many inputs arrive as formatted time strings (e.g., `"04:32 AM"`, `"07:15 PM"`). All time calculations begin with parsing these into `LocalTime` objects.

```
Algorithm: parseTimeString(timeStr: String) → LocalTime

Step 1: Trim the input string.
Step 2: Define a formatter matching the API time format: "hh:mm a" (12-hour with AM/PM).
Step 3: Attempt to parse timeStr using DateTimeFormatter with Locale.ENGLISH.
Step 4: If parsing succeeds, return the resulting LocalTime.
Step 5: If a DateTimeParseException is thrown, log the error and return LocalTime.MIDNIGHT
        as a safe default.
```

---

## Core Utility: Time Difference in Milliseconds

Used by the prayer timer to compute countdown values.

```
Algorithm: getMillisUntil(targetTimeStr: String) → Long

Step 1: Parse targetTimeStr into targetTime (LocalTime) using parseTimeString().
Step 2: Get currentTime = LocalTime.now().
Step 3: Compute diff = ChronoUnit.MILLIS.between(currentTime, targetTime).
Step 4: If diff < 0 (target time has already passed today):
            diff = diff + (24 * 60 * 60 * 1000)   // Add 24 hours in milliseconds
            // This handles midnight crossover (e.g., Isha ending at Fajr next day)
Step 5: Return diff.
```

---

## Core Utility: Adjust Time by Minutes

Used to compute derived times such as Ishraq (Sunrise + 15 min) or Iqamah offset times.

```
Algorithm: adjustTimeByMinutes(baseTimeStr: String, minutes: Int, addOrSubtract: Boolean) → String

Step 1: Parse baseTimeStr into baseTime (LocalTime).
Step 2: If addOrSubtract == true:
            adjustedTime = baseTime.plusMinutes(minutes.toLong())
        Else:
            adjustedTime = baseTime.minusMinutes(minutes.toLong())
Step 3: Format adjustedTime back to the API time format "hh:mm a".
Step 4: Return the formatted string.
```

---

## Iqamah Time Display Value

Determines what value to display in the Iqamah column, handling both manual and automatic modes.

```
Algorithm: getDisplayValue(context, displayObject: DisplayObject, settings: Settings) → String

Step 1: Identify the prayer name from displayObject.object_name.

Step 2: If settings.manualTime == true:
            Return the corresponding manual Iqamah time from Settings:
            - "fajr"    → settings.fajrQiyamah
            - "zuhr"    → settings.zuhrQiyamah
            - "asr"     → settings.asrQiyamah
            - "maghrib" → settings.maghribQiyamah
            - "isha"    → settings.ishaQiyamah
            - "jummah"  → settings.jummahQiyamah (Friday only)

Step 3: If settings.manualTime == false (automatic calculation):
            azan_time = displayObject.time_start
            iqamah_offset_min = settings.iqamahCountDownIntervalMin
            iqamah_time = adjustTimeByMinutes(azan_time, iqamah_offset_min, addOrSubtract = true)
            Return iqamah_time

Step 4: For non-prayer rows (is_prayer == false):
            Return displayObject.display_value directly (already set to the relevant time).
```

---

## Friday / Jummah Detection

The Zuhr row is conditionally replaced by the Jummah row on Fridays.

```
Algorithm: isJummahDay() → Boolean

Step 1: Get the current day of week from the system clock (java.time.DayOfWeek).
Step 2: Return (dayOfWeek == DayOfWeek.FRIDAY).

Usage in DisplayObject construction:
    if settings.showJummah == true AND isJummahDay():
        display the Jummah DisplayObject instead of Zuhr
        use settings.jummahQiyamah as the Iqamah time
```

---

## Prayer Window Detection

Determines whether the current time falls within a specific prayer's active window.

```
Algorithm: isCurrentPrayer(displayObject: DisplayObject) → Boolean

Step 1: Parse displayObject.time_start into startTime (LocalTime).
Step 2: Parse displayObject.time_end   into endTime   (LocalTime).
Step 3: Get currentTime = LocalTime.now().

Step 4: Handle midnight-crossing windows (e.g., Isha ends at Fajr next day):
        if endTime < startTime:
            // Window crosses midnight
            return (currentTime >= startTime OR currentTime < endTime)
        else:
            return (currentTime >= startTime AND currentTime < endTime)
```

---

## Iqamah Window Detection

Determines whether the countdown timer should switch to Iqamah mode.

```
Algorithm: isInIqamahCountdownWindow(prayerObject: DisplayObject, settings: Settings) → Boolean

Step 1: Parse prayerObject.prayer.iqamah_time into iqamahTime (LocalTime).
Step 2: Get currentTime = LocalTime.now().
Step 3: windowStartTime = iqamahTime.minusMinutes(settings.qiyamahCountMin.toLong())
Step 4: Return (currentTime >= windowStartTime AND currentTime < iqamahTime)
```

---

## Post-Iqamah Window Detection

Determines whether the Jamaat message and Iqamah slides should be showing.

```
Algorithm: isInPostIqamahWindow(prayerObject: DisplayObject, settings: Settings) → Boolean

Step 1: Parse prayerObject.prayer.iqamah_time into iqamahTime (LocalTime).
Step 2: Get currentTime = LocalTime.now().
Step 3: windowEndTime = iqamahTime.plusMinutes(settings.afterQiyamahTimeMin.toLong())
Step 4: Return (currentTime >= iqamahTime AND currentTime < windowEndTime)
```

---

## Hijri Date Formatting

Constructs the display string for the Islamic date.

```
Algorithm: getHijriDateDisplay(context) → String

Step 1: Check settings.hijriDate. If it is non-empty and not the default ("2000-01-01"):
            Return settings.hijriDate directly (server-provided override).

Step 2: Otherwise, query DbFunc.getPerpetualCalendar(today).
        Extract hijri_date_full_en from the result.

Step 3: If result is null or empty, return a safe placeholder string.
```

---

## Gregorian Date Formatting

```
Algorithm: getGregorianDateDisplay() → String

Step 1: Get the current date from the system clock (LocalDate.now()).
Step 2: Format as: "EEEE, d MMMM yyyy"  (e.g., "Thursday, 1 May 2025")
        using DateTimeFormatter with Locale.ENGLISH.
Step 3: Return the formatted string.
```

---

## Network Availability Check

Before initiating any sync, the application checks for network connectivity.

```
Algorithm: isInternetAvailable(context) → Boolean

Step 1: Get the ConnectivityManager system service.
Step 2: Get the active network and its capabilities.
Step 3: Return true if capabilities contain NET_CAPABILITY_INTERNET AND
        NET_CAPABILITY_VALIDATED.
Step 4: If any step throws or returns null, return false.

Note: This is a pre-flight check only. Individual API calls implement their
own error handling; a positive result here does not guarantee a successful request.
```

---

## Device ID Resolution

A stable device identifier is derived at runtime for use in API calls.

```
Algorithm: getDeviceID(context) → String

Step 1: Retrieve the persisted masjid ID from AuthPreferences.getMasjidId(context).
Step 2: If a valid, non-null value exists, return it.
Step 3: If not yet set (first boot before login), return a temporary placeholder
        that will be replaced after the login flow completes.

// Proprietary device fingerprinting logic — not disclosed
```
