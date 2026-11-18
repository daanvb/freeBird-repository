# Experimental seek preview images

Update to freeBird **1.8.280** (Player remains **0.1.54**), then enable
**Playback → Seek preview images (experimental)** in freeBird settings.
It is off by default and stays local to the device. **Press OK to seek** makes
testing easier: Left/Right selects a position, OK commits it, and Back cancels.
The existing release-to-seek mode still works; generating a picture never delays
or changes the requested seek.

## Decoder availability

On **Fire TV and Shield running Android Kodi 21/22**, enabling the setting requests
the matching `vfs.freebird.preview` package from the freeBird repository. Kodi
handles installation; accept its prompt and keep the repository enabled. The
worker uses this native library through Kodi VFS, with no external executable or
FFmpeg Direct dependency. Four packages cover Kodi 21/22 and ARMv7/AArch64;
selection uses Kodi's process architecture. From 1.8.280, these are on-demand
addons rather than app-manifest imports: Kodi no longer lists incompatible
platform variants as unavailable during the app update. The matching installed
decoder is checked for enabled state; turning previews on enables it if disabled
and shows a confirmation. If already enabled, a confirmation still appears.
The player picks up a completed installation on its next local discovery retry.

**CoreELEC/LibreELEC and desktop** use an executable FFmpeg with HTTP,
Matroska/MP4, the source's video codec, scaling and JPEG encoding. Discovery checks
PATH and `tools.ffmpeg-tools/bin/ffmpeg` / `tools.ffmpeg/bin/ffmpeg` in Kodi's addon
directory. `tools.ffmpeg-tools` is an **on-demand** addon,
because the package is supplied by CoreELEC/LibreELEC rather than universally
available on Kodi devices. When previews are enabled in freeBird's Playback
settings on these distributions, freeBird asks Kodi to install that addon if no
executable is available. The device's repository must be enabled; Kodi handles
repository selection, download and any installer prompts. Installation requested
does not mean installation succeeded. A local ten-second discovery retry picks
up the executable during the current playback session without restarting Kodi.
The service never initiates installation or enables addons. Use freeBird's own
Playback settings for setup; Kodi's native addon settings do not run that action.

Missing/unsupported decoders leave the normal timeline available. freeBird never
downloads binaries directly or changes executable permissions. The package layout
is confirmed by [CoreELEC's FFmpeg Tools build](https://github.com/CoreELEC/CoreELEC/blob/coreelec-21/packages/addons/tools/ffmpeg-tools/package.mk).

Native installation and rendering on Fire TV/Shield, CoreELEC FFmpeg Tools setup,
4K CPU/RAM, playback smoothness and HDR thumbnail colour still need device tests.
The Android helper initially supports H.264/HEVC in MKV/MP4 and has no HDR tone
mapping. Missing decoders, unsupported media or failed extraction retain the
normal seek timeline. The investigation below records the earlier design work;
the integrated 1.8.279 path supersedes its prototype-only descriptions.

## Quick device test

1. On 1.8.280, turn previews off and on in freeBird Playback settings. Let Kodi
   install or enable the decoder. On CoreELEC keep its
   device repository enabled; on Android keep the freeBird repository enabled.
2. Start a direct MKV/MP4 movie or episode. Pause for a few seconds, open the seek
   bar, and move Left/Right. Try **Press OK to seek** to inspect before jumping.
3. Check pictures appear, broadly match their labelled times, and cached pictures
   return quickly when revisiting a position. Cold/remote frames can take longer.
4. Change direction quickly, press Back, seek with OK, stop, and play another
   episode. Check for wrong-title pictures, hangs, crashes or playback stutter.
5. Turn previews off to compare playback. Report device/Kodi version, codec and
   whether the issue happens on a cold or previously previewed position. No
   account details or private stream URLs are needed.

When no images are available, the seek bar now shows a fixed status message for
missing/disabled decoders, loading failure, waiting for playback, frame failure
or the per-playback read limit. No media address or raw exception is displayed.
An isolated Linux Kodi 20.5 runtime reproduced `System.HasAddon=true` with
`enabled=false` and a failing VFS open. The production settings function enabled
that addon through Kodi JSON-RPC, after which the production Python/VFS decoder
extracted a synthetic H.264 frame at 2.0 seconds. The test uses the host helper
and an injected Android selector; it does not certify the Android loader or Fire
TV rendering. Unit regressions cover state checking and failure reporting.

The app also requires `resource.uisounds.appletv` 0.0.2, the all-platform
[AppleTv UI Sounds pack in Kodi's official repository](https://kodi.tv/addons/omega/resource.uisounds.appletv/).
Kodi resolves it when installing/updating freeBird. Select **AppleTv UI Sounds**
in Kodi's interface sound settings to use it; dependency installation preserves
the current sound selection and volume. Sound assets are not copied into our ZIP.

## Pipeline and responsiveness

- `lib/seek_preview.py` owns one cancellable worker in the existing service. Only
  owned movie/episode playback in the companion skin is eligible. Startup waits
  five seconds; low/unknown buffer, buffering, resume recovery and native seeking
  suspend generation. A paused or local file can generate without a network buffer.
- The dialog exchanges a hashed source identity, session token and latest target
  through window properties. It performs no media I/O. New targets replace the
  plan and cancel obsolete reads; cached nearby frames stay immediately available
  in a small dialog memory catalogue. Polling is 50 ms in the worker and 100 ms
  in the existing painter; there is no thumbnail wait on remote callbacks.
- Ten-second buckets match tap seeks. The current target, next steps and previous
  step take priority over a 15-point coarse pass across the film. One decoder
  process runs at a time with one decode/filter thread. There is no second player.
- `core/seek_preview.py` seeks FFmpeg directly to a keyframe and produces a maximum
  384×216 JPEG. A loopback adapter permits two overlapping range connections from
  that decoder to avoid blocking a seek behind the previous connection. It shares
  an 8 MiB read cache, serialises upstream reads, and uses the existing reader's
  authentication, redirect stripping and range/validator checks. Raw source URLs
  never enter decoder arguments, published properties, cache names or logs.
- Read budgets are 8 MiB/five seconds per picture and 64 MiB per playback session.
  Cancellation checks run during reads and every 40 ms while decoding; an active
  network request may take up to its two-second socket timeout to finish. Playback
  and UI threads never join it. Per-picture failures back off for 60 seconds.
- JPEGs are committed atomically under `profile/seek-previews/<source hash>/`.
  Storage is capped at 64 MiB and seven days, pruned during use. Switching a file,
  duration or playback session invalidates mailbox ownership. Signed URL changes
  intentionally miss the disk cache. Off stops work and keeps existing cached files.

The three-image strip uses **≈** timestamps because keyframes may precede the
selected bucket. The central seek label remains the actual requested destination.
No alternate film copy is selected. HLS/DASH, live TV, inputstream/DRM playback,
network filesystem schemes and servers without valid byte ranges are excluded.
HDR/Dolby Vision thumbnail colours are not tone-mapped or certified. Large-frame
software decoding and high-bitrate files may hit budgets or be unsuitable for
low-memory devices; native performance remains unverified.

## Verification and device acceptance

Focused regressions: `python tools/run-audit-tests.py test_seek_preview
test_player_scrub test_player_controls test_file_chapters test_file_chapter_cache
test_audit_player test_settings_page` (run as one command).

Real-decoder smoke: `python tools/verify-seek-previews.py --ffmpeg /path/to/ffmpeg`.
Pillow is required on the test host only. This generates three coloured scenes,
extracts and checks six frames from local H.264 MKV and ranged HTTP MP4, and reports
uncached extraction and cached mailbox latency. It never accesses user media.

Before enabling by default, validate native Kodi rendering, sustained Left/Right,
direction reversal, Back/Stop/episode replacement, five-second timeouts, no-range
servers, storage pressure, 4K CPU/RAM load and buffering on CoreELEC, Fire TV and
Shield. Measure cold/warm remote latency and playback buffer drain against the
feature disabled. This development change does not install or publish anything.

## Platform investigation — 18 November 2026

**Recommendation:** retain the executable backend for CoreELEC/LibreELEC and
desktop testing. For Android, first prove a minimal native Kodi addon can install,
load, decode and cancel on the target devices. Prefer a dedicated thumbnail
library with a narrow versioned interface over depending on FFmpeg Direct's
internal FFmpeg symbols. Keep the existing UI, scheduling and range/cache limits.
This recommendation is based on source and package inspection, not an Android
runtime test. No helper has been built, installed or enabled by this investigation.

### What FFmpeg Direct actually provides

The [official Omega listing](https://kodi.tv/addons/omega/inputstream.ffmpegdirect/)
supplies ARMv7 and AArch64 Android builds. Its [Omega interface](https://github.com/xbmc/inputstream.ffmpegdirect/blob/Omega/src/StreamManager.h)
and [Piers interface](https://github.com/xbmc/inputstream.ffmpegdirect/blob/Piers/src/StreamManager.h)
provide stream packets, playback seeking, chapters and timeshift operations, with
no thumbnail request/result interface. The [build configuration](https://github.com/xbmc/inputstream.ffmpegdirect/blob/Omega/depends/common/ffmpeg/CMakeLists.txt)
explicitly disables the `ffmpeg` and `ffprobe` programs. Adding this dependency
cannot make our present subprocess decoder work.

However, **direct reuse of its native internals is not impossible**. Both official
21.3.8 Android ZIPs were downloaded to ignored research storage and inspected as
data, without loading their libraries. Each contains one
`libinputstream.ffmpegdirect.so` and no `ffmpeg`/`ffprobe` executable. Both export
14 selected FFmpeg functions, including `avformat_open_input`, `av_seek_frame`,
`av_read_frame`, `avcodec_find_decoder`, `avcodec_open2`,
`avcodec_send_packet` and `avcodec_receive_frame`, plus `ADDON_Create`.

Neither binary exports `sws_getContext`, `sws_scale`, `avcodec_version`,
`avformat_version`, `av_version_info` or `avcodec_configuration`. Its dynamic
dependencies are `libm.so`, `libdl.so`, `libc++_shared.so` and `libc.so`, not
separate FFmpeg libraries. Function exports alone do not prove a particular
decoder/encoder works or that Android can resolve all dependencies in our caller.
A bridge could potentially use those exports plus separate scaling/encoding, but
would need verified FFmpeg structure layouts, loader/lifetime handling and version
gates. The source dependency changes from [FFmpeg 6.0 on Omega](https://github.com/xbmc/inputstream.ffmpegdirect/blob/Omega/depends/common/ffmpeg/ffmpeg.txt)
to [9.0.2 on Piers at inspection](https://github.com/xbmc/inputstream.ffmpegdirect/blob/Piers/depends/common/ffmpeg/ffmpeg.txt).
These internal exports are not the addon's compatibility contract. Avoid Python
`ctypes` structures tied to one build in production; mistakes can crash Kodi.

Reproducible package identity (links are those from the official listing):

| Package | ZIP bytes | SHA-256 |
| --- | ---: | --- |
| [21.3.8 Android ARMv7](https://ftp.halifax.rwth-aachen.de/xbmc/addons/omega/inputstream.ffmpegdirect%2Bandroid-armv7/inputstream.ffmpegdirect-21.3.8.zip) | 11,647,753 | `54550ce557adca6747d5883c2c85c4a7cf1fe6d43e459bdcbeae13c7c7443389` |
| [21.3.8 Android AArch64](https://ftp.halifax.rwth-aachen.de/xbmc/addons/omega/inputstream.ffmpegdirect%2Bandroid-aarch64/inputstream.ffmpegdirect-21.3.8.zip) | 11,391,357 | `26b0f54cfd1964ab78c802a218c2bd6053d959cdee1f65ddc8d6145807efdc80` |

Verification used a Python standard-library ELF section/dynamic-symbol reader,
then independently checked both files with pyelftools: ELF class/machine,
defined global/default-visible selected symbols, missing scaling exports and
`DT_NEEDED` libraries. Both checks passed. Temporary ZIPs, inspection scripts and
pyelftools are development-only, not addon dependencies or release payloads.
Piers was source-inspected; no Piers binary equivalence is claimed.

freeBird's own `lib/live_tv_playback.py:resolve` currently selects
`inputstream.adaptive`. The presence of FFmpeg Direct on a device does not mean
this Live TV path invokes it. No Live TV route was changed.

### Android packaging is possible without an executable

Android's [API 29 execution restriction](https://developer.android.com/about/versions/10/behavior-changes-10#execute-permission)
specifically prevents directly executing files in writable app storage under
the stated OS/target conditions. It is not evidence that all native Kodi addons
are impossible. Kodi's [Android addon loader](https://github.com/xbmc/xbmc/blob/Omega/xbmc/addons/binary-addons/AddonDll.cpp)
copies addon libraries to `special://xbmcaltbinaddons/`, with a
`KODI_ANDROID_LIBS` fallback; its [shared-library loader](https://github.com/xbmc/xbmc/blob/Omega/xbmc/cores/DllLoader/SoLoader.cpp)
uses `dlopen`. The Android handling is also present in current Kodi master.

A native addon therefore has an established loading route worth testing. One
possible supported Python-to-native bridge is a [Kodi VFS addon](https://github.com/xbmc/xbmc/blob/Omega/xbmc/addons/kodi-dev-kit/include/kodi/addon-instance/VFS.h)
with a private preview protocol: the Python service requests bounded JPEG bytes
through Kodi's filesystem API on its worker. The follow-up below proves this
interface in a host harness; actual Kodi routing, instance lifetime and thread
behavior still need device validation before adoption. A bare `.so` in a Python addon would need its own
loader integration; Kodi's binary-addon path does not automatically apply to it.

Build a small FFmpeg library from pinned upstream source, with MKV/MP4 demuxing,
H.264/HEVC software decoding, scaling and JPEG output initially. Keep its FFmpeg
symbols private so it cannot collide with Kodi or FFmpeg Direct, and expose only
the small helper interface. Preserve applicable source/license materials when
distributing builds; use [FFmpeg's build/license guidance](https://ffmpeg.org/legal.html).
Do not copy Stremio Origin's native binary into freeBird.

Produce Android ARMv7 and AArch64 packages selected for **Kodi's process ABI**,
not just the processor model. Match the Kodi addon ABI and Android minimum SDK;
validate the runtime dependencies and page alignment. Android's [ABI guide](https://developer.android.com/ndk/guides/abis)
and [page-size guide](https://developer.android.com/guide/practices/page-sizes)
describe these build constraints. Fire TV here means Fire OS devices running
Android Kodi; this does not establish support for Vega OS devices. Package/feed
support and actual device installation remain work to do; the follow-up below
adds isolated development builds.

### Alternatives compared

| Route | Finding | Decision |
| --- | --- | --- |
| CoreELEC/LibreELEC FFmpeg Tools | Their [package recipe](https://github.com/CoreELEC/CoreELEC/blob/coreelec-21/packages/addons/tools/ffmpeg-tools/package.mk) installs executable tools under `bin/`; this fits our prototype. | Keep optional; validate the installed build on each target distribution. It is not an Android package. |
| Stock FFmpeg Direct | Android binaries contain useful FFmpeg exports, but no supported thumbnail API or scaling exports checked above. | Possible research bridge, not a drop-in dependency. Prefer a controlled helper to tracking private internals. |
| Extend/fork FFmpeg Direct | We could add a separate preview interface/worker and required conversion code. | Technically plausible, but requires maintaining native builds and separating preview lifetime from playback. No advantage yet over a small dedicated addon; do not replace the user's installed addon. |
| Kodi thumbnail API | [Video thumbnail loader](https://github.com/xbmc/xbmc/blob/Omega/xbmc/video/VideoGeneratedImageFileLoader.cpp) calls [ExtractThumbToTexture](https://github.com/xbmc/xbmc/blob/Omega/xbmc/cores/VideoPlayer/DVDFileInfo.cpp), choosing one third of the file or a chapter start and forcing software decoding. | Can supply coarse chapter previews, not arbitrary seek positions in an unchanged source. The ordinary video-image route also depends on Kodi's extract-thumbnail setting. |
| Small sample clips + Kodi thumbnail API | Inference: reconstruct a bounded local MKV containing a selected random-access frame/group, then ask Kodi's `image://video@...` loader for its thumbnail. | Worth a limited proof of concept before committing to native packaging. Requires new container/index handling and tests for decoder reordering/EOF, texture-cache cleanup, CPU and cancellation. No working backend demonstrated; one isolated keyframe is not guaranteed to produce a picture. |
| RenderCapture / another Python Player | [RenderCapture](https://github.com/xbmc/xbmc/blob/Omega/xbmc/interfaces/legacy/RenderCapture.h) captures the active player; Python [seekTime](https://github.com/xbmc/xbmc/blob/Omega/xbmc/interfaces/legacy/Player.cpp) seeks the application player. | Capturing already-watched positions is possible, but does not provide future frames. Do not seek the visible player to make previews. |
| Android MediaMetadataRetriever | [getScaledFrameAtTime](https://developer.android.com/reference/android/media/MediaMetadataRetriever#getScaledFrameAtTime(long,int,int,int)) offers keyframe choices on API 27+, with older unscaled frame APIs. | An Android/JNI bridge or companion app is needed. Codec/container coverage, blocking calls, headers/range behavior and resource contention need testing. Not a guaranteed hardware-accelerated shortcut or directly callable Kodi Python API. |
| Companion Android app | Can bundle FFmpeg or use Android media APIs in a separate process. | Fallback with better crash isolation, but adds installation, IPC/authentication and service-lifecycle work. Not required merely because the executable backend is unsuitable. |
| Existing trickplay images/server | [Jellyfin](https://github.com/jellyfin/jellyfin/blob/master/Jellyfin.Api/Controllers/TrickplayController.cs) exposes generated tiles; [service.trickplay](https://github.com/jonnypuma/service.trickplay) reads/generates sidecars using external FFmpeg for generation. | Excellent when images already correspond to this file. No universal source of thumbnails for arbitrary freeBird streams. A PC/NAS generator adds a separate server dependency. |
| Title-based thumbnail service | [Seekr](https://seekr.tv/) advertises title-ID/duration lookup returning WebVTT and sprites. No authenticated API was tested. | Avoid as the default: availability and exact-cut alignment are unverified, and it adds a service/key and disclosure of viewing selections. Its illustrative latency is not our benchmark. |

### How to make the eventual backend responsive

[Stremio Origin's preview scheduler](https://github.com/stremio-origin/androidtv/blob/8ef1d29417ba75b4b88ad300d31bfb336f09175c/src/core/seekprev.h)
prioritises focused keyframes and caches pictures. Its [native build](https://github.com/stremio-origin/androidtv/blob/8ef1d29417ba75b4b88ad300d31bfb336f09175c/tools/build-ffmpeg.sh)
embeds selected FFmpeg components; it is not installing an FFmpeg command through
Kodi. The transferable design is indexed range reads, a warm decoder and cache,
latest-target priority, and backing off when playback needs resources.

For freeBird, retain one background decoder and immediate cached display. Avoid
opening/probing the full source anew per thumbnail where a warm session can be
reused. Read the index once; bound fetched data; return the actual decoded frame
timestamp so the approximate label reflects the image. Do not substitute another
cut merely because its title and duration match. Start with one software decoder
thread and no concurrent hardware decoder; test any hardware path separately.
Scaling to 384x216 does not cap memory needed to decode a 4K frame.

A native in-process worker cannot safely be force-killed like our current child
process. Use cooperative I/O interruption, deadlines and bounded packet/decode
work; [FFmpeg's interrupt callback](https://ffmpeg.org/doxygen/8.1/structAVIOInterruptCB.html)
addresses blocking I/O, not an unconditional CPU-decode timeout. Reject stale
results immediately and prevent an old worker from overlapping a new session.
Native crashes still affect Kodi; this is a tradeoff against a separate process.

Next proof-of-concept gates, before claiming support or changing dependencies:

1. Reproduce the native loader path with an inert version/capability request on
   Kodi 21/22, ARMv7 and AArch64. No playback replacement or automatic installation.
2. Extract synthetic H.264/HEVC scenes from local MKV/MP4 and bounded ranged HTTP,
   verify scene colours and actual timestamps, then test malformed/oversized input.
3. Exercise seek reversal, Stop, Back and episode replacement while extraction is
   busy; test no-range servers, expired requests and short reads. A wrong/stale
   image must never escape a retired session.
4. On a representative Fire TV, Shield and CoreELEC device, compare feature off/on
   for render latency, cold/warm preview latency, peak RAM, CPU and playback stalls.
   Include 4K HEVC 10-bit/HDR/DV and a constrained network. Investigate colour
   conversion separately; a decoded frame is not proof of correct HDR thumbnails.
5. Target cached display within 100 ms and remote handlers within 50 ms at p95;
   these are proposed acceptance targets, not measured device results. Cold
   extraction has no instant guarantee. Misses should leave a nearby cached image
   or normal timeline, never block a seek. Retain the existing five-second/byte
   budgets until evidence supports changing them.

At the investigation stage, only the source review and two Android package
inspections above were run. The follow-up proof below adds native builds and host
tests; Android runtime validation is still outstanding. The earlier Windows smoke
timings remain Windows-only. AppleTv GUI sounds
remain an independent, all-platform resource dependency and require no decoder.

### Native helper proof (18 November follow-up)

`companion/vfs.freebird.preview/` now contains an isolated native VFS addon and a
development Python client. It embeds a minimal upstream FFmpeg library, exposes
bounded asynchronous requests, and reads through the existing authenticated
loopback range adapter. No FFmpeg Direct internals or Android executable are
required. The player does not select this backend yet; no dependency/default
setting was changed in this follow-up.

Four local Android development packages were built: Kodi 21/Omega and 22/Piers,
each for ARMv7 and AArch64. Their different global addon ABI requirements are
derived from pinned SDKs. ELF validation checks machine/class, 16 KiB load-segment
alignment, no text relocations, only libc/libm/libdl dynamic dependencies, and
exactly the three Kodi addon entry-point exports. Stripped ZIPs are approximately
1.48 MB (ARMv7) and 1.73 MB (AArch64), including license notices.

The actual shared addon passed seven host integration tests through Kodi's C ABI
with both SDKs and the same minimal FFmpeg 9.0.2 source configuration used for
Android. These check 12 real H.264/HEVC 10-bit SDR frames across MKV/MP4, frame
timestamps/aspect, invalid input, job replacement/cancellation, stale IDs, a
trickling-response deadline, malformed media and oversized dimensions. A separate
helper ASan/UBSan run against system FFmpeg passes with fatal sanitizer errors.
The existing preview/dependency/package regression selection passes 28 tests.

This proves native builds and host extraction, not an installed Android backend.
Each job still opens its own demuxer/decoder; warm context/index reuse remains
future work. CoreELEC retains the executable backend and would need its own Linux
ARM toolchain/package for this helper. Real Kodi loading, client/VFS routing,
playback integration, HDR colour, CPU/RAM and device latency remain open gates.
See [the component guide](../companion/vfs.freebird.preview/README.md) for protocol,
reproducible commands, pins, exact limits and device-validation steps.
