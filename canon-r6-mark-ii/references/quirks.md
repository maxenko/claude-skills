# Canon EOS R6 Mark II — Confirmed Quirks & Non-Intuitive Behaviors

The things that aren't on the spec sheet but bite real users. Each is corroborated by a reputable lab/reviewer or Canon's own docs. Confidence is labeled. When you cite one, match the confidence — don't upgrade "widely reported" to "confirmed."

---

## 1. 4K60 is full-width but NOT oversampled — it's softer than 4K30
**Confirmed.** This is the single most misunderstood thing about the camera, and two reputable sources can *both* be quoted in ways that sound contradictory. The reconciliation:
- The R6 II reads the **full sensor width with no crop at every 4K rate, including 60p** — a genuine improvement over the original R6 (which applied a ~1.07× crop at 4K60).
- **But "no crop" ≠ "oversampled."** 4K **≤30p is downsampled from the full ~6K readout** → sharpest, lowest aliasing/moiré, cleanest. 4K **50/60p switches to a line-skipped/subsampled readout** → visibly softer with more aliasing, *despite* being full-width.
- **Why:** oversampling the full 6K at 60p would generate too much heat/processing load on the non-stacked sensor + DIGIC X, so Canon line-skips for 60p. (DPReview notes the rolling-shutter figure barely changes between the modes, implying it's a deliberate processing choice, not a readout-speed limit.)
- **Workaround / how to advise:** for maximum detail (interviews, landscapes, product), shoot **4K 24/25/30**. Reserve **4K60** for motion you'll slow down and where slightly softer/aliased footage is acceptable. Don't tell a user "4K60 is uncropped so it's the same quality as 4K30" — it isn't.
- Source: DPReview in-depth review; corroborated across DPReview forums and Filmkit recording-mode breakdowns.

## 2. There is NO "Auto power off temperature (Standard/High)" menu item
**Confirmed (manual-grounded).** Online guides written for the R5/R5C routinely tell R6 II users to "set Auto power off temp to High." **That setting does not exist on the R6 II.** Heat is managed by **[Overheat control]** (Standby uses a lower-resolution feed to run cooler) and **[Standby: Low res.]** (manual p. 1052, 1041). If a user can't find Standard/High, that's why — point them at Overheat control instead. (Do not invent the Standard/High item just because the C50/R5 has it.)

## 3. Electronic shutter drops RAW from 14-bit to 12-bit
**Confirmed.** Any time you're in full electronic shutter — the **40 fps burst, RAW Burst mode, focus bracketing, silent shooting** — RAW readout is **12-bit, not 14-bit**. Penalty shows up in smooth gradients (skies) under heavy contrast pushes and in lifted-shadow noise; "seldom noticed in well-exposed images" (DPReview). **Your single best-quality file is mechanical-shutter, ISO 100, 14-bit.** Use 12-bit modes when speed/silence beats marginal latitude. Source: DPReview, The-Digital-Picture.

## 4. Electronic-shutter rolling shutter & banding on fast/flickering subjects
**Confirmed.** Full-readout scan is roughly **14.5–18 ms** (The-Digital-Picture ~14.5 ms; DPReview ~18 ms — source variance, quote a range). Because the sensor is **not stacked**, e-shutter skews fast horizontal motion (race cars, golf swings, prop planes, bird wings) and the 40 fps burst inherits the same readout — Canon itself advises against the 40 fps mode for close-up motorsport/golf analysis. E-shutter also **bands under LED/fluorescent light and from other people's flashes**. For geometry-critical fast action, fall back to **12 fps mechanical**. Note: **CineD never ran their full lab suite on a production R6 II** (pre-production unit), so there is no CineD per-mode rolling-shutter/DR/latitude table for this body — don't attribute the original R6's CineD numbers (e.g. 30.6 ms) to the Mark II.

## 5. Anti-Flicker vs High-Frequency Anti-Flicker solve different problems
**Confirmed.** Users conflate these and pick the wrong one.
- **Anti-flicker shooting** fixes *frame-to-frame brightness flicker* from 100/120 Hz mains-powered fluorescent/LED — the camera times the exposure to the brightness peak. Works mech/EFCS. Can't detect frequencies other than 100/120 Hz. (manual p. 206)
- **High-Frequency Anti-Flicker** fixes *intra-frame banding* from high-frequency / PWM-dimmed LED panels, scoreboards, signage. Detects **50.0–2011.2 Hz** and lets you fine-tune shutter speed in tiny fractional steps so each frame spans whole flicker cycles. **Requires Tv or M mode** (it sets a precise shutter speed) and works with electronic shutter. If it's greyed out, the user is in P/Av. (manual p. 208–211, Canon KB ART182172)

## 6. Canon Log 3 horizontal banding on dark, flat subjects
**Confirmed (manual warning, p. 405).** With Canon Log, dark flat areas (clear skies, gradients) can show faint horizontal banding, and it "may even occur at relatively low ISO speeds around ISO 800." This is documented Canon behavior, not a defect. Advise: avoid under-exposing log, add a touch of exposure to dark flat scenes, and don't crush the curve in grade. Also: **C-Log 2 is not available** on this body — only Canon Log 3.

## 7. Not fully ISO-invariant at base — shoot ISO 400, not "ISO 100 and push"
**Confirmed (DPReview, PhotonsToPhotos).** Base **ISO 100** carries more deep-shadow read noise than slightly higher ISOs; the gap largely closes by **~ISO 400** (dual-conversion-gain behavior — distinct from dual *base* ISO, which this camera does not have). For a dark scene you intend to lift hard, exposing at **ISO 400+ yields cleaner shadows** than ISO 100 lifted in post. Measured base-ISO photographic dynamic range ≈ **11.5 stops** (PhotonsToPhotos PDR) — solid, not class-leading.

## 8. USB-C powers the camera only with a battery installed
**Confirmed (Canon manual / ART177951).** The R6 II **cannot run battery-less off USB** like a webcam rig. Rules: switch **OFF** → adapter *charges*; switch **ON** → battery only charges during auto-power-off; a **depleted battery makes the adapter revert to charging and stop powering** the camera, and the level can still slowly drain on a weak source during heavy recording. Requires a true **USB-PD** source (PD-E1 or quality PD bank) — a plain 5 V phone charger won't power it. For guaranteed continuous power many users run a USB-C-to-dummy-battery (LP-E6 coupler).

## 9. False Color exists for video — but with hard limitations
**Confirmed.** The R6 II has a 6-color brightness-based **false color** for video exposure. Gotchas: it **cannot be displayed at the same time as View Assist, Focus Peaking, or Zebra** (one assist at a time), and it **cannot be assigned to a custom button** — you toggle it through the menu. So you can't flick between false color and a peaking view the way you can on a cinema body. Source: Canon KB ART174760, Canon manual false-color page.

## 10. RAW Burst pre-shooting captures ~0.5 s *before* you press
**Confirmed.** RAW Burst mode shoots **30 fps, 12-bit**, writing all frames into **one .CR3 "roll"** (not separate files). With **Pre-shooting enabled**, holding a half-press fills a rolling buffer and a full press retains roughly the **0.5 s (~15–20 frames) before** you reacted — superb miss-insurance for wildlife/sports. Cost: extract frames in-camera or in **DPP** (the reliable route; Lightroom historically couldn't open the roll directly), and it's 12-bit. Don't use it for routine bursts where the normal drive ingests more cleanly. (manual p. 296–298, Canon KB ART182305)

## 11. Focus bracketing is electronic-shutter, no flash, no MF
**Confirmed.** In-camera focus bracketing + depth compositing (2–999 frames, exposure smoothing, saves merged + source frames) uses the **electronic shutter**, so you **cannot fire a flash per frame** — a real limit for macro shooters who light each plane (the Panasonic S5 II is the usual counter-example). It's **unavailable in MF**, max shutter 1/8000, and **fails on flat/uniform or repetitive-pattern subjects**. Use a wider aperture (f/2.8–f/4) so frames are distinct rather than tiny apertures. (manual p. 299–303, Canon KB ART182171)

## 12. IBIS is tied to the lens IS switch (RF IS lenses)
**Confirmed.** With a stabilized RF lens, IBIS and lens IS are linked and **controlled by the lens's physical IS switch** — you cannot independently toggle IBIS from the menu (the menu IS-mode item effectively governs non-IS lenses). Flip the lens switch off and IBIS goes off too. Confuses users who go hunting in the menu for an IBIS on/off. The headline **8 stops is lens-dependent** (measured with RF 24-105 F4L at 105 mm); most lenses deliver fewer. (Canon KB ART182146)

## 13. Dual SD UHS-II, both same type — backup writing slows buffer clear
**Confirmed.** No CFexpress; both slots are SD UHS-II (≈312 MB/s ceiling). Internal 4K60 tops out ~340 Mbps, comfortably within UHS-II, so video isn't bottlenecked. But writing RAW to **both** cards simultaneously (backup) **reduces effective buffer-clear speed** vs a single fast card, and slow cards dramatically lengthen recovery (one lab saw 11.5 s vs 32 s to clear the same buffer on fast vs slow UHS-II). Use genuine **V90 UHS-II** cards, especially for 40 fps RAW and 10-bit log (which needs V60+). Source: RFShooters lab buffer tests.

## 14. Ergonomic surprises for DSLR / R5 migrants
**Confirmed / widely reported.**
- **Dedicated two-position still/movie switch** (a physical collar, not a mode-dial position) swaps the *entire* menu/UI set. Users panic when settings "vanish" — they're in the other mode. It is a **two-position** (still ⟷ movie) switch, **not** a three-position still/movie/auto switch.
- **No top status LCD** (unlike R5/R3) — read settings off the rear screen/EVF.
- **Mode dial has a center lock button** — press to rotate; prevents accidental changes but slows fast mode swaps.
- **micro-HDMI Type D** is fragile under cable strain — a real durability complaint for rigs feeding an external recorder; use a cable clamp/cage HDMI lock.

## 15. Movie pre-recording is NOT a feature (don't promise it)
**Confirmed absent.** RAW Burst **stills** pre-shooting (item 10) is real and often gets miscited as "the R6 II can pre-record video." The R6 II has **no movie/video pre-recording buffer** — that arrived on later bodies (R5 II / R6 III class). If a user wants pre-roll video, it isn't on this camera.

## 16. Things people wrongly attribute to firmware
**Widely reported, verify per release.** The frequently cited "1.5.2 AF improvement" belongs to the **original R5/R6**, not the R6 II. On the R6 II, **1.5.0** was mainly FTPS/SDK/security; later updates (through **1.7.0**, the bundled manual's version) added Wi-Fi band selection and FTP/SFTP fixes. When a user reports a bug, ask their firmware version ([Set-up] → [Firmware]) and check Canon's per-version R6 II release notes rather than repeating cross-model claims.
