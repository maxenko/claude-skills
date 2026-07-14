# Canon EOS R6 Mark II — Spec Sheet (fast lookup)

Announced 2 Nov 2022. Bundled manual documents **firmware ver. 1.7.0** (doc CT2-D244-F). All page numbers below are the manual's own page numbers; grep `r6mii-manual.txt` to confirm before quoting.

## Body & sensor
- **Sensor:** 24.2 MP effective full-frame CMOS (~36.0 × 24.0 mm), 6000×4000 native, **non-stacked, non-BSI** front-side-illuminated. DIGIC X. (manual p. 1060–1061)
- **No dual base ISO / no Dual Gain Output.** Single conversion-gain readout. Do not claim cinema-body DGO behavior. The shadow read-noise drop near ISO 400 is dual-*conversion*-gain, a different thing (see `quirks.md`).
- **Mount:** RF (RF/RF-S native; EF/EF-S via Canon mount adapter).
- **Cards:** **Dual SD/SDHC/SDXC, both slots UHS-II.** **No CFexpress.** (p. 1060)
- **Viewfinder:** 3.69M-dot OLED, 0.76×, 100%. **Screen:** 3.0" 1.62M-dot fully articulating touch. (p. 1069)
- **No top status LCD.** Settings are read off the rear screen / EVF only.
- **Ports:** USB-C (USB 3.2 Gen 2 / SuperSpeed Plus), **micro-HDMI Type D** (Auto/1080p, no CEC), 3.5 mm mic in, 3.5 mm headphone out, multi-function (digital) shoe. (p. 1069, 1077)
- **Battery:** LP-E6P / LP-E6NH / LP-E6N / LP-E6. USB-PD via PD-E1 (or compatible PD source). (p. 1078)
- **Weight:** ~670 g with battery+card (588 g body). **Operating temp 0–40 °C.** (p. 1079–1080)

## ISO
- **Stills:** ISO 100–102400 normal; expanded **L = 50, H = 204800**. Highlight Tone Priority limits to 200–102400. Expanded ISO unavailable in HDR / HDR PQ. (p. 1070)
- **Movie (M, Log off):** 100–25600. **Canon Log 3 on: 800–25600** (L equiv ~100–640, H to 204800). (p. 1072)

## Shutter & sync (p. 1074; flash sync confirmed Canon KB ART182185)
| Mode | Speed range | Flash X-sync | RAW depth |
|---|---|---|---|
| Mechanical | 1/8000–30 s, Bulb | **1/200 s** | 14-bit |
| Electronic 1st-curtain (EFCS) | 1/8000–30 s, Bulb | **1/250 s** | 14-bit |
| Electronic (full) | **1/16000**–30 s, Bulb | **none** | **12-bit** |

Note the counter-intuitive bit: **EFCS syncs flash *faster* (1/250) than full mechanical (1/200).** 1/16000 is Tv/M only (1/8000 in Fv/P/Av), and electronic caps at 1/8000 in Hi-speed continuous+, RAW burst, HDR, and focus bracketing.

## Drive / continuous (p. 1075; detail p. 549)
| Drive | Mechanical | EFCS | Electronic |
|---|---|---|---|
| High-speed continuous **+** | 12 fps | 12 fps | **40 fps** (12-bit) |
| High-speed continuous | 5.5 fps | 7.0 fps | 20 fps |
| Low-speed continuous | 3.0 fps | 3.0 fps | 5.0 fps |

- **40 fps is lens-gated** — only specific RF lenses sustain it with Servo AF; verify on Canon's "Lenses Supporting Maximum High-Speed Continuous Shooting" supplement before relying on it. Rate also degrades with low battery/temperature, flicker, slow shutter, small aperture, dim subjects.
- **Buffer (dual UHS-II):** RAW ~85 (standard card) / ~110+ (fast UHS-II) per manual p. 1063; real-world ~240 RAW / ~1000+ JPEG on fast V90. Buffer-clear time is strongly card-dependent.

## Autofocus
- **Dual Pixel CMOS AF II, 1053 zones**, ~100% × 100% coverage. Low-light **−6.5 EV** (f/1.2, center, One-Shot).
- **Subject detection:** Auto / People (eye/face/head/body) / Animals (dogs, cats, birds, horses) / Vehicles (cars, motorcycles, +trains, +aircraft) / None. Left/right/auto eye selection.
- **AF area:** Spot, 1-point, Expand (4/8), Flexible Zone 1/2/3, Whole-area. Menu: [AF] → [AF area] (p. 488), [Subject to detect] (p. 491), [Eye detection] (p. 496).

## Image stabilization
- **IBIS up to 8.0 stops CIPA, lens-dependent** (8-stop figure measured with RF 24-105 F4L IS at 105 mm). Coordinated body+lens IS on RF IS lenses; body-only on non-IS lenses.
- **With a stabilized RF lens, IBIS is governed by the lens's physical IS switch** — the menu IS-mode toggle effectively applies to non-IS lenses. Lens switch off → IBIS off.

## Video — see `video.md` for the full matrix. Headlines:
- **4K UHD up to 60p, full sensor width, NO crop** (improvement over original R6's 1.07× crop). **4K ≤30p is oversampled from 6K (sharpest); 4K 50/60p is subsampled/line-skipped (softer, more aliasing)** — both full-width. See `quirks.md`.
- **FHD up to 180 fps** (HFR; no audio in HFR). 4K-Cropped (APS-C/Super35) mode also available.
- **6K (and 3.7K S35) ProRes RAW out over micro-HDMI** to Atomos; **locked to Canon Log 3** color.
- **Codecs:** H.264 8-bit 4:2:0 (standard); **H.265/HEVC 10-bit 4:2:2** when Canon Log 3 **or** HDR PQ is on. MP4 container.
- **Canon Log 3 only — no Canon Log 2 on this body.** HDR PQ available.
- **No 30-minute clip limit.** Thermal: 4K30 ≈ unlimited (≤6 hr cap), 4K60 ≈ 40 min (FF) / 50 min (crop), FHD180 ≈ 60 min. Heat managed via **[Overheat control]** and **[Standby: Low res.]** — there is **no** R5-style "Auto power off temperature (Standard/High)" item on the R6 II (manual p. 1052, 1041).

## Card speed requirements (manual p. 1066)
- 4K and FHD HFR (8-bit): **UHS Speed Class 3 (U3) or higher**.
- 10-bit 4K / HFR (Canon Log 3 or HDR PQ): **Video Speed Class V60 or higher**.

## Sources
Canon manual (`r6mii-manual.txt`), Canon USA KB (ART182215 specs, ART182185 flash sync, ART182175 shutter), DPReview in-depth review, PhotonsToPhotos (PDR), The-Digital-Picture (readout), Atomos compatibility, Tascam CA-XLR2d list.
