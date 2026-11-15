# Private client diagnostic reports

Settings > Troubleshooting now includes **Send diagnostic report now**. It requests an immediate upload by the existing service, using the same opt-in and token as automatic reports. Check Diagnostic export for the verified upload result.

Focused app evidence includes the last UI action and allocation/modal/return stage, an app-session marker that survives termination, and short fingerprints of the installed search/window source files. A session without a matching live Kodi lease is reported as interrupted; this can mean a crash, forced close, power loss, or interrupted shutdown, and is not proof of a native crash. Normal closure and caught Python exceptions are recorded separately.

No titles, queries, URLs, exception messages, paths or session tokens are included in these additional fields. Android native crash dumps still require platform tools; the app cannot obtain those through Kodi's API.

Automatic diagnostic summaries use the existing fine-grained Contents read/write token for daanvb/strem4kodi-sync. The repository must be private; uploads fail closed otherwise. Settings > Troubleshooting includes an opt-out and export status. Without a token nothing uploads.

The service checks every 15 minutes while playback is idle. Each installation receives a random identifier unrelated to hardware or account identity and updates diagnostics/<identifier>.json. Only changed reports upload. Uploads are read back and verified; failures retry on a later check without blocking playback. Git history retains older versions, so deleting a current file does not erase history.

Includes broad software/platform versions, startup/performance timings, last playback stage timings, provider request status/durations, search category result counts, requested IMDb episode identifiers and release episode tags, selected-source format markers and metadata field availability, refresh-rate configuration, bounded counts of known Kodi error/crash patterns from kodi.log and kodi.old.log, and a bounded runtime trace for navigation and playback recovery. Recovery events may include an allowlisted failure reason and outcome plus retry attempt and limit values capped at 25; these fields describe the retry decision without identifying a title, episode, source or URL.

Version 2 reports also include readable technical log excerpts from **both kodi.log and kodi.old.log**. Each log exports at most 64 entries: the latest 32 recognised failures/traceback frames, the latest 16 errors, and the final 16 timestamped/recognised entries, deduplicated in original order. Timestamps, thread numbers, severity, control/window IDs, known app navigation fields, native signal names and this addon's traceback file basenames/line numbers are retained. The line index is relative to the scanned tail, not necessarily the complete log. Unknown messages appear as `[message omitted]`; arbitrary exception messages, traceback source code and external file names are omitted. No best-effort secret redactor is trusted with raw messages. This means an unknown error may still require a separately reviewed log, but missing-control errors and their addon frame locations can now be investigated directly from private reports.

Excludes raw logs, raw exceptions, search text, release filenames, provider configuration, playable URLs, credentials, account details, device names and filesystem paths. Log scanning reads only the final 256 KiB of each log and drops an incomplete first line. These are indicators, not a proven crash diagnosis: Android native crash dumps and OS logcat are not accessible through this feature. A crash may prevent upload until Kodi restarts; kodi.old.log then supplies available evidence. The existing service checks immediately when first running and subsequently every 15 minutes while idle; manual reports use the same collection path. An idle check may miss transient session timing before restart, and further Kodi restarts can rotate away the previous crash log.

Reports require authenticated access to the private repository for review. No public issues are created and no LAN access is needed. Automatic exports do not directly probe or fetch media streams.

Source-choice evidence now includes the latest complete or early automatic
selection: quality/fast priority, explicit provider/format preferences, bounded
candidate counts and technical comparisons. It records the chosen source, up to
eight eligible alternatives and four excluded examples, including fixed reasons
such as blacklisting, episode identity/title, quality ceiling, filters or language.
Only fixed provider families, reported service (TB, RD or Unknown), format enums, booleans and bounded numeric measures
are retained and revalidated for export. No titles, filenames, media IDs, playback
links, link hashes or configured provider names are included. This uses the
existing diagnostic consent/upload flow and makes no extra provider requests.

## Seek and buffering timeline

The `seek_recovery` section retains four recent owned playback episodes separately
from navigation events. Each episode keeps the latest 24 samples, taken at most
once every two seconds during an observed seek, pending resume or buffering.
Healthy playback does not write samples. The final outcome distinguishes cleared
buffer/seek flags, pause, lost observation, cancellation, replacement, native
playback errors and the buffering watchdog. Cleared flags are not proof of smooth
decoded video. An `observing` episode may be incomplete if the app was interrupted.

Samples contain elapsed time, cache availability/percentage, the cache high-water
mark and no-progress age immediately before the watchdog check, seek/pause flags,
pending-resume state and playback-position deltas between samples. Final decision
counters are saved separately. Queue high-water marks and the last progress
signal (`cache_pct` or one of the four fixed queue fields) show why the watchdog's
no-progress deadline was renewed. During buffering, diagnostic samples reuse the
exact queue readings supplied to the watchdog. Neither absolute playback positions nor source
URLs, titles, playback attempt IDs, hashes or exception text are stored. A random
diagnostic-only trace ID and UTC timestamps correlate each bounded episode.

Audio/video queue and data-queue percentages use p3i/CoreELEC 21's dotted labels,
with Kodi 22 label fallbacks. Numeric Amlogic frame-rate triplets and drop values
are included when available; the triplet is retained as three numbered values
without inferring its meaning across firmware versions. Unsupported or invalid
measurements are omitted, never converted to zero. Native log excerpts remain
necessary to correlate decoder errors with these measurements.

These local records join the existing opt-in diagnostic summary and playback
report bundles. They add no network probes, upload worker or timeout changes.
After reproducing a failure on a version containing this instrumentation, use
**Settings > Troubleshooting > Send diagnostic report now** and check the upload
result. Existing automatic exports also include this section.

## Passive source-search measurements

Normal source searches save the latest bounded provider timing snapshot through
the existing search worker, without extra requests, polling, threads or uploads.
Open **Settings > Troubleshooting > Last source search** to view the per-provider
breakdown. This is separate from Last playback loading and works without starting
playback or recording Home startup. Existing 1.8.146 snapshots are readable after
updating. An empty report explains how to obtain a fresh search using Sources >
Refresh; opening a cached source list may not make a new search.

**Send source-search report** opens an addon-owned review with Send and Cancel.
Only Send requests an upload by the existing service. Private diagnostic reporting
must be enabled and its private sync token configured. The timing-only payload
contains the bounded provider measurements and broad software versions; it does
not collect Kodi logs, playback history, titles, stream links or account details.
The destination is a separate `diagnostics-source/<client>.json` slot in the
configured private repository, verified private before writing and read back to
verify success. One current report per client is kept; Git history retains earlier
versions. This does not replace the ordinary diagnostic or playback reports.
Reopen Troubleshooting to refresh **Source-search upload**, which distinguishes
verified upload, failure and cancellation. Viewing and cancelling upload nothing.

Explicit source-search uploads make at most two attempts for a connection failure,
temporary server failure, concurrent write or failed read-back verification. A
retry rechecks consent/token, fetches the current file SHA and verifies the result.
Denied access, a public/redirected repository, rate limits and server-requested
backoff are not automatically retried. Failure status distinguishes these causes
using fixed messages; unexpected failures retain only their class and package
stack locations in the ordinary diagnostic failure history, never response bodies
or exception text. The upload remains timing-only and starts no automatic report.

The report includes these measurements:

- `request_ms`: time obtaining the response, including protocol cache lookup,
  HTTP/server wait, response reading and JSON decoding; retry attempts are summed.
- `parse_ms`: local source validation and row construction after the response.
- `delivery_ms`: time delivering accumulated results to the subscribed callbacks,
  including any synchronous ranking/list-control work in those callbacks.
- `published_ms`: elapsed search time when a successful result callback returns.
  Missing for empty results, absent callbacks or a callback failure.
- Total provider time, result count, failure flag and bounded retry-attempt count.

These distinguish provider-response latency from local handling. They do not
measure actual rendered frames, isolate AIOStreams' internal services, or measure
stream startup. Cached picker opens without a provider search retain the previous
search snapshot. Cancelled/inactive subscribers may ignore a delivered callback;
callback completion alone is not proof that cards were visible.

Only fixed families AIOStreams, Usenet Ultimate, PenguPlay and Other add-on are
retained, with at most 40 rows and finite bounded numbers. Configured display
names, titles, source URLs, account data and exception text are excluded. Data is
revalidated before inclusion in the existing explicit performance report. This
adds no automatic upload or change to diagnostic-export consent/settings.

### Source-search stage timings

The source-only performance report also includes `streamSearch`, a numeric-only
snapshot of the latest completed picker search. `total_ms` is measured from search
setup through final result handling. `worker_start_ms` and `worker_done_ms` are
offsets from that start; `worker_ms` includes backend preparation, collection and
post-collection work. `cache_ms` measures the worker's final cache/lock stage.
`queue_ms` accumulates delivery-to-pump delay once per coalesced batch.
`result_ms` includes result preparation and rendering; `render_ms` includes
`rank_ms` (filter/sort/group) and `items_ms` (list construction/native calls).
`speed_wait_ms` measures the separate throughput-test wait. These stages overlap
and must not be added together. `render_calls` counts renders before completion.

Snapshots freeze at completion and save off the UI thread. Refresh supersedes an
older generation's pending save. Cached searches may have no worker timings;
provider timings remain the latest provider collection and may precede a cached
search. No title, source URL, account data or raw log is included. Native display
presentation after Kodi returns from its list APIs is not measurable here.

Source-list follow-up measurements split `items_ms` into `item_prepare_ms`
(presentation matching plus item creation) and `native_update_ms` (list replacement,
append and selection). `item_create_ms` is the creation/property-setting subset of
preparation. `cache_lookup_ms` measures saved-source lookup before worker launch.
`items_created`, `items_reused`, `list_resets`, `list_appends` and `list_unchanged`
distinguish real native updates from render passes that do no list update. The
numeric-only bounds and export allowlist still apply; overlapping stages are not
additive.

New source-list items are prepared with Kodi's documented `offscreen=True` and
bulk properties before insertion, then treated as immutable. Changed presentation
gets a replacement item; no setter may mutate a retained item after publication.
This respects the locking caveat for WindowXML containers in the
[Kodi ListItem documentation](https://xbmc.github.io/docs.kodi.tv/master/kodi-dev-kit/group__python__xbmcgui__listitem.html).

Source item creation also reports `item_construct_ms`, `item_properties_ms` and
`item_property_values` (the number of nonempty property values sent to Kodi).
When the platform supports a per-thread CPU clock, `item_create_cpu_ms` and
`item_create_wait_ms` split active thread CPU from approximate non-CPU elapsed
time; `item_cpu_samples` identifies how many created items have that coverage.
CPU/wait fields are omitted when unsupported. Waiting can include GIL contention,
OS scheduling and native waits; these figures alone do not identify which one.
`item_create_ms` remains wall time around construction and property-setting;
these nested metrics must not be added to it. Metrics merge once per render batch.

Kodi 21.2's [setProperties implementation](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/interfaces/legacy/ListItem.cpp)
loops over individual setters. Its [GuiLock](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/interfaces/legacy/AddonUtils.cpp)
still opens/closes delayed calls for offscreen items; the
[Python language hook](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/interfaces/python/LanguageHook.cpp)
releases/reacquires the GIL. Fewer property values therefore reduce potential
thread handoffs. New source items omit empty values and the unused native
`stream_release` property; release names remain in source data, reuse signatures
and the details dialog. Visible labels, service badges and live logos are retained.


## Metadata and artwork monitoring

Settings > Troubleshooting > **Metadata and artwork monitoring** shows passive
measurements from this device. Reproduce the slow Home/title/episode screen,
leave Kodi running for at least 15 seconds, then use **Send diagnostic report now**.
Check **Diagnostic export** for a successful upload and share the diagnostic report
ID for authorised review of the private GitHub report. This uses the existing
private-report opt-in and token; enabling monitoring does not grant public access
or bypass upload consent. No separate speed test or extra image download runs.

The `browsing` section includes up to 128 request observations and 128 presentation/
cache observations, timestamps, sample counts, medians and maximum durations:

- Metadata/catalogue/manifest request results: cached, refreshed or failed,
  AIOmetadata/Cinemeta/Other family, fixed failure category or HTTP status, and
  configured timeout on failures. Request duration includes cache/coalesced waiting
  and network/JSON work; it is not a server-only or WiFi-throughput measurement.
- Metadata first-result and completion times, cancellation and exhausted budgets.
- Nearby Home poster and episode-thumbnail initial cache misses, initial hits,
  later readiness, unknown cache state and still pending after 120 seconds.
- Home full/incremental publications and background refresh duration, to compare
  the timing of artwork symptoms with Home updates.

Request history uses the existing throttled persistence plus one delayed flush,
so the final request is saved within approximately 15 seconds even when browsing
stops. Abrupt exit can lose that unflushed tail. Request histories merge across interpreters; both histories
are revalidated for export. Busy request traffic cannot evict the separate
presentation/cache history. New fields contain only fixed enums and bounded
numbers, with no titles, URLs, URL hashes, configured names or credentials. The
normal diagnostic report still includes its existing other evidence and logs.

Cache sampling observes up to four nearby remote images while the browsing pump
is active and settled; it stops following images that leave that sample. Readiness
starts at first observation, not download start, and cannot prove GPU rendering.
A cache miss may have no subsequent completion observation if the user moves away.
Timings can suggest a bottleneck but cannot establish server versus network cause
without further evidence. These additions neither change artwork quality nor
change metadata cache lifetimes/invalidation.


Artwork observations with `timing_version: 2` use the worker completion time and
restart after observation gaps longer than five seconds. Older artwork history is
shown in separate `.legacy` summary groups: long legacy durations may include time
spent away from the screen, rather than waiting for an image. Native image display
still requires device validation.


## Fresh versus retained timings

Provider rows show the age of the last saved request. A recent `Last: Timed out`
result is still a saved observation, not a live server health check. Results more
than 15 minutes old, from before the latest known app session, or with an unknown/
future timestamp use neutral `Earlier` styling. Details include the absolute UTC
observation time. Reopen Troubleshooting after browsing to refresh these labels.
A subsequent request for the same provider/operation replaces its latest result
as before; unrelated requests do not clear a catalogue failure.

Browsing report version 2 retains bounded `events`, but `summary` now covers only
the last 15 minutes, excluding observations before the latest known app-session
start. `historical_summary` preserves older results separately, including earlier
failures. `captured_at`, `session_started` and `recent_window_seconds` make the
scope explicit; without a known session start the age window alone applies.
Request summaries and newly recorded performance timings retain `checked` times.
Old entries without timestamps are not relabelled as fresh. Startup launch medians
remain explicitly historical measurements.

This does not clear metadata or Kodi image caches and makes no extra provider
requests. Reading the screen writes no diagnostic state. The capture timestamp
alone is excluded from automatic-upload deduplication; actual changed evidence or
an observation moving out of the recent window can still update the report.
