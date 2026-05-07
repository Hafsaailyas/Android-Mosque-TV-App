# Media Slideshow Flow — MasjidWala TV

## Overview

The MasjidWala TV slideshow system displays a sequence of mosque-specific media (announcements, event banners, community notices) alongside the prayer time display. Images are fetched from the server API, stored as binary BLOBs in SQLite, and rendered in a `ViewPager2`-backed carousel with a synchronised horizontal progress bar indicating slide duration.

---

## Storage Model

### BLOB Persistence

Slide images are stored in the `slides` SQLite table as raw byte arrays in the `objectContent` (BLOB) column. This design choice:

- **Eliminates file system dependency** — no external storage permissions required
- **Guarantees atomicity** — the image and its metadata are written and deleted together
- **Simplifies cache invalidation** — a full table replace on sync clears all old content

On sync, the server delivers images as Base64-encoded strings in the JSON response. `GenFunc` decodes them to `ByteArray` before passing them to `DbFunc.insertSlides()`.

### Slide Domain Model

```
Slides {
    id             : String   // Unique server-assigned ID
    objectContent  : ByteArray // Decoded image bytes (PNG/JPG/GIF)
    masjid_id      : Int
    name           : String   // Human-readable name for logging
    type           : String   // "png", "jpg", "gif"
    sequence_no    : Int      // Ascending display order
    second_to_show : Int      // Duration in seconds (used by progress bar)
    full_screen    : Int      // 1 = fullscreen, 0 = framed
    status         : Int      // 1 = active, 0 = inactive (filtered out before display)
    iqamah_screen  : Int      // 1 = only show during Iqamah window
    updated_at     : String
}
```

---

## Loading Flow

### Step 1 — Query Active Slides

```
Pseudocode:

allSlides = DbFunc.getSlides()
activeSlides = allSlides
    .filter { it.status == 1 }
    .sortedBy { it.sequence_no }

if (isIqamahWindowActive):
    displaySlides = activeSlides.filter { it.iqamah_screen == 1 }
    if displaySlides.isEmpty():
        displaySlides = activeSlides  // fallback to all slides if no Iqamah-specific ones exist
else:
    displaySlides = activeSlides.filter { it.iqamah_screen == 0 }
```

### Step 2 — Decode BLOBs to Bitmaps

Each `ByteArray` in `objectContent` is decoded to an Android `Bitmap` using `BitmapFactory.decodeByteArray()`. Decoding is performed off the main thread (inside a coroutine or background handler) to prevent UI jank.

```
Pseudocode:

for each slide in displaySlides:
    bitmap = BitmapFactory.decodeByteArray(slide.objectContent, 0, slide.objectContent.size)
    if bitmap != null:
        bitmapList.add(Pair(bitmap, slide.second_to_show))
```

### Step 3 — Populate CarouselAdapter

`CarouselAdapter` is a `RecyclerView.Adapter` subclass backing the `ViewPager2`. Each item view holds an `ImageView` that receives the decoded `Bitmap`. `ViewPager2` is configured with:

- `orientation = ViewPager2.ORIENTATION_HORIZONTAL`
- `offscreenPageLimit = 1` (preloads the adjacent slide)
- A `PageTransformer` for smooth transition animation

---

## Auto-Advance Logic

Slide advancement is managed by a custom `Handler`-based scheduler, not `ViewPager2`'s built-in auto-scroll (which doesn't support per-slide durations).

```
Pseudocode:

function scheduleNextSlide(currentIndex: Int):
    currentSlide = displaySlides[currentIndex]
    delay = currentSlide.second_to_show * 1000L  // convert to milliseconds

    handler.postDelayed({
        nextIndex = (currentIndex + 1) % displaySlides.size
        viewPager2.setCurrentItem(nextIndex, smoothScroll = true)
        startProgressBar(nextIndex)
        scheduleNextSlide(nextIndex)
    }, delay)

// Called on ViewPager2 onPageSelected callback to cancel pending advances
// when the user (or Iqamah trigger) changes the slide externally
function cancelScheduledAdvance():
    handler.removeCallbacksAndMessages(null)
```

---

## Progress Bar (Horizontal)

Each slide has an associated horizontal `ProgressBar` that fills from 0% to 100% over `second_to_show` seconds, giving the viewer a visual cue of how long the current slide will remain visible.

### Implementation via `ProgressBarHandler`

`ProgressBarHandler` wraps Android's `CountDownTimer` to animate the progress bar:

```
Pseudocode:

class ProgressBarHandler(progressBar: ProgressBar, durationMs: Int):

    function startProgressBar():
        progressBar.progress = 0

        timer = CountDownTimer(durationMs, intervalMs = 1000):
            onTick(millisRemaining):
                elapsed   = durationMs - millisRemaining
                progress  = (elapsed * 100 / durationMs).toInt()
                progress  = min(progress, progressBar.max)  // clamp to max
                progressBar.progress = progress

            onFinish():
                if progressBar.progress == 100: return  // prevent duplicate trigger
                progressBar.progress = 100

        timer.start()
```

### Synchronisation with Slide Advance

When `viewPager2.setCurrentItem()` is called (either by the scheduler or by the Iqamah window trigger), the active `ProgressBarHandler` is cancelled and a new one is started with the duration of the incoming slide. This ensures the progress bar always reflects the *current* slide's remaining time.

---

## Iqamah Screen Override

When the `prayerTimer` enters **Post-Iqamah / Jamaat mode**, the slideshow is temporarily redirected to show only `iqamah_screen = 1` slides (if any exist). This typically includes a "صف سیدھی کریں" (Straighten the rows) banner or similar Iqamah announcement graphic.

```
Pseudocode:

prayerTimer.onTick():
    ...
    if (justEnteredIqamahWindow):
        cancelScheduledAdvance()
        iqamahSlides = activeSlides.filter { it.iqamah_screen == 1 }
        if (iqamahSlides.isNotEmpty()):
            reloadCarouselAdapter(iqamahSlides)
            viewPager2.setCurrentItem(0, false)
            scheduleNextSlide(0)

    if (justExitedIqamahWindow):
        cancelScheduledAdvance()
        normalSlides = activeSlides.filter { it.iqamah_screen == 0 }
        reloadCarouselAdapter(normalSlides)
        viewPager2.setCurrentItem(0, false)
        scheduleNextSlide(0)
```

---

## Full Slideshow State Machine

```
DbFunc.getSlides()
        │
        ▼
Filter: status == 1, sorted by sequence_no
        │
        ├── isIqamahWindow? ──► filter iqamah_screen == 1
        └── normal?         ──► filter iqamah_screen == 0
        │
        ▼
Decode ByteArray → Bitmap (background thread)
        │
        ▼
CarouselAdapter.submitList(bitmapList)
        │
        ▼
ViewPager2 renders slide[0]
        │
        ▼
ProgressBarHandler.startProgressBar(slide[0].second_to_show)
        │
        ▼  [after second_to_show seconds]
ViewPager2.setCurrentItem(nextIndex)
        │
        ▼
ProgressBarHandler.startProgressBar(slide[nextIndex].second_to_show)
        │
        ▼  [loop until Iqamah window or sync rebuild]
```

---

## Memory Considerations

Decoded `Bitmap` objects are held in memory for the duration of one carousel cycle. For large images or high slide counts, `Bitmap` size is monitored and oversized images are scaled down using `BitmapFactory.Options.inSampleSize` before display. The `objectContent` `ByteArray` raw bytes are not retained in memory after decoding; they are loaded from SQLite only when the carousel is rebuilt.
