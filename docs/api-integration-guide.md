# API Integration Guide — MasjidWala TV

## Overview

All network communication is handled through `VolleySingleton`, which wraps the Volley library's `RequestQueue` with an application-scoped singleton pattern. A custom request subclass, `CustomJsonObjectRequest`, intercepts the `getHeaders()` method to inject the correct `Authorization` header based on the operation being performed.

---

## Authentication Types

The `AuthType` enum defines three modes, used as a parameter on every outgoing request:

| Enum Value | Header Injected | Use Case |
|---|---|---|
| `NO_AUTH` | None | Public handshake and OTP requests |
| `BASIC_AUTH` | `Authorization: Basic <base64(user:pass)>` | Initial device registration |
| `BEARER_TOKEN` | `Authorization: Bearer <token>` | All authenticated API calls post-login |

The Bearer Token is obtained during the OTP login flow and persisted via `AuthPreferences.savePreference()`. It is retrieved on every subsequent request via `AuthPreferences.getToken(context)`.

---

## Endpoint Reference

The `UrlEndpoints` enum defines all service endpoints. Actual base URLs are configured via a build-time constant (`BuildConfig.API_BASE_URL`) and are not stored in version-controlled source files.

| Endpoint Enum | Method | Path | Auth Type | Purpose |
|---|---|---|---|---|
| `HANDSHAKE` | GET | `/handshake` | `NO_AUTH` | Verify server reachability on first boot |
| `OTP` | POST | `/otp` | `NO_AUTH` | Request an OTP for the given mobile number |
| `LOGIN` | POST | `/login` | `BASIC_AUTH` | Validate OTP, receive Bearer Token and Masjid ID |
| `SYNC` | GET | `/sync?masjid_id=X` | `BEARER_TOKEN` | Receive sync control flags and next-run interval |
| `SCREEN_SETTINGS` | GET | `/screen_settings?id=X` | `BEARER_TOKEN` | Fetch full masjid configuration |
| `CALENDAR` | GET | `/calendar?city=X&country=Y` | `BEARER_TOKEN` | Fetch 30-day prayer time calendar |
| `SLIDE_SHOW` | GET | `/slideshow?masjid_id=X` | `BEARER_TOKEN` | Fetch ordered list of media slides |
| `TRANSLATION` | GET | `/translation?lang=ur\|en` | `BEARER_TOKEN` | Fetch locale-specific UI label strings |

---

## Making an Authenticated Request (Pseudocode)

```kotlin
// Pseudocode — exact implementation is proprietary

val request = CustomJsonObjectRequest(
    method      = Request.Method.GET,
    url         = buildUrl(UrlEndpoints.SYNC, mapOf("masjid_id" to globalDeviceID)),
    jsonObject  = null,
    listener    = { response -> handleSyncResponse(response) },
    errorListener = { error -> handleSyncError(error) },
    authType    = AuthType.BEARER_TOKEN,
    credentials = AuthPreferences.getToken(context) ?: ""
)

VolleySingleton.getInstance(context).addToRequestQueue(request)
```

---

## Request Queue Lifecycle

`VolleySingleton` is initialised with `context.applicationContext` to prevent Activity memory leaks. The underlying `RequestQueue` is lazily instantiated on first use and kept alive for the duration of the application process. Requests are automatically retried by Volley's default `RetryPolicy` on transient failures.

---

## JSON Response Contracts

### Sync Response (`/sync`)

```json
{
  "run_settings_req": true,
  "run_calendar_req": false,
  "run_slideshow_req": false,
  "run_language_req": true,
  "next_request_run_in": 15
}
```

`next_request_run_in` is in **minutes**. `GenFunc.syncRequest()` converts it to seconds before returning to the caller.

### Settings Response (`/screen_settings`)

A flat JSON object containing all fields that map 1:1 to the `Settings` domain class. See [Settings Sample](../samples/Settings.MD) for the full field reference.

### Calendar Response (`/calendar`)

```json
{
  "data": [
    {
      "date": "2025-04-01",
      "city": "Lahore",
      "country": "Pakistan",
      "timezone": "Asia/Karachi",
      "gregorian_day": "Tuesday",
      "gregorian_month": "April",
      "hijri_date": "02-10-1446",
      "hijri_month_en": "Shawwal",
      "hijri_month_ar": "شوال",
      "fajr": "04:32 AM",
      "sunrise": "05:55 AM",
      "zuhr": "12:15 PM",
      "asr": "04:02 PM",
      "sunset": "06:35 PM",
      "maghrib": "06:35 PM",
      "isha": "07:52 PM"
    }
    // ... up to 30 entries
  ]
}
```

### Slides Response (`/slideshow`)

```json
{
  "slides": [
    {
      "id": "17427624759885",
      "masjid_id": 1,
      "name": "announcement_banner",
      "type": "png",
      "sequence_no": 1,
      "second_to_show": 10,
      "full_screen": 1,
      "status": 1,
      "iqamah_screen": 0,
      "updated_at": "2025-03-24 10:24:23",
      "objectContent": "<base64-encoded image bytes>"
    }
  ]
}
```

`objectContent` is decoded from Base64 and stored as a BLOB in the `slides` SQLite table.

---

## Error Handling Strategy

All API calls follow a consistent three-stage error handling pattern:

1. **Network error (no connectivity)** — The `VolleyError` callback is triggered. `GenFunc` logs the failure and falls back to the corresponding `DbFunc` read method to serve cached data.
2. **Server error (4xx / 5xx)** — The status code is extracted from `VolleyError.networkResponse`. A 401 triggers a re-authentication flow; other errors increment a retry counter (max 3).
3. **Parse error (malformed JSON)** — A `try/catch` around `JSONObject` parsing returns a default domain object and logs a structured error message.

> **Design principle:** The app must always show *something*. An API failure is never allowed to result in a blank or crashed screen.

---

## Volley Singleton Pattern

```kotlin
// Pseudocode — structural reference only

class VolleySingleton private constructor(context: Context) {

    companion object {
        @Volatile private var INSTANCE: VolleySingleton? = null

        fun getInstance(context: Context): VolleySingleton =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: VolleySingleton(context).also { INSTANCE = it }
            }
    }

    private val requestQueue: RequestQueue by lazy {
        Volley.newRequestQueue(context.applicationContext)
        // applicationContext prevents Activity/BroadcastReceiver leaks
    }

    fun <T> addToRequestQueue(req: Request<T>) = requestQueue.add(req)
}
```
