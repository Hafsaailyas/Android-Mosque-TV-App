# Data Layer — MasjidWala TV

## Overview

The data layer is the single source of truth for all persisted and remote data in the application. It consists of three components: `Database` (schema definition), `DbFunc` (typed CRUD operations), and `VolleySingleton` (HTTP request management). Together they form a clean boundary that the domain and presentation layers never cross directly.

---

## Database (`Database.kt`)

`Database` extends `SQLiteOpenHelper` and is responsible exclusively for schema lifecycle management. It performs no data operations itself.

### Responsibilities

- **`onCreate(db)`** — Executes the three `CREATE TABLE` DDL statements on first install
- **`onUpgrade(db, oldVersion, newVersion)`** — Executes version-gated `ALTER TABLE` migration SQL
- **Schema constants** — All table names and column name constants are defined as `companion object` properties, preventing typo-prone magic strings throughout the codebase

### Design Decisions

- The `settings` table uses a **fixed primary key** (`INTEGER PRIMARY KEY` with value always `1`) rather than `AUTOINCREMENT`. This enforces the single-row contract at the schema level.
- The `namaz_times` table uses the ISO date string (`YYYY-MM-DD`) as its primary key — a natural key that prevents duplicate entries for the same calendar day.
- The `slides` table uses a server-assigned timestamp string as its primary key, which also serves as a change-detection mechanism (a changed slide gets a new ID on the server).

---

## DbFunc (`DbFunc.kt`)

`DbFunc` is the only class that directly instantiates `Database` and executes SQL. Every method opens the database, performs its operation, and closes the connection before returning. This prevents connection leak in the absence of a connection pool.

### Settings Operations

| Method | Description |
|---|---|
| `insertSettings(settings: Settings): Boolean` | Deletes all existing rows, then inserts the provided `Settings` object. Returns `true` on success. |
| `getSettings(): Settings` | Reads the single settings row and maps all columns to a `Settings` instance. Returns a default `Settings()` if no row exists. |
| `settingsExist(): Boolean` | Returns `true` if at least one row exists in the `settings` table. Used by `GenFunc` to determine whether a forced sync is needed. |
| `getHijriDate(): String?` | Convenience method. Reads only the `hijri_date` column from settings. Returns `null` if the settings row does not exist. |
| `deleteAllSettings()` | Internal helper. Truncates the settings table before an insert. |

### Prayer Times Operations

| Method | Description |
|---|---|
| `insertPrayerTimes(list: List<PerpetualCalendar>): Boolean` | Upserts each calendar entry using insert-or-replace by date primary key. |
| `getPerpetualCalendar(date: String): PerpetualCalendar?` | Retrieves the prayer times row for the given `YYYY-MM-DD` date string. Returns `null` if not found. |
| `prayerTimesExist(): Boolean` | Returns `true` if the `namaz_times` table contains at least one row. |

### Slides Operations

| Method | Description |
|---|---|
| `insertSlides(list: List<Slides>): Boolean` | Deletes all existing slides, then bulk-inserts the new list. |
| `getSlides(): List<Slides>` | Returns all rows from the `slides` table as a `List<Slides>`, ordered by `sequence_no` ascending. |
| `slidesExist(): Boolean` | Returns `true` if at least one slide row exists. |

### Implementation Pattern

All methods follow this structure:

```kotlin
// Pseudocode — structural reference only

fun someOperation(): ReturnType {
    try {
        objDatabase = Database(context!!)
        val db = objDatabase!!.writableDatabase  // or readableDatabase

        // ... SQL operation via ContentValues or Cursor ...

        objDatabase!!.close()
        return result
    } catch (ex: Exception) {
        Log.e("->>> DatabaseFunc", "Error in someOperation: ${ex.message}")
        return safeDefault
    }
}
```

---

## VolleySingleton (`VolleySingleton.kt`)

`VolleySingleton` provides an application-scoped HTTP request queue. It is initialised with `context.applicationContext` (not an Activity context) to prevent memory leaks across Activity lifecycle events.

### Singleton Pattern

```kotlin
// Pseudocode — structural reference only

companion object {
    @Volatile
    private var INSTANCE: VolleySingleton? = null

    fun getInstance(context: Context): VolleySingleton =
        INSTANCE ?: synchronized(this) {
            INSTANCE ?: VolleySingleton(context).also { INSTANCE = it }
        }
}
```

The `@Volatile` annotation on `INSTANCE` and the `synchronized` block together implement a thread-safe double-checked locking pattern, ensuring the queue is initialised exactly once even under concurrent access from the sync coroutine and the boot receiver.

### CustomJsonObjectRequest

All API requests are made through `CustomJsonObjectRequest`, which overrides `getHeaders()` to inject the appropriate `Authorization` header based on the `AuthType` parameter:

```kotlin
// Pseudocode — structural reference only

class CustomJsonObjectRequest(
    method: Int,
    url: String?,
    jsonObject: JSONObject?,
    listener: Response.Listener<JSONObject>,
    errorListener: Response.ErrorListener,
    private val authType: AuthType,
    private val credentials: String = ""
) : JsonObjectRequest(...) {

    override fun getHeaders(): Map<String, String> {
        val headers = mutableMapOf("Content-Type" to "application/json")

        when (authType) {
            AuthType.BASIC_AUTH -> {
                // credentials = "username:password"
                val encoded = Base64.encodeToString(credentials.toByteArray(), Base64.NO_WRAP)
                headers["Authorization"] = "Basic $encoded"
            }
            AuthType.BEARER_TOKEN -> {
                // credentials = raw token string
                headers["Authorization"] = "Bearer $credentials"
            }
            AuthType.NO_AUTH -> {
                // No Authorization header — public endpoints only
            }
        }
        return headers
    }
}
```

If `BASIC_AUTH` or `BEARER_TOKEN` is specified but `credentials` is empty, an `AuthFailureError` is thrown immediately, preventing a silent unauthenticated request.

---

## AuthPreferences (`AuthPreferences.kt`)

A Kotlin `object` (singleton) that wraps `SharedPreferences` for session-scoped data that does not belong in SQLite:

| Key | Type | Description |
|---|---|---|
| `login_username` | String | Registered mobile number |
| `login_token` | String | Bearer token for all authenticated requests |
| `login_masjid_id` | String | Server-assigned masjid/device identifier |
| `layout_direction` | String | `"ltr"` or `"rtl"` — persisted per device |
| `is_test_user` | Boolean | Whether the logged-in user is a test/demo account |
| `translationsObj` | String | Serialised JSON of the `Translations` object |

### Key Methods

| Method | Description |
|---|---|
| `savePreference(...)` | Saves all auth fields atomically using `apply()` (async write) |
| `hasPreferences(context)` | Returns `true` if all four core fields are present — used by `LoginActivity` to skip the login flow on relaunch |
| `clearPreferences(context)` | Removes all auth keys — called on logout or re-registration |
| `saveTranslationPreference(context, json)` | Saves the serialised translations blob |
| `hasTranslationPreferences(context)` | Returns `true` if a translation blob exists |
| `getLayoutDirection(context)` | Returns `"ltr"` or `"rtl"` for the current device session |
