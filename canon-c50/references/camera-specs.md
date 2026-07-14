# Canon EOS C50 — Quick Spec Reference

One-screen lookup. Source: Canon official specs + the bundled manual (`c50-manual.pdf`, "Specifications" appendix starts p. 240). Cross-check the manual for any spec you're about to quote in a binding way (delivery commitments, rental tech sheets).

## Body & Mount
| | |
|---|---|
| Release | September 2025 |
| Mount | Canon RF (full-frame native) |
| EVF/LCD | 3.5" LCD touchscreen, articulating |
| Card slots | CFexpress Type B (slot 1) + SD UHS-II (slot 2) |
| Audio in | Internal stereo mic + 3.5mm; dual XLR via included handle top |
| Outputs | HDMI Type A, SDI via accessory, USB-C |
| Networking | Wi-Fi (2.4/5 GHz), Ethernet via adapter, Frame.io C2C, XC protocol |
| Weight | ~670 g body only (Canon's smallest Cinema EOS) |
| Assignable buttons | 14 |

## Sensor
| | |
|---|---|
| Type | 7K full-frame CMOS (newly developed for C50) |
| Effective pixels | 7K (video) / 32 MP (stills) |
| Dynamic range | Canon-stated **15+ stops Full Frame, 16 stops Super 35** (the extra stop is crop-mode processing, not sensitivity — see "Sensor modes" below); ≈14 in C-Log 3. CineD lab numbers run lower (~10 stops FF / ~11 S35 @ SNR2). |
| Base ISO | 800 and 6400 (Dual Base ISO with Auto option) |
| ISO range | Expandable; see manual p. 78 |
| AF | Dual Pixel CMOS AF II + subject detection (people / animals / vehicles) |
| Architecture | **Non-stacked, non-BSI** CMOS, and **no DGO (Dual Gain Output)** — unlike the C70 (DGO) or stacked C80/C400. This is *why* DR and rolling shutter are middling: it's the tradeoff for the small body and price. Dual Base ISO is the simpler DR tool. |
| Rolling shutter | ~14.3 ms in standard 17:9 FF/S35 modes; **~18.1 ms in 7K 3:2 Open Gate** (worst — watch skew on whip-pans/strobes/fast action); **7.1 ms at 4K120** (fast, because it subsamples). Moderate overall, not a global shutter. |

## Sensor modes — resolution ceiling & the Super 35 DR quirk

The C50 has four sensor modes. **Resolution scales with how much of the sensor you read**, so cropping costs resolution — but on the C50 it *gains* dynamic range (see below).

| Sensor mode | Reads | Max resolution | Notes |
|---|---|---|---|
| Full Frame 3:2 | Whole sensor, 3:2 | 6960×4640 (~7K) | Open Gate; sub recording disabled (manual p. 65) |
| Full Frame | Whole sensor, 17:9/16:9 | 6912×4608 (~7K) | The only way to hit headline 7K |
| **Super 35mm (Cropped)** | S35 center | **4096×2160 / 3840×2160 (4K)** | ~1.46× crop; the mode sub-frame lenses must use |
| **Super 16mm (Cropped)** | S16 center | 2K | Deep crop, niche |

Verified against the manual recording-config tables (RAW table p. ~2742 in `c50-manual.txt`; XF-AVC tables p. ~2890 show S35 capping at 4096×2160 vs Full Frame's 6912×4608).

### Frame-rate crops & Open Gate caps (don't get caught out)
- **4K120: NO extra crop** — it stays full-width (uses subsampling, hence the fast 7.1 ms readout). Unusual for a full-frame body; many cameras crop for 120p, the C50 doesn't.
- **2K180: ~12% narrower** angle of view (a small additional crop). Factor this into lens choice for high-speed work.
- **7K 3:2 Open Gate RAW caps at 30p**, not 60p. The headline "7K60 RAW" is the 17:9 Full Frame readout; the full 6960×4640 Open Gate in Cinema RAW Light tops out at 29.97p (60p only in compressed codecs). Verified in the manual RAW table (`c50-manual.txt` ~p. 2750: the 59.94P column is blank for 6960×4640).

### Lens image circle vs sensor mode
- **Full-frame RF glass** → any mode. Full Frame for max resolution; crop modes for reach/DR.
- **APS-C / Super-35 image-circle lens** (7artisans APS-C cine, RF-S, etc.) → **must** run **Super 35mm (Cropped)**. Full Frame / FF 3:2 will **vignette** (lens can't fill the sensor — manual p. 34 explicitly warns of this). Practical ceiling with such glass is therefore **4K**, by physics, not defect.
- **EF-EOS R 0.71× focal reducer** → squeezes a *full-frame* EF lens's circle onto the S35 readout: restores FF angle of view AND gains ~1 real stop of *light*. This is the only "+1 stop of exposure" path — it does NOT apply to native APS-C lenses (manual p. 34).

### The Super 35 "+1 stop" — it's dynamic range, not exposure
This trips people up. Canon and CineD both confirm crop mode measures ~1 stop **more dynamic range** than Full Frame:

| Mode (ISO 800, Cinema RAW Light) | DR @ SNR 2 | DR @ SNR 1 | Canon spec |
|---|---|---|---|
| Full Frame 7K | ≈9.98 stops | ≈11.2 stops | 15+ stops |
| **Super 35 5K** | **≈11 stops** | **≈12.3 stops** | **16 stops** |

- **Mechanism:** not sensor sensitivity — Canon applies different/heavier shadow-noise processing in crop mode (measured shadow noise ≈1.5–2% S35 vs ≈3% FF), likely costing a touch of fine detail. CineD notes pixel-to-pixel RAW DR "should" be identical when cropping, so this is processing, not optics.
- **What it does NOT mean:** it does **not** brighten the image, let you drop ISO, or let you stop down. T-stop and exposure are identical in every mode. The benefit is cleaner shadows / more grading latitude in the lift.
- **Free bonus with sub-frame glass:** since APS-C lenses force Super 35 mode anyway, you're parked in the C50's better-DR mode for free — handy, since the C50's DR is otherwise middling (CineD rates it below the C80/C400 and R5 II).

Sources: [CineD lab test](https://www.cined.com/canon-eos-c50-lab-test-rolling-shutter-dynamic-range-and-exposure-latitude/) · [Newsshooter Q&A](https://www.newsshooter.com/2025/10/16/canon-c50-overview-qa/) · [Canon Europe specs](https://www.canon-europe.com/video-cameras/eos-c50/specifications/).

## Recording — codecs and resolutions
| Codec | Max | Notes |
|---|---|---|
| Cinema RAW Light HQ | 7K60 12-bit | ~2900 Mbps peak; requires CFexpress VPG 400 |
| Cinema RAW Light ST | 7K60 12-bit | Lighter than HQ |
| Cinema RAW Light LT | 7K60 12-bit | Smallest RAW variant |
| XF-AVC | 4K120 10-bit 4:2:2 | Mainline broadcast/cine codec |
| XF-AVC S | 4K to 120 | Simpler file structure, NLE friendly |
| XF-HEVC S | 4K to 120 | H.265, smallest files of the non-RAW group |
| 3:2 Open Gate | 4K120 | Full sensor area for reframing |
| Slow & Fast | 4K120, 2K180 | See manual p. 124 and p. 214 |

## Sub recording / proxies
- Simultaneous 2K crop record (manual p. 71)
- Proxy clips to SD slot (manual p. 69)
- Chunk recording for automatic transfer (manual p. 73)

## Color science
| Mode | Use |
|---|---|
| Canon Log 2 + Cinema Gamut | Max DR; narrative / VFX / ACES |
| Canon Log 3 + BT.2020 (or 709) | Faster grading; doc / broadcast |
| Wide DR | Out-of-camera look with extra latitude |
| BT.709 | Direct delivery, no grade |
| PQ / HLG | HDR delivery direct |

ACES workflow: IDT provided — see Workflow Overview p. 19.

## Monitoring tools (all on-body)
- Waveform monitor (luma / RGB / Y' Pb' Pr')
- Vectorscope
- False color
- Zebras (single + dual threshold)
- Peaking (red/blue/yellow, configurable)
- Magnification (focus assist)
- Anamorphic desqueeze: 1.3× / 1.5× / 1.8× / 2.0× (manual p. 128)
- LUT to LCD/HDMI without baking (Look Files p. 137)

## Power
- LP-E6P battery (supplied) — same family as R5 / R5 C
- DC IN via cable + DC coupler
- USB-C power adapter supported (manual p. 24)

## Notable gaps (be honest about these)
- **No internal ND filter.** Plan variable ND (77/82mm typical) or matte box.
- **No IBIS** (in-body image stabilization). Handheld stability comes from lens IS, a gimbal, or your shoulders.
- **No EVF.** Articulating LCD only — plan a hood/loupe or external monitor for bright daylight.
- **No SDI on body.** Requires accessory.
- **Single CFexpress slot.** Dual record means RAW to CFexpress + proxy to SD only, not RAW + RAW.
- **No internal timecode out without TC terminal use** — has TC in/out on the body (p. 106), but no genlock.
- **Hot shoe is electronic but cannot trigger a flash.** Multi-Function Shoe is for audio/data accessories (mics, XC controllers), not strobes.
- **Electronic IS is the only in-camera stabilization, and it crops the frame.** With no IBIS, turning on Electronic IS eats into your angle of view (and can't save heavy handheld shake). Plan stabilized RF glass, a gimbal, or a rig instead of leaning on EIS.
- **Thermal: fan-cooled, but not unlimited.** Continuous 7K60 RAW runs ~60+ min before thermal management becomes a concern (fan-assisted, better airflow than the R5 C, but it's still a heat-managed body). For long lockoffs in heat, monitor temp and consider a lighter codec.
- **Only 2 XLR channels** (via the detachable handle) + the internal mic. No 4-channel audio. Plan your mix accordingly (e.g., lav + boom, no room for a second boom on a separate channel).

## Firmware
Current shipping firmware is **1.0.3.1** (Feb 27, 2026). Two real-world bugs fixed in this release — see the "Firmware awareness" section of SKILL.md. Summer 2026 firmware (announced at NAB 2026) adds playback view-assist, USB-C gimbal control, and SRT auto-reconnect. Check Canon support for current version.

## Compatible lens notes (manual p. 246)
- All RF mount lenses supported with appropriate features per lens.
- EF lenses via Canon EF-EOS R adapter (also EF-EOS R 0.71× for S35 framing of full-frame coverage).
- Cinema RF prime lenses (CN-R series) recommended for narrative; consumer/L hybrid lenses (the new RF85 F1.4 L VCM hybrid line) work well for one-person crews.
