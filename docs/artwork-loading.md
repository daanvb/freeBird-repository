# Artwork loading and response reuse

Poster resolution and title details share the complete Stremio metadata response through `core/protocol.py` and `lib/metadata_response.py`. Sharing includes concurrent requests inside one interpreter and disk reuse across interpreters. Each caller receives a copy; title enrichment cannot mutate another caller's response.

Keys include the exact provider URL, artwork generation/revision, profile cache location and current cache generation. Memory and in-flight keys also include cache policy. Refresh artwork and provider-role changes advance the artwork revision; category clears revoke old writers. Explicit bypass and disabled caches retain no shared response. A poster retry bypasses cached partial metadata and replaces the shared response when successful. Derived poster entries cannot outlive the source response's deadline. Existing stream and subtitle request paths are unchanged.

The shared format uses a new cache namespace, so older raw metadata responses are fetched once before becoming reusable. Existing valid derived posters remain available. This caches JSON and image URLs, not downloaded image bytes; Kodi still owns the image cache.

## Home handoff and refresh recovery (1.8.270)

A generated Home rebuild first retains newer resolved artwork, then prepares cached catalogue rows. The second step previously overwrote that retained artwork with an older disk response. The regression now exercises both steps together; older same-signature responses and fallback-only results cannot displace an already resolved preferred poster. Newer results and explicit provider/artwork revision changes still apply.

Successful title metadata now publishes artwork to Home before optional hero formats, ratings and credits, and again when returning from details. Only the configured artwork provider and the request's original artwork signature qualify. Capture that signature before the request so a concurrent refresh cannot relabel an old response as current. This handoff updates native artwork without changing progress/focus, performing another provider request or extending disk-cache expiry.

The handoff rebinds the native image using an option-free `image://` wrapper. Kodi uses a distinct control/texture-manager identity while unwrapping the same original URL for its persistent cache and download. No query parameters or signed upstream URLs are changed. See [Kodi texture cache](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/TextureCache.cpp) and [image URL unwrapping](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/TextureDatabase.cpp).

The wired Ugoos report includes 45–90 second artwork waits and posters disappearing during “Updating Home”. The stale refresh replacement is reproduced offline; the hardware's complete latency cause is not established. In particular, Kodi already routes remote URLs through its background loader even without a `background` XML attribute, so adding that attribute alone is not evidence of a network latency fix ([GUITexture](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/guilib/GUITexture.cpp), [TextureManager](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/guilib/TextureManager.cpp)).

## Placeholder recovery (1.8.269)

Generated cinematic Home/Discover cards must carry `background="true"` and the dynamic `poster_fallback` texture attribute. The source template alone is insufficient because card generation replaces those controls. Tests now inspect the final generated layout, including focused cards and the Discover grid. Kodi resolves dynamic fallbacks when texture allocation fails ([GUIImage.cpp](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/guilib/GUIImage.cpp)); this does not detect or cancel a still-running image request.

While preferred metadata is pending, a valid non-legacy catalogue poster can display immediately as a temporary fallback. Provider ownership remains pending and the preferred result still replaces it. Rows carrying an invalidated provider signature retain their neutral placeholder until resolution, preserving explicit refresh rules. No image sizes, signed URLs or cache identities change. This removes metadata waiting from the first-image path when usable catalogue artwork exists; it does not establish a measured Ugoos speedup.

## Automatic poster previews (1.8.267)

Appearance > Poster loading offers **Full quality** (default) and **Automatic measured speed**. The choice stays on this device and is excluded from portable setup. It does not infer screen size from resolution or Wi-Fi. Enabling it on a Fire TV Stick does not lower image quality on another device.

Automatic shares the existing asynchronous Cloudflare throughput meter, even when source bitrate limiting is off. It neither enables nor changes a video bitrate cap. Tests download at most 10 MB, reuse a result for five minutes, invalidate on detected network changes, and stop during playback. Until a valid measurement exists, keep full quality. Confirmed Ethernet keeps full quality; unknown Android interface types can use measured throughput.

Below 10 Mbps, supported posters use 342-pixel-wide previews. At 15 Mbps or above, new posters use full quality; between those thresholds retain the prior choice. These initial thresholds require device tuning. Direct public TMDB artwork and TMDB fallback images explicitly supplied in supported poster-proxy URLs can provide previews. Never edit the signed upstream URL or invent a variant endpoint. Unsupported artwork remains full quality. A fallback preview may temporarily omit the configured provider's overlays/badges or use its alternative artwork; the exact configured poster is restored on upgrade.

Before choosing a preview, query only the exact full-image URL in Kodi's texture database and check its cached file. Keep cached full-quality artwork; failed cache inspection also keeps full quality. Cache probes are bounded in memory, read-only and never exported. Preview state exists only on native list items, not in provider metadata or saved snapshots.

The Home browse pump upgrades the selected poster and nearby items after five seconds without input (15 seconds below 3 Mbps, 30 seconds below 1 Mbps). Retain a preview layer underneath the asynchronous full image. Dispatch upgrades at least two seconds apart and wait for the previous image's cache entry, with a 15-second escape for failed/stalled loads. No upgrades are queued for other rows. Playback, suspended windows, navigation input and closed windows pause or retire queued work; Kodi owns any image download already dispatched. Cold progressive loading downloads both variants, so it can improve first-image latency while increasing total bytes.

The full-quality default and source URLs are unchanged. No provider settings, server deployment, texture deletion or global image-size changes are required. This does not promise a preview for every signed provider poster: a supported fallback must be supplied.

References: [TMDB image URLs](https://developer.themoviedb.org/docs/image-basics), [Kodi texture cache inspection](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/interfaces/json-rpc/TextureOperations.cpp).

## Device validation still needed

The affected Fire TV Stick is on Wi-Fi and is not remotely accessible. Offline request-count tests do not establish native image-cache reuse or a startup speedup. After an authorised installation, compare the same Home row on first visit, return navigation and Kodi relaunch without clearing caches. Distinguish metadata completion from image availability and record elapsed times and counts only. Cache inspection must remain read-only and must not export configured image URLs, account data or raw logs. Repeated slow rendering can also involve failed image requests or changing URLs; do not assume it proves cache eviction.


## Progressive metadata delivery (1.8.270)

Title details and Home heroes publish usable provider metadata before optional
supplements finish. Partial delivery paints presentation only; episode lists and
playback actions wait for the final shared merge/reconciliation. IMDb episode
reference requests overlap the primary provider, and configured supplements do
not wait for a fallback manifest. Provider role precedence and shared response
cache TTL/invalidation rules are unchanged.

A 20-second caller budget covers identity resolution, provider metadata,
fallback discovery and episode-reference waiting. Individual metadata requests
use at most the remaining budget (capped at 15 seconds). A bounded daemon pool
prevents executor teardown from making the caller wait for a stuck transport;
Python cannot forcibly terminate an already-running socket/DNS call. Cancellation
suppresses late presentation, cancels queued work and leaves transport/cache
invalidation rules intact. Optional format/ratings/credits enrichment and Kodi
image downloads are outside this metadata budget. Budget-exhausted hero results
are displayed but not retained as complete hero-cache entries.

Home schedules poster metadata for the settled selection and at most 19 nearby
cards. Navigation reschedules this window; off-screen pending titles are dropped
before admission. Already-started requests retain their existing timeout and
shared-response coalescing. Existing full-quality and Automatic settings remain
unchanged.

Local performance stages distinguish `metadata.first` from `metadata.complete`.
The active browse pump samples up to four nearby bound image URLs, with at most
two exact texture-cache checks per second on a separate bounded worker. Samples
record cache-ready, inspection-unknown or still-pending after 120 seconds, plus
probe counts and first-check hits. Option-free retry wrappers use the original
cache identity; resized image wrappers retain their transformed identity.
No image URLs, titles or credentials enter these timing records. Reporting uses
the existing explicit performance-report flow. Cache availability is not proof
of GPU rendering; elapsed time begins when a bound card is first observed, not
when its remote download necessarily began. Fire Stick/Ugoos cache reuse and
visible latency still require an authorised real-device comparison.


## Interrupted observation and retired input

Artwork timing version 2 ends completed measurements at the worker's sampling
instant, not the later UI tick that collects the result. A gap of more than five
seconds between samples starts a new observation and discards obsolete results.
This covers modal navigation, playback and suspend without creating additional
workers when an old probe is still running. A first-probe hit still only proves
cache availability at that probe; it is not a download or rendering measurement.
Older artwork events remain in reports under `.legacy` summary groups and must
not be compared as corrected cache durations.

Home and details native input callbacks reject already-closed windows and catch
only the explicit `WindowRetired` cancellation if retirement races a callback.
This does not wait on the renderer, reopen a retired window or suppress unrelated
native errors. Offline tests cover late input and a close occurring during action
handling. These fixes do not establish the cause of provider catalogue timeouts
or prove that every blank poster symptom is resolved on hardware.
