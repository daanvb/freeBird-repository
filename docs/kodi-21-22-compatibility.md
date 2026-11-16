# Kodi 21 and 22 compatibility

This pass targets the current freeBird app/player on Kodi 21 Omega and Kodi 22
Piers RC2. Python and mocked API tests verify source behaviour; they do not
certify a Kodi installation, renderer, decoder, remote or Widevine CDM.

## Compatibility policy

- Keep the Python 3.8 runtime floor used by older Kodi 21 builds. Newer Python
  runtimes must not require different saved settings or a separate package.
- Retain `xbmc.python` 3.0.0 and `xbmc.gui` 5.17.0 dependency floors. Kodi 22
  explicitly retains these ABIs in its [Python manifest](https://github.com/xbmc/xbmc/blob/22.0rc2-Piers/addons/xbmc.python/addon.xml)
  and [GUI manifest](https://github.com/xbmc/xbmc/blob/22.0rc2-Piers/addons/xbmc.gui/addon.xml).
- Gate optional features by the API/add-on that implements them. Never infer
  Kodi's release from `xbmc.__version__`, which describes the Python API.

## Changes and inspected boundaries

| Area | Decision |
| --- | --- |
| Audio/subtitle languages | Accept ISO codes and BCP 47 tags on either Kodi version. Normalise bibliographic/terminological aliases. Keep the saved base `language`; add an optional `language_tag` for variants. Prefer an exact variant within eligible tracks, with base-language fallback. Preserve forced/accessibility requirements, manual selections, source identity and the existing English audio-quality policy. |
| Provider subtitles | Keep allowed languages as a set. Preserve variant information for display/preferred ranking and maintain exact-release checks for forced downloads. No extra provider requests. |
| Track pickers | Show region/script tags even if Kodi returns only a base-language display name. A missing/failed conversion falls back to the supplied tag. |
| Keymap writes | Use `Path.open(..., newline=...)`, available in Python 3.8, instead of the newer `Path.write_text(newline=...)` argument. Preserve file names, LF bytes, idempotence and unrelated keymaps. |
| HTTP failures | Handle bodyless `HTTPError` instances on Python 3.8, where response-wrapper methods are not initialised. Preserve error-slate detection, metadata redirect limits and fixed speed-test failure reasons. |
| Up/Down | Preserve freeBird's suppression, scrub preview, chapter menu and modal ownership rules. For playback outside the owned player, use Kodi 22 RC2's scene-aware actions; keep existing fallbacks for Kodi 21, older labelled previews and unknown builds. |
| Live TV / InputStream Adaptive | Use the JSON DRM interface from ISA 22.1.5 onwards. Keep legacy properties on older/unknown ISA versions. Preserve raw challenge/response proxying and single-session behaviour. Unencrypted channels receive no DRM configuration. |
| Metadata and resume | Existing `InfoTagVideo` setters and `setInfo('video', ...)` remain usable on both versions. No need to introduce Kodi 22-only setters or change resume/progress storage. |
| Native controls / skin ABI | Retain XML video windows and XML-managed scrollbars. Do not call Kodi 22-only `ControlVideoWindow` on Kodi 21. Existing strict control, window retirement, reinitialisation and focus tests remain part of validation. |
| HDR / live statistics | Existing guarded info-label reads already expose available HDR details and live queue/bitrate data; absent labels remain unknown/omitted. No inference of FEL processing or output capabilities. |
| Python standard library | Review runtime imports and removed APIs; parse all shipped Python at the 3.8 grammar floor and execute the suite with old/new interpreters. Third-party native add-ons remain supplied by the platform's Kodi repository. |

Sources: [Kodi 22 Python API changes](https://xbmc.github.io/docs.kodi.tv/master/kodi-base/d1/ddd/python_v22.html),
[Kodi 22 keyboard map](https://github.com/xbmc/xbmc/blob/22.0rc2-Piers/system/keymaps/keyboard.xml),
[Kodi 21 keyboard map](https://github.com/xbmc/xbmc/blob/Omega/system/keymaps/keyboard.xml),
[InputStream Adaptive integration](https://github.com/xbmc/inputstream.adaptive/wiki/Integration),
[DRM interface](https://github.com/xbmc/inputstream.adaptive/wiki/Integration-DRM).

Skin-load notifications, CEC power reporting and SVG assets are optional future
features, not prerequisites for compatibility. This pass does not introduce a
new skin-recovery service or change global Kodi/CEC settings. Kodi's own HDR,
subtitle and passthrough fixes require its updated player/platform build.

## Offline verification (16 November 2026)

Run `python tools/run-tests.py` with each interpreter after installing
`requirements-test.txt`. CI now uses Python 3.8 and 3.14 as separate jobs.

| Local Windows runtime | Full-suite result |
| --- | --- |
| Python 3.8.10 / Pillow 10.4.0 | 3,241 passed, 149.741 seconds |
| Python 3.14.0 / Pillow 12.3.0 | 3,241 passed, 153.561 seconds |

The suites include 27 new compatibility regressions, a Python 3.8 grammar check
of all 304 runtime sources, and the existing app/skin/repository packaging tests.
Initial runs reproduced the old keymap and bodyless-HTTP failures; nine of the
twelve new language tests failed before the language changes. Tests/build helpers
were also adjusted to run on Python 3.8 without weakening their assertions.
These local results do not claim a GitHub Actions run or a native Kodi test.

Second review: the original full-suite results above are from `cffd55a1`.
The follow-up restores the audio label's fallback for any ordinary conversion
exception and parses only Kodi's release token, keeping unknown Kodi 22.0
preview/fork suffixes on the legacy actions and ignoring trailing Git metadata.
Both issues were reproduced with failing regression cases before the fixes.
The language compatibility test now runs independently of discovery order.
All 28 compatibility tests pass independently on both runtimes; the broader
focused selection passes **468 tests on Python 3.8.10 (12.808s)** and **468 on
Python 3.14.0 (14.898s)**. It covers player controls/display, subtitles and series
preferences, live TV, keymaps, source/chapter HTTP handling and speed checks.
Version formatting was cross-checked against Kodi's
[SystemInfo implementation](https://github.com/xbmc/xbmc/blob/22.0rc2-Piers/xbmc/utils/SystemInfo.cpp).

## Device acceptance before release

On both Kodi 21 and Kodi 22 RC2, where supported by the device distribution,
check Fire TV Stick, NVIDIA Shield and CoreELEC separately:

1. Cold launch, Settings, Home/details re-entry and skin reload without stale
   focus, stranded dialogs or background publication to retired windows.
2. Start/resume/cancel, manual audio/subtitle selection, forced captions and a
   saved series preference carried between ISO-coded and BCP 47-tagged tracks.
3. Physical Up/Down/OK/Back/long presses, chapter and scrub menus, playback
   outside freeBird, Up Next and live-TV focus/seek restrictions.
4. Unencrypted and Widevine live TV, cancellation, channel changes and stopping;
   verify the installed ISA/CDM versions and actual licence exchange.
5. HDR/Dolby Vision, PGS subtitles, passthrough after seeking and translucent
   OSD rendering. Offline tests cannot verify HDMI/FEL/audio correctness.

No source changes in this pass authorise a release, device update or restart.
