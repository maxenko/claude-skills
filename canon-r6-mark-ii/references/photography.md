# Canon EOS R6 Mark II — Stills / Photography Reference

24.2 MP full-frame, DIGIC X, Dual Pixel CMOS AF II, dual SD UHS-II. Figures cross-checked against Canon docs, DPReview, The-Digital-Picture, PhotonsToPhotos.

## Continuous shooting
- **Mechanical / EFCS: 12 fps** (full 14-bit RAW, Servo AF/AE) — the everyday "safe" high-speed mode.
- **Electronic: up to 40 fps** (12-bit, silent), with intermediate 20/5 fps e-shutter steps.
- **40 fps is lens-gated** — only specific RF lenses sustain it with Servo AF (RF 14-35 F4L, 15-35 F2.8L, 24-70 F2.8L, 24-105 F4L, 70-200 F2.8L/F4L, 24 F1.8 Macro, etc.). Many STM / third-party / adapted EF lenses fall short. **Check Canon's "Lenses Supporting Maximum High-Speed Continuous Shooting" supplement.** Rate also degrades with low battery/temp, flicker, slow shutter, small aperture, dim/low-contrast subjects.
- **Buffer (dual UHS-II):** ~240 RAW / ~1000+ JPEG on fast V90 cards (manual lists ~85–110 RAW depending on card). Buffer-clear time is strongly card-dependent.

## RAW Burst mode + pre-shooting
- A distinct drive mode: **30 fps, 12-bit RAW**, up to ~191 frames, written as **one .CR3 "roll"**.
- **Pre-shooting** retains ~**0.5 s (~15–20 frames) before** the full press — miss-insurance for unpredictable action.
- Extract frames in-camera or in **DPP** (reliable; Lightroom historically couldn't open the roll). 24 MP per frame, 12-bit. See `quirks.md` #10.

## Autofocus
- **Dual Pixel CMOS AF II, 1053 zones**, ~100% coverage, **−6.5 EV** low-light (f/1.2, center, One-Shot).
- **Subject detection:** Auto / People (eye/face/head/body) / Animals (dogs, cats, birds, horses) / Vehicles (cars, motorcycles, trains, aircraft). Left/right/auto eye selection. Registered-person priority to stick to one subject in a crowd.
- **AF area:** Spot, 1-point, Expand (4/8), Flexible Zone 1/2/3 (user-sizable), Whole-area. Servo AF cases 1–4 + A (tracking sensitivity, accel/decel).

## Dynamic range & image quality
- **Base ISO 100** (expand L50 / H204800). **Measured PDR ≈ 11.5 stops** at base (PhotonsToPhotos) — solid, not class-leading.
- **Not fully ISO-invariant at base:** ISO 100 deep shadows are noisier than ISO ~400; the gap closes by ~ISO 400 (dual-*conversion*-gain, not dual base ISO). **For dark scenes you'll lift hard, expose at ISO 400+ rather than ISO 100-and-push.** See `quirks.md` #7.
- **24 MP tradeoff:** less crop latitude/fine detail than 45 MP bodies, but better high-ISO noise, larger buffer, smaller files, faster readout — a deliberate speed/low-light choice.
- Aggressive 4–5 stop lifts can show faint shadow banding, more in 12-bit files (e-shutter/40 fps/RAW Burst) than 14-bit mechanical.

## Shutter modes (see `camera-specs.md` for the table)
- **Mechanical** (X-sync 1/200) — geometry-critical fast panning; flicker-heavy rooms where you want a physical shutter.
- **EFCS** (X-sync **1/250**, *faster* than mechanical) — quiet default, fastest flash sync, no shutter-shock at most speeds.
- **Electronic** (1/16000, **no flash**, 12-bit) — silent, 40 fps, ultra-fast shutter; rolling-shutter skew + LED/flash banding (`quirks.md` #3–4).

## Image stabilization
- **Up to 8 stops CIPA, lens-dependent** (8-stop figure: RF 24-105 F4L at 105 mm). Coordinated body+lens IS; body-only on non-IS lenses. With an RF IS lens, **IBIS follows the lens IS switch** (`quirks.md` #12).

## Flicker handling
- **Anti-flicker** = even frame-to-frame brightness under 100/120 Hz mains light (mech/EFCS).
- **High-Frequency Anti-Flicker** = kills intra-frame banding from PWM/high-frequency LED (50.0–2011.2 Hz); **Tv/M only**, works with e-shutter. Pick the right one — they don't overlap. See `quirks.md` #5.

## Special features
- **Focus bracketing + in-camera depth composite:** 2–999 frames, exposure smoothing, saves merged + sources. **E-shutter, no flash, no MF**; use f/2.8–f/4 so frames are distinct; fails on flat/repetitive subjects (`quirks.md` #11).
- **Multiple exposure** (additive/average/bright/dark); **in-camera HDR**; **HEIF (HDR PQ)** 10-bit stills.
- **Panoramic Shot Mode** — native sweep panorama, **JPEG only** (no RAW); commit exposure/WB in-camera.
- **Interval timer** and **bulb timer** built in.

## Photographer gotchas (confirmed) — all detailed in `quirks.md`
1. 40 fps is lens-gated. 2. E-shutter skews fast horizontal motion → use 12 fps mechanical for sports geometry. 3. 40 fps / RAW Burst / e-shutter = 12-bit; best file is mechanical ISO 100 14-bit. 4. EFCS syncs flash faster (1/250) than mechanical (1/200); no flash in full e-shutter. 5. Dedicated still/movie switch swaps the whole UI. 6. No top LCD. 7. Mode dial center lock. 8. Backup-writing to both SD cards slows buffer clear; use V90. 9. HF anti-flicker is M/Tv only. 10. Panorama/HDR outputs are JPEG. 11. For clean shadows shoot ISO 400, not ISO-100-and-push.
