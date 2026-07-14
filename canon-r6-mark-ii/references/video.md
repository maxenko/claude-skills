# Canon EOS R6 Mark II — Video / Movie Reference

Numbers verified against Canon's official manual/specs, DPReview, and CineD. Grep `r6mii-manual.txt` for menu paths and the recording-mode appendix before quoting page numbers.

## Recording matrix (internal, MP4)
| Mode | Frame rates | Bitrate IPB / IPB Light |
|---|---|---|
| 4K UHD 3840×2160 | 59.94 / 50.00 | 230 / 120 Mbps |
| 4K UHD 3840×2160 | 29.97 / 25.00 / 23.98 | 120 / 60 Mbps |
| 4K UHD time-lapse | 29.97 / 25.00 | ALL-I 470 Mbps |
| 4K UHD Cropped (APS-C/S35) | same rates | same as full-width 4K |
| FHD HFR 1920×1080 | 179.82 / 150.00 | 180 / 105 Mbps |
| FHD HFR 1920×1080 | 119.88 / 100.00 | 120 / 70 Mbps |
| FHD 1920×1080 | 59.94 / 50.00 | 60 / 35 Mbps |
| FHD 1920×1080 | 29.97 / 25.00 / 23.98 | 30 / 12 Mbps |
| FHD time-lapse | 29.97 / 25.00 | ALL-I 90 Mbps |

- Selectable standard rates: **23.98, 25.00, 29.97, 50.00, 59.94**. A flat **24.00p** is not a documented separate option (only 23.98). HFR uses 100/119.88 and 150/179.82.
- **Oversampling:** **4K ≤30p is oversampled (downsampled from ~6K full width) = sharpest.** **4K 50/60p is full-width but subsampled/line-skipped = softer, more aliasing.** Both uncropped. See `quirks.md` #1 — this is the key non-intuitive point.
- **4K Cropped** uses the APS-C/Super35 region (~1.6× crop), pixel-level read (not oversampled from full width).

## Codecs / color / bit depth
- **H.264 (8-bit 4:2:0)** when Canon Log 3 **and** HDR PQ are both **Off**.
- **H.265/HEVC (10-bit 4:2:2)** when **Canon Log 3 on OR HDR PQ on**. Time-lapse uses ALL-I.
- So **10-bit internal recording is gated behind Canon Log 3 or HDR PQ** — a normal picture profile records 8-bit only.
- **Canon Log 3 only** (no Canon Log 2). Color spaces: BT.709, BT.2020, Cinema Gamut. Movie ISO in Log 3: **800–25600**.
- **HDR PQ** (BT.2100 PQ) for in-camera HDR / 10-bit HDR-display delivery.

## External output (micro-HDMI Type D)
- **6K up to 59.94p RAW** to a compatible external recorder (Atomos Ninja V+) as **ProRes RAW**; also **3.7K RAW with a Super35 crop**.
- **HDMI RAW output is locked to Canon Log 3** color (cannot change).
- The headline external path is the **6K/3.7K ProRes RAW** workflow; treat any claim of clean ≥4K non-RAW HDMI to a recorder as needing confirmation (Canon's spec lists HDMI display as Auto/1080p). Port is **micro-HDMI — fragile under strain; clamp it.**
- Frame grab from 4K is **not possible from Canon Log 3 movies** (manual p. 1076).

## High frame rate / slow motion
- **FHD up to 179.82 fps**; 4K up to 59.94p (standard speed, not a dedicated slow-mo mode).
- **AF works in FHD HFR** (Dual Pixel CMOS AF II) — a notable improvement.
- **Audio is not recorded in HFR modes.**

## Recording limits & thermal
- **No 30-minute clip limit** (removed). Auto-stop at **6 hr**; HFR caps at 1 hr 30 min (100/119.88) / 1 hr (150/179.82).
- Canon-rated record times at 23 °C: **4K30 ≈ unlimited**, **4K60 ≈ 40 min** (full width) / **≈50 min** (APS-C crop), **FHD180 ≈ 60 min**. DPReview TV ran 4K60 over an hour at room temp — limits are conservative.
- Heat managed via **[Overheat control]** and **[Standby: Low res.]** — **no** "Auto power off temp (Standard/High)" item on this body (see `quirks.md` #2).
- Card requirements: 8-bit 4K/HFR need **U3**; 10-bit (Log 3 / HDR PQ) need **V60+**. Slow cards stop recording — low-level format if read/write seems slow.

## Stabilization for video
- **IBIS** (up to 8 stops, lens-dependent) + **Movie Digital IS**: Off / On (Standard) / Enhanced. Digital IS adds a **crop** (Enhanced = larger crop) and a small electronic-correction quality cost. Coordinated control with RF IS lenses.

## Audio
- Built-in stereo mic; **3.5 mm mic in**; **3.5 mm headphone out** (level-adjustable).
- **Multi-function shoe** supports cable-free **digital audio** via **Tascam CA-XLR2d-C** (bus-powered through the shoe, XLR + 48 V phantom). Avoids analog A/D loss.

## Monitoring / assist tools
- **Zebras**, **focus peaking**, **focus guide**, **aspect markers**, **false color** (6-color). Limitation: false color **can't combine with View Assist / peaking / zebra and can't be assigned to a button** (see `quirks.md` #9).
- **View Assist** applies a viewing LUT over Canon Log 3 so the image isn't flat/dark on screen without baking it in.

## Movie AF
- **Movie Servo AF** with subject detection (people/animals/vehicles), works including FHD HFR.
- **Detect Only** holds focus where a tracked subject left the frame instead of racking to the background — fixes the classic "AF jumps to background when subject exits" problem.

## Quick recipes
- **Best-quality interview / narrative:** 4K **24/25/30** (oversampled) → Canon Log 3 (10-bit H.265) → View Assist on → V60+ card. Unlimited record time.
- **Slow motion:** FHD **120/180p** (no audio) for buttery slow-mo, or 4K60 (softer, has audio) for milder slow-mo with detail-critical framing.
- **Run-and-gun handheld:** IBIS on + Movie Digital IS Standard (accept the crop), stabilized RF lens, Movie Servo AF + subject detection.
- **External HDR/RAW master:** 6K ProRes RAW to Atomos over HDMI (locked to C-Log 3); clamp the micro-HDMI.
