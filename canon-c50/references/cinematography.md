# Cinematography Fundamentals

The general-knowledge reference for when the user asks beyond the C50 — exposure, framing, lighting, lensing, movement. Camera-agnostic principles, but with C50 callouts where relevant.

## Contents

- [The cinematographer's exposure triangle (different from photography)](#the-cinematographers-exposure-triangle-different-from-photography)
- [Shutter angle cheat sheet](#shutter-angle-cheat-sheet)
- [Exposing log footage](#exposing-log-footage)
- [Lens choice](#lens-choice)
- [Composition fundamentals](#composition-fundamentals)
- [Lighting ratios](#lighting-ratios)
- [Color temperature](#color-temperature)
- [Camera movement](#camera-movement)
- [Frame rates and their feels](#frame-rates-and-their-feels)
- [Sound on set (because DPs often shoot solo)](#sound-on-set-because-dps-often-shoot-solo)
- [Color grading prep (camera-side)](#color-grading-prep-camera-side)

## The cinematographer's exposure triangle (different from photography)

Photography treats shutter, aperture, ISO as three equal dials. Video doesn't — because **shutter is locked by motion aesthetics**, not by exposure.

The working order:

1. **Frame rate** — set by deliverable (24p cine, 23.98p NTSC cine, 25p PAL, 29.97p broadcast, 60p sports/slow-mo source).
2. **Shutter angle** — set by motion feel. 180° default (= 1/[2× frame rate]). Tighter for crisp action, wider for dreamlike blur. **Almost never changed mid-scene.**
3. **ISO** — locked at base ISO unless the lighting situation demands the second base (C50: 800 or 6400).
4. **Aperture** — shapes depth of field. This is your *creative* exposure dial.
5. **ND** — modulates light without changing any of the above. **This is how cinematographers actually expose outdoors.**

So when light changes during a shoot, the cinematographer's instinct is **swap ND first**, then maybe step ISO base (if dramatic), then maybe stop down. They almost never touch shutter.

## Shutter angle cheat sheet

| Frame rate | 180° shutter | Speed |
|---|---|---|
| 23.98p | 1/48 (1/50 if no 1/48 option) | Cine standard |
| 24.00p | 1/48 | Cine standard |
| 25p (PAL) | 1/50 | |
| 29.97p | 1/60 | |
| 50p | 1/100 | |
| 59.94p / 60p | 1/120 | |
| 100p | 1/200 | |
| 119.88p / 120p | 1/240 | |

Going below 180° (e.g., 90° = 1/96 at 24p) sharpens action — the *Saving Private Ryan* beach landing was 45° to make explosions jagged and visceral. Going above 180° (e.g., 270° or 360° = 1/24 at 24p) softens motion, dreamy — used for memory sequences, drug states.

## Exposing log footage

Log gamma curves (C-Log 2, C-Log 3, S-Log, LogC) look gray and flat in the viewfinder by design — they preserve highlight and shadow detail by compressing them logarithmically. Three working methods:

1. **Waveform method.** Skin tones (Caucasian average) sit around IRE 50–60 in C-Log 2, ~60 in C-Log 3. Black point around IRE 5–10. Highlights you want to retain under IRE 85. Clipping happens around IRE 95+.
2. **False color method.** Faster but less precise. Configure false color so skin is green/turquoise, highlights to retain are yellow/orange, clipping is red. On the C50, false color is a one-button toggle (assign it).
3. **LUT-to-monitor method.** Apply a Rec.709 (or show LUT) to the LCD/HDMI without baking into the recording. You see a rough graded image; you expose by eye to that.

**Expose to the right (ETTR) is generally good for log** because noise lives in the shadows. Push exposure as bright as you can without clipping highlights, then bring it back down in post.

## Lens choice

| Focal length (FF equivalent) | Feel | Use |
|---|---|---|
| 14–24mm | Very wide, dramatic | Establishing, environments, dynamic handheld |
| 24–35mm | Wide | Naturalistic interiors, walk-and-talks |
| 35–50mm | Normal | Documentary, naturalism (35 = European naturalism; 50 = Hollywood standard) |
| 50–85mm | Short tele | Classic portraiture; interviews |
| 85–135mm | Tele | Beauty, romantic close-ups |
| 135mm+ | Long tele | Compression, isolation, voyeuristic |

**Two senior-DP rules:**
- Pick a focal length, *then* find your distance to subject. Don't zoom to crop — move.
- Spielberg's "if the lens choice tells the story, you've used the right lens."

## Composition fundamentals

- **Rule of thirds** — eyes on the upper third, horizon on top or bottom third (rarely middle).
- **Headroom** — small gap above the head in a CU/MCU. Too much = subject looks lost; too little = claustrophobic (sometimes desired).
- **Lookroom / noseroom** — when a subject looks screen-left, leave space on the left for their gaze. Reverse for screen-right.
- **Eyeline match** — in shot/reverse-shot, the two subjects' eyelines must converge consistently. Crossing the 180° line breaks the spatial relationship.
- **Negative space** — empty area is a tool, not a failure. Use it to isolate, to imply something off-frame, to convey loneliness.

## Lighting ratios

Ratio = key light : fill light (measured at the subject).

| Ratio | Look | Use |
|---|---|---|
| 1:1 | Flat, no modeling | High-key comedy, news, beauty |
| 2:1 | Soft modeling, one stop difference | Documentary interview, naturalistic drama |
| 4:1 | Two stops, classic Hollywood | Drama, character work |
| 8:1+ | Three+ stops, harsh shadows | Noir, thriller, horror |

**Motivated light** — every light source on screen should have a fictional reason for being there (window, lamp, fire, moonlight). Even fully invented light should *appear* motivated.

## Color temperature

| Source | Kelvin |
|---|---|
| Candle | ~1800 K |
| Tungsten bulb | 3200 K |
| Sunrise/sunset | ~3500 K |
| Fluorescent (cool white) | ~4000 K |
| Daylight (noon) | 5600 K |
| Overcast | ~6500 K |
| Shade | ~7500 K |
| Blue hour | ~10000 K |

Match the camera white balance to your dominant source, then *let other sources* read warm or cool relative to it. A daylight WB indoors makes tungsten lamps glow warm — often desirable.

## Camera movement

- **Locked off** — formality, observation, distance.
- **Handheld** — immediacy, subjectivity, naturalism (or chaos when overused).
- **Gimbal / Steadicam** — flow, dreamlike, "invisible" movement.
- **Dolly** — physical weight, deliberate, often parallels emotional beats.
- **Crane / jib** — reveal, scope, transitions.
- **Whip pan / snap zoom** — kinetic energy, comedy or action accent.

The rule: **movement should be motivated.** Either the subject moves and the camera follows (anchored), or the camera reveals/conceals information (narrative), or it expresses character interior state. Movement for its own sake fatigues the audience.

## Frame rates and their feels

| Rate | Feel |
|---|---|
| 24 / 23.98 | Cinema. The standard. |
| 25 | European broadcast cinema; matches PAL TVs |
| 29.97 / 30 | Broadcast / "soap opera" feel if no motion treatment |
| 48 / 50 / 60 | "Hyperreal" / sports / HFR cinema |
| 60+ | Slow-motion source (conform to 24) |
| 120 / 180+ | Strong slow-mo or beauty work |

The C50 supports up to 4K120 in XF-AVC and 2K180 in slow & fast motion mode (manual p. 214).

## Sound on set (because DPs often shoot solo)

- Internal mic = scratch only.
- Two-channel pro: lav on subject (channel 1), boom (channel 2). C50 dual XLR via top handle.
- Always record room tone (60 sec per location) for the editor.
- 48 kHz / 24-bit minimum for any pro deliverable.

## Color grading prep (camera-side)

What you do on-set to make your colorist's life easier:
- **Slate a gray card** + white card at each new lighting setup, 5 sec each.
- **Lock white balance** at the start of a scene; don't AWB mid-scene.
- **Shoot a few seconds of color chart** (Macbeth/ColorChecker) when lighting is critical.
- **Note your custom picture / LUT** in the slate or paperwork.
- **Don't bake** sharpening, NR, or grade into log recordings.
