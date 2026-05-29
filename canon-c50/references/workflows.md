# C50 Configuration Recipes

Concrete starting points for the most common shoots. Each recipe states the goal, the picks, and the *why*. Adapt — these are senior-DP defaults, not laws.

## Recipe selector

| Project | Recipe |
|---|---|
| Indie narrative, color graded, modest budget | [Narrative](#narrative) |
| Documentary / interview / journalism | [Doc](#doc) |
| Music video / commercial / heavy stylization | [Stylized](#stylized) |
| Wedding / event | [Event](#event) |
| Social media / vertical / quick turnaround | [Social](#social) |
| VFX plate / greenscreen | [VFX](#vfx) |
| Slow motion beauty / sports | [Slowmo](#slowmo) |

---

## Narrative

**Goal:** Maximum latitude for a colorist. 24p cinema rhythm.

| Setting | Pick | Why |
|---|---|---|
| System frequency | 24.00 Hz | True cine cadence |
| Sensor mode | Full-frame 4K | Oversampled from 7K — sharpest 4K the camera makes |
| Recording format | Cinema RAW Light **ST** | 12-bit RAW, smaller than HQ, plenty for grade |
| Frame rate | 24.00p | |
| Shutter | 180° (1/48s) | Cine standard |
| ISO | 800 (base 1) | Cleanest, max DR |
| Color | C-Log 2 / Cinema Gamut | Maximum DR, ACES-ready |
| View Assist | Canon 709 LUT or custom show LUT to LCD/HDMI | So crew sees a graded preview |
| Proxy (slot 2) | XF-AVC S 1080p with LUT baked in | For editorial |
| AF | Manual / One-Shot AF + AF Lock on REC | Don't trust continuous AF for narrative |
| ND | External variable ND or matte box | C50 has no internal ND |

**Exposure target:** Skin tones around IRE 50–60 on the WFM. Highlights (Caucasian skin highlight, white shirt) under IRE 75 to leave headroom for grade. False color magenta = clipping, walk back ND or aperture.

---

## Doc

**Goal:** Lock and run. Predictable file sizes. Fast color in NLE.

| Setting | Pick | Why |
|---|---|---|
| System frequency | 23.98 Hz (or 29.97 for broadcast) | Match deliverable |
| Sensor mode | Full-frame 4K | |
| Recording format | XF-AVC 4K 10-bit 4:2:2 (Long GOP or Intra) | Broadcast accepted, sane file sizes |
| Frame rate | 23.98p / 29.97p | |
| Shutter | 1/48 or 1/60 | |
| ISO | Auto Base ISO (800 ↔ 6400) | Interview lighting varies; let the camera pick the cleaner native point |
| Color | C-Log 3 / BT.709 (or BT.2020) | Gentler grade, fast turn |
| AF | Continuous AF + face/eye detection | Subject is talking, camera is following |
| Audio | Dual XLR via handle top, channel 1 lav, channel 2 boom | Standard interview redundancy |
| Frame.io C2C | Enabled | Editor pulls dailies immediately |

---

## Stylized

**Goal:** Distinctive look, often anamorphic or open gate.

| Setting | Pick | Why |
|---|---|---|
| Sensor mode | 7K Open Gate 3:2 (or Full-Frame 4K + anamorphic) | Reframing flexibility, anamorphic ratios |
| Recording format | Cinema RAW Light HQ | Maximum data for stylized grade |
| Frame rate | 24p (or 23.98 / 25 / 29.97) | |
| Anamorphic desqueeze | Set on the camera (1.3 / 1.5 / 1.8 / 2.0) | LCD shows correct ratio without baking |
| Color | C-Log 2 + custom Look File | Apply your show LUT to monitor only |
| ND | Variable + IRND set | Stylized shoots often live wide-open |

If using anamorphic: see manual p. 128. The de-squeeze is monitor-only — your recording is still squeezed, and you de-squeeze in post.

---

## Event

**Goal:** Don't miss the moment. Long record times. Reasonable color.

| Setting | Pick | Why |
|---|---|---|
| System frequency | 29.97 Hz (US) / 25 Hz (PAL) | |
| Recording format | XF-HEVC S 4K | Smaller files, longer record, fine for events |
| Frame rate | 29.97p (or 23.98p if cinematic feel desired) | |
| Shutter | 1/60 (US) / 1/50 (PAL) | Avoid flicker under venue lighting |
| ISO | Auto Base ISO | Lighting changes constantly |
| Color | Wide DR or BT.709 | Out-of-camera usable look |
| AF | Continuous AF + face detection | Trust it — you're solo operator |
| Pre-record | Enable, 3–5 sec | Catches the toast you didn't see starting (p. 125) |
| Slot 2 SD | Proxy or relay record | Backup for long ceremonies |

---

## Social

**Goal:** Vertical or square delivery, fast cut, often LUT-baked.

| Setting | Pick | Why |
|---|---|---|
| Sensor mode | 7K Open Gate 3:2 | Crop to vertical 9:16 or square 1:1 in post without losing resolution |
| Recording format | XF-HEVC S or XF-AVC S | Manageable file sizes |
| Frame rate | 23.98p or 29.97p | |
| Color | Wide DR or BT.709 with custom Look File | Skip the grade |
| Onscreen markers | Enable 9:16 safe area | Frame for vertical (manual p. 100) |
| Frame.io C2C | Enabled | Edit on phone if needed |

---

## VFX

**Goal:** Clean plate, max latitude, no in-camera processing that fights the comp.

| Setting | Pick | Why |
|---|---|---|
| Sensor mode | Full-frame 4K (or 7K if budget allows) | Native, oversampled |
| Recording format | Cinema RAW Light **HQ** | 12-bit, no compression artifacts in green channel |
| Frame rate | 23.98p | |
| Shutter | 180° (1/48s) | Match plate motion blur |
| ISO | 800 (base 1) | Cleanest signal, cleanest key |
| Color | C-Log 2 / Cinema Gamut | Scene-referred for ACES, max DR for matchmoving |
| Lens correction | OFF (in-camera lens correction) | VFX wants the raw distortion + vignette, applied in comp |
| Detail / NR | OFF in Custom Picture | No baked-in sharpening or noise reduction |
| ACES | Enable IDT workflow (manual p. 19) | Tracking/CG/comp pipeline |

---

## Slowmo

**Goal:** 4K120 or 2K180 conformed to 23.98 base.

| Setting | Pick | Why |
|---|---|---|
| Recording format | XF-AVC 4K120 — **NOT Cinema RAW Light** for ≥100 fps | RAW on the C50 is 7K only and capped at 60p. High-frame-rate work has to be non-RAW. Verify per-mode caps at manual p. 214. |
| Frame rate | 4K up to 119.88p; 2K up to 179.82p | 4K120 needs XF-AVC; 2K180 uses Slow & Fast Motion mode |
| Shutter | 1/240 minimum for 120p (matches 180° rule at higher frame rate) | Cinematic blur at fast capture |
| ISO | 6400 (base 2) if you need shadow detail in low light; otherwise 800 | High-frame-rate needs lots of light — plan for it |
| Special record mode | Slow & Fast Motion (manual p. 124) | Sets the camera up for variable frame rate properly |
| Firmware note | On firmware <1.0.3.1, 125P / 144P / 150P RAW recording can corrupt clips — fixed in 1.0.3.1. Update first or stay clear of those rates. | |

**Light math:** 120p at 180° is 1/240s shutter. That's two stops less light than 24p at 180°. You need ~4× the light, or to open up, or to raise ISO.

---

## Assignable button preset (my default)

The C50 has 14 assignables (manual p. 131). I set these up the same way on every C50 I work with:

| Button | Function | Reason |
|---|---|---|
| Camera 1 (AF-ON) | One-Shot AF | Pull focus on demand without giving up manual |
| Camera 2 (MAGN.) | Magnification | Critical-focus check |
| Camera 3 (MENU) | MENU | Default |
| Camera 4 (DISP) | Cycle display info | Default-ish |
| Camera 5 (PUSH AUTO IRIS) | Push Auto Iris | Momentary auto exposure for quickly nailing aperture |
| Camera 6 | **False Color** | Single most useful exposure tool. Press, expose, press again |
| Camera 7 (REC top) | REC | |
| Camera 8 (LOCK) | Key Lock | Prevent accidents on long takes |
| Camera 9 (PEAKING) | Peaking | |
| Camera 10 (WFM) | Waveform | |
| Camera 11 (FUNC) | FUNC / Direct Setting Mode | Default; this is your in-camera quick menu |
| Camera 12 (AF LOCK) | AF Lock | Lock focus during a take |
| Camera 13 (front REC) | REC | |
| Camera 14 (LCD side) | **Anamorphic desqueeze toggle** OR **LUT on/off** | Whichever you use more on this job |

The trio I rely on every shot: **False Color, Waveform, Magnification.** Map them somewhere your finger finds without looking.
