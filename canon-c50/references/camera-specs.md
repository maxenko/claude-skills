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
| Dynamic range | 15+ stops Canon-stated (≈16 at base ISO 800 C-Log 2; ≈14 in C-Log 3) |
| Base ISO | 800 and 6400 (Dual Base ISO with Auto option) |
| ISO range | Expandable; see manual p. 78 |
| AF | Dual Pixel CMOS AF II + subject detection (people / animals / vehicles) |

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

## Firmware
Current shipping firmware is **1.0.3.1** (Feb 27, 2026). Two real-world bugs fixed in this release — see the "Firmware awareness" section of SKILL.md. Summer 2026 firmware (announced at NAB 2026) adds playback view-assist, USB-C gimbal control, and SRT auto-reconnect. Check Canon support for current version.

## Compatible lens notes (manual p. 246)
- All RF mount lenses supported with appropriate features per lens.
- EF lenses via Canon EF-EOS R adapter (also EF-EOS R 0.71× for S35 framing of full-frame coverage).
- Cinema RF prime lenses (CN-R series) recommended for narrative; consumer/L hybrid lenses (the new RF85 F1.4 L VCM hybrid line) work well for one-person crews.
