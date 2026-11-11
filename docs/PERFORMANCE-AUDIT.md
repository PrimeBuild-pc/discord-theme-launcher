# Performance audit — 2026-11-11

What this repo does: It builds server-scoped CSS themes and a Windows editor that installs them through Vencord. Discord receives CSS and embedded artwork/fonts; the editor runs a separate WebView2 process tree. Expected load: one editor, one active personal configuration, frequent edits and large imported images, alongside a single local Discord client.

## Must fix

1. **The editor kept a history of response buffers** (`launcher/Windows/StudioSurface.cs`, resource handler; `launcher/Windows/BuilderPreview.cs`).
   - **What this is:** WebView2 reads local UI, fonts, artwork and preview documents from managed memory streams.
   - **Problem:** The Studio's static-response list grew with every request and released buffers only on close. Preview buffers also depended on a later readiness message or navigation completion to clear their lists.
   - **Fix:** Removed the host response histories in both hosts. WebView2 keeps each managed stream alive while consuming it; after native references are released the garbage collector can reclaim it. These streams contain managed byte arrays and own no file handles. They are not disposed prematurely during an asynchronous read. This does not change the strict local resource allowlist.
   - **If we skip it:** Repeated resource loads retain earlier buffers for the entire editor session. In the native regression, 40 distinct requests for the bundled 569,208-byte font retained 22,771,928 bytes with the old handler. With the fix the measured managed-heap delta after collection was -296 bytes, effectively no growth. This is an editor-host test, not a Discord measurement.

## Should fix

2. **Compilation repeatedly read, hashed and copied embedded resources** (`launcher/Core/ThemeCompiler.cs`).
   - **What this is:** Each valid edit compiles personal CSS on the UI thread.
   - **Problem:** A compilation repeatedly parsed the same catalog and reread/verified the same immutable presets. Sequential replacements copied embedded image text, including newly inserted custom images, through subsequent rewrites.
   - **Fix:** Cache the three verified embedded presets and parsed catalog, remove image payloads from the working CSS before palette/font rewrites, and insert imported payloads at the end. Return a copy of the public catalog array and keep validation/hashing of mutable imported assets on every compile.
   - **If we skip it:** The editor creates unnecessary temporary objects and adds garbage collection pressure during editing. The cache has a fixed three-preset limit; no user configuration or imported asset is cached globally.

3. **Compressed image limits allowed large decoded textures** (`launcher/Windows/AssetImport.cs`).
   - **What this is:** Imported images become embedded PNG artwork in Discord and the preview.
   - **Problem:** A small compressed 4000 × 4000 image passes the existing limits but requires roughly 61 MiB for one 32-bit pixel buffer. Scaling its CSS display size does not reduce its intrinsic pixels.
   - **Fix:** Downscale new imports with high quality interpolation, preserving aspect ratio and alpha, to at most 1920 px per side and 2,073,600 total pixels. Smaller images are never enlarged. Input size/format/dimension checks and normalized PNG validation stay in place.
   - **If we skip it:** Large banners/backgrounds carry avoidable decoding and texture costs in both applications. The new maximum pixel buffer is about 7.9 MiB, approximately 87% below the old maximum. This is a pixel-buffer calculation, not measured total process savings. Existing shared configurations remain byte-preserving and need re-import to benefit. Very large/high-DPI displays may show less artwork detail.

## Nice to have

4. **Narrow chats retained an invisible masked decoration box** (`src/server-theme.css`).
   - **What this is:** The Prime Build board-photo decoration appears only when the chat container is at least 720 px wide.
   - **Problem:** Below that threshold the pseudo-element still had a box, background and mask, with only its opacity set to zero.
   - **Fix:** Use `display: none` below the existing threshold and `display: block` above it. Regenerate all three theme files, catalog sizes/digests and checksums.
   - **If we skip it:** The browser still needs to account for the invisible box. No CPU/GPU or VRAM saving is claimed for this small change; the browser may already optimize transparent drawing. Browser tests verify both sides of the width threshold.

## Measurements and checks

Release .NET builds on the same Windows host, three warm-up calls then 20 synchronous compilations per case. Allocations use `GC.GetAllocatedBytesForCurrentThread`; timings use `Stopwatch`. The baseline was the pre-audit working tree, including its existing uncommitted compatibility changes. The custom workload uses two synthetic 2 MiB PNG-signature payloads accepted by Core; it measures string/compilation work, not image decoding or WebView2.

| Workload | Before | After | Change |
| --- | ---: | ---: | ---: |
| Default preset: allocated per compile | 26.10 MiB | 4.07 MiB | -84% |
| Default preset: time per compile | 5.20 ms | 0.96 ms | -82% |
| Two 2 MiB imported payloads: allocated per compile | 135.07 MiB | 55.48 MiB | -59% |
| Two 2 MiB imported payloads: time per compile | 21.72 ms | 10.27 ms | -53% |

Timings are a local sample and vary by machine and concurrent load. Allocations are cumulative temporary allocation volume per compile, not retained RAM or whole-process working set. Before rebuilding the changed decoration CSS, compiled SHA-256 outputs were identical across all 60 preset/palette/bundled-or-native-font combinations and one custom-asset workload.

- `npm run verify`: 64 Core checks and 46 Playwright browser tests passed, including checksums, exact server isolation, offline fonts/artwork, custom-image readability and the new photo-width regression.
- `dotnet run --project launcher/WindowsTests/WindowsTests.csproj -c Release`: 45 native checks passed. New cases cover square/landscape/portrait reductions, alpha/aspect retention, no upscaling, and repeated-response retention. Existing real WebView2 large-preview, rapid-edit, language, confirmation and navigation-blocking checks passed.
- The old Studio response handler was restored temporarily for the new native regression: it failed specifically on retained response bytes. The corrected source was restored afterward.
- A full Release rebuild completed with zero warnings/errors, the final corrected native suite passed again, and `scripts/package-windows.ps1` produced the Windows x64 EXE, portable ZIP and checksums successfully.
- No production dependency was added. No Discord profile, hardware-acceleration preference, theme enablement or running client was modified by these tests.

## Discord RAM, CPU and hardware acceleration

These findings explain avoidable work in this project. They do not establish why a particular Discord session uses several gigabytes. The theme contains no message cache, runtime JavaScript, timer loop, blur or continuously animated wallpaper. The editor preview exists only while its host is open. Discord startup observation is limited to 40 seconds with a 250 ms asynchronous delay, rather than an idle background poll.

Use the existing **no artwork** choices and **native typography** to compare a lighter configuration. For an attributable measurement, close the editor, use the same Discord version/server/channel/window size and activity, restart between conditions, and compare the entire Discord process tree with themes disabled and enabled. Record private bytes, working set, CPU and GPU memory after warm-up and after the same navigation/scroll workload. Repeat the comparison to separate startup/cache noise from persistent growth; test calls, streams and media separately.

The launcher does not force GPU flags. Hardware acceleration is a separate Discord setting; changing it is a troubleshooting comparison rather than a guaranteed RAM reduction. Discord's [official troubleshooting guide](https://support.discord.com/hc/en-us/articles/31623498041623-Discord-Troubleshooting-Guide) documents disabling it for troubleshooting. The WebView2 stream read/disposal boundary is documented in [Microsoft's local-content guidance](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/working-with-local-content#example-of-handling-the-webresourcerequested-event); unlike the file-stream example, these responses own only managed byte buffers.

Verdict: The demonstrated editor retention and repeated compilation allocations are fixed. New imports also bound artwork pixel costs in Discord. The reported multi-gigabyte Discord usage still requires a controlled live-client comparison.

Lean: removed three response-history collections; zero new production dependencies. Performance fixes reuse the existing compiler, importer and renderer.

Not checked: controlled live Discord working-set/private-memory/CPU/GPU traces, game frame times, long call/stream sessions, minimized-WebView suspension, clean-VM setup, third-party plugins and Discord internals. This is a performance-focused audit, not a full security or architecture certification.
