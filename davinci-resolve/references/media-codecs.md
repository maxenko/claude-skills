# Media, Codecs, Free-vs-Studio & Delivery

Reference for codec choices, the edition split, proxy/optimized/cache pipelines, and delivery settings.

## Free vs Studio (decide what's even possible)

DaVinci Resolve 20 (2025) is the current line. Verify current limits at blackmagicdesign.com if a feature is borderline.

| Capability | Free | Studio (~$295 one-time) |
|---|---|---|
| H.264/H.265 decode | **8-bit only** | 8-bit + 10-bit, 4:2:2 HEVC |
| Output resolution | up to **UHD 4K (3840×2160) ~60p** | up to 32K / high fps |
| Multiple GPUs | No (single GPU) | Yes |
| Hardware-accelerated H.264/H.265 **encode** | limited | Yes (NVENC/QuickSync/etc.) |
| ProRes **encode** on Windows | No | Yes |
| Noise Reduction (temporal/spatial) | No | Yes |
| Most AI Neural Engine FX (Magic Mask, Voice Isolation, Face Refinement, Super Scale, Depth Map) | No | Yes |
| Dolby Vision / full HDR tools | No | Yes |
| Resolve FX (full set, ~45+ extra) | subset | full |
| Some AI tools added to free in v20 | IntelliScript, Animated Subtitles, Multicam SmartSwitch | + all of the above |

You can *import/edit/grade* footage above the free limits — you just can't *export* above UHD on free. "Why can't I export 4K above 60p / why is 10-bit greyed out" → free-version limit, needs Studio.

## Codec guidance

**Edit-friendly (smooth playback/scrub) — intra-frame:**
- **DNxHR** (Avid) — cross-platform, free to encode, great proxy/optimized/master codec on Windows. Flavors: LB/SQ/HQ/HQX(10-bit)/444.
- **ProRes** (Apple) — native decode everywhere; encode native on Mac, **Windows encode needs Studio**. Flavors: Proxy/LT/422/422HQ/4444.
- **BRAW** (Blackmagic RAW) — excellent performance for a RAW format; partial-debayer keeps it light.

**Stutter culprits — long-GOP / inter-frame:**
- **H.264 / H.265 (HEVC)** from phones, drones, mirrorless, GoPro, screen/game capture. Heavy to decode; transcode to Optimized Media/Proxies for smooth editing.
- **10-bit 4:2:2 H.264/HEVC** — Studio + hardware decode strongly recommended.
- **VFR (variable frame rate)** — transcode to **CFR** before import (HandBrake) or audio drift is inevitable; relinking does not fix it.

**Camera RAW:** BRAW, R3D (RED), ARRIRAW, Canon Cinema RAW Light, Nikon/Sony RAW — RAW settings (decode quality, ISO, color space) live in the Camera Raw palette/project settings; lower decode quality for editing, full for final.

## Optimized Media vs Proxy vs Render Cache (don't conflate)

| | Transcodes | Stored | Travels with project | Purpose |
|---|---|---|---|---|
| **Optimized Media** | Source clip | Cache (machine-local, disposable) | No | Smooth editing of hard-to-decode source |
| **Proxy Media** (18.5+) | Source clip | Proxy folder, linkable | Yes (linkable on other machines) | Portable proxy workflow; toggle on/off |
| **Render Cache** | Processed/graded frames + effects | Cache | No | Smooth playback of heavy grades/Fusion/NR |

- Set Optimized/Proxy/Cache codec in **Project Settings ▸ Master Settings ▸ Optimized Media and Render Cache** (DNxHR SQ on Windows, ProRes 422 on Mac is a good default).
- Cache/working files location: **Project Settings ▸ Working Folders** → point at a fast SSD with space.
- Generate: right-click clips ▸ Generate Optimized Media / Generate Proxy Media. Enable **Playback ▸ Use Optimized Media if Available** / Use Proxy Media.

## Delivery (Deliver page) — get the export right

- **Format/codec**: H.265 (HEVC) or H.264 for web; DNxHR/ProRes for masters/intermediates. ProRes encode on Windows requires Studio.
- **Data Levels**: keep **Video (legal)** for normal delivery. A Full↔Video mismatch causes crushed blacks / washed highlights. Match source clip levels (Clip Attributes) and output levels.
- **Color tagging / gamma**: Mac washed-out exports → enable Preferences ▸ "Use Mac display color profiles for viewers" and deliver **Rec.709-A** for web (YouTube/Vimeo). Verify by re-importing the export and reading scopes — not by trusting QuickTime Player.
- **Bit depth / chroma**: 10-bit 4:2:2+ for masters; 8-bit 4:2:0 H.264/H.265 for web.
- **Hardware encoding** (Studio): enable to massively speed up H.264/H.265 renders (NVENC / QuickSync / VideoToolbox).
- **YouTube/web preset**: H.265, high bitrate (or "Restrict to" a generous ceiling), Rec.709-A on Mac. Bars-at-head round-trip is the definitive correctness check.
- **Resolution cap**: free version exports ≤ UHD — going higher needs Studio.

## Media management & project portability

- **Folder discipline**: dedicate one project folder; keep media paths stable; never rename source files post-import (breaks relink-by-name).
- **Relink**: Media Pool ▸ right-click ▸ Relink Clips for Selected Bins → point at the folder; Resolve matches by filename.
- **Drives**: consistent drive names (Mac) / letters (Windows) across machines so paths resolve.
- **Database**: Disk database for solo; **PostgreSQL / Project Server** for multi-user collaboration. Back it up (project backups + DB dumps). Move/connect databases via Project Manager ▸ Databases while Resolve is closed to that DB.
- **Portability**: `.drp` = project (no media), `.drt` = timeline. For full hand-off use Media Management or "Copy" to collect media alongside the project.
