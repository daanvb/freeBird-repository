# Artwork loading and response reuse

Poster resolution and title details share the complete Stremio metadata response through `core/protocol.py` and `lib/metadata_response.py`. Sharing includes concurrent requests inside one interpreter and disk reuse across interpreters. Each caller receives a copy; title enrichment cannot mutate another caller's response.

Keys include the exact provider URL, artwork generation/revision, profile cache location and current cache generation. Memory and in-flight keys also include cache policy. Refresh artwork and provider-role changes advance the artwork revision; category clears revoke old writers. Explicit bypass and disabled caches retain no shared response. A poster retry bypasses cached partial metadata and replaces the shared response when successful. Derived poster entries cannot outlive the source response's deadline. Existing stream and subtitle request paths are unchanged.

The shared format uses a new cache namespace, so older raw metadata responses are fetched once before becoming reusable. Existing valid derived posters remain available. This caches JSON and image URLs, not downloaded image bytes; Kodi still owns the image cache.

## Automatic poster previews (unreleased)

Appearance > Poster loading offers **Full quality** (default) and **Automatic measured speed**. The choice stays on this device and is excluded from portable setup. It does not infer screen size from resolution or Wi-Fi. Enabling it on a Fire TV Stick does not lower image quality on another device.

Automatic shares the existing asynchronous Cloudflare throughput meter, even when source bitrate limiting is off. It neither enables nor changes a video bitrate cap. Tests download at most 10 MB, reuse a result for five minutes, invalidate on detected network changes, and stop during playback. Until a valid measurement exists, keep full quality. Confirmed Ethernet keeps full quality; unknown Android interface types can use measured throughput.

Below 10 Mbps, supported posters use 342-pixel-wide previews. At 15 Mbps or above, new posters use full quality; between those thresholds retain the prior choice. These initial thresholds require device tuning. Direct public TMDB artwork and TMDB fallback images explicitly supplied in supported poster-proxy URLs can provide previews. Never edit the signed upstream URL or invent a variant endpoint. Unsupported artwork remains full quality. A fallback preview may temporarily omit the configured provider's overlays/badges or use its alternative artwork; the exact configured poster is restored on upgrade.

Before choosing a preview, query only the exact full-image URL in Kodi's texture database and check its cached file. Keep cached full-quality artwork; failed cache inspection also keeps full quality. Cache probes are bounded in memory, read-only and never exported. Preview state exists only on native list items, not in provider metadata or saved snapshots.

The Home browse pump upgrades the selected poster and nearby items after five seconds without input (15 seconds below 3 Mbps, 30 seconds below 1 Mbps). Retain a preview layer underneath the asynchronous full image. Dispatch upgrades at least two seconds apart and wait for the previous image's cache entry, with a 15-second escape for failed/stalled loads. No upgrades are queued for other rows. Playback, suspended windows, navigation input and closed windows pause or retire queued work; Kodi owns any image download already dispatched. Cold progressive loading downloads both variants, so it can improve first-image latency while increasing total bytes.

The full-quality default and source URLs are unchanged. No provider settings, server deployment, texture deletion or global image-size changes are required. This does not promise a preview for every signed provider poster: a supported fallback must be supplied.

References: [TMDB image URLs](https://developer.themoviedb.org/docs/image-basics), [Kodi texture cache inspection](https://github.com/xbmc/xbmc/blob/21.2-Omega/xbmc/interfaces/json-rpc/TextureOperations.cpp).

## Device validation still needed

The affected Fire TV Stick is on Wi-Fi and is not remotely accessible. Offline request-count tests do not establish native image-cache reuse or a startup speedup. After an authorised installation, compare the same Home row on first visit, return navigation and Kodi relaunch without clearing caches. Distinguish metadata completion from image availability and record elapsed times and counts only. Cache inspection must remain read-only and must not export configured image URLs, account data or raw logs. Repeated slow rendering can also involve failed image requests or changing URLs; do not assume it proves cache eviction.
