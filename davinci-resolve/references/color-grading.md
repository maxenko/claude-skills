# Color Grading & Color Management

Deep reference for the Color page. The two highest-leverage decisions (color-management mode and node order) are in SKILL.md; this expands the recipes.

## Color management modes in depth

### DaVinci YRGB (unmanaged, display-referred)
No automatic transforms. You manage log→Rec.709 yourself (CST nodes or LUTs). Good for a single Rec.709 camera, fast jobs, or colorists who want full manual control. The "classic" Resolve feel.

### DaVinci YRGB Color Managed (RCM) — recommended default
Project Settings ▸ Color Management ▸ Color Science = **DaVinci YRGB Color Managed**.
- **Color Processing Mode**: pick **DaVinci Wide Gamut** (DWG/DI) for the modern wide-gamut pipeline; "HDR DaVinci Wide Gamut Intermediate" is a safe preset.
- **Input Color Space**: enable Automatic Color Management to tag by camera metadata, or set per-clip (right-click clip ▸ Input Color Space) — critical for mixed-camera projects. A mistagged input is the usual cause of "this camera looks wrong."
- **Output Color Space**: your delivery, e.g. **Rec.709 Gamma 2.4** (broadcast/standard), Rec.709-A (web on Mac), Rec.2100 ST2084 (HDR10).
- Grade between input and output; RCM handles the heavy transforms so wheels behave predictably across cameras.

### ACES
Project Settings ▸ Color Science = **ACEScct** (recommended grading variant) or ACEScc.
- Set **ACES Version**, **Input Transform (IDT)** per clip, **Output Transform (ODT)** for delivery.
- Choose ACES only when interchange/VFX/cinema delivery mandates it. The RRT imposes a specific look and the pipeline is more rigid than RCM. Don't push it on a simple web edit.

**Decision recap:** single Rec.709 cam + speed → YRGB. Mixed/log/HDR/consistency → RCM+DWG. Mandated interchange/VFX → ACES.

## Node tree recipes

Node types:
- **Serial** (Alt/Opt-S) — sequential, output feeds next input. The default; order matters.
- **Parallel** (Alt/Opt-P) — branches share the same input, combined equally in a Parallel Mixer. For independent secondaries that shouldn't stack destructively.
- **Layer** (Alt/Opt-L) — like parallel but with top-to-bottom **priority** (compositing order). Use when one correction must sit "over" another.
- **Outside** node — inverts the previous node's qualifier/window (grade everything *except* the selection).
- **Timeline / post-clip node** — one adjustment across the entire timeline (e.g. a final trim or LUT).

A clean professional tree:
```
[1 Input/CST] -> [2 Balance/Normalize] -> [3 Primary contrast] -> [4 Parallel: skin / sky / etc.] -> [5 Look] -> [6 Output/Limiter]
```

Order-of-operations rule: **correct before you create.** Neutralize and balance (exposure, white balance, contrast) before adding a stylized look. A look built on an unbalanced base falls apart across shots and won't match scene-to-scene.

## Secondaries: qualifiers, windows, tracking

- **Qualifier (HSL)** — key a color (skin, sky, foliage). Tighten with the qualifier's blur/clean controls; view the highlight/matte to refine. Add an **Outside** node to grade the rest.
- **Power Windows** — shape-based isolation (vignette, relighting a face). Combine with qualifiers (window AND qualifier) to constrain a key to a region.
- **Tracking** — track a window to follow a subject (Color ▸ Tracker, forward/backward). For tricky motion use the point/Magic Mask tracker (Magic Mask is Studio-only and heavy — cache it).
- **Magic Mask / Depth Map** (Studio) — AI subject/depth isolation; powerful but GPU-expensive, cache before scrubbing.

## Reading scopes (trust these, not the monitor)

- **Waveform** — luma/levels. Keep within legal range for broadcast (0–100 IRE / video levels). Spread for contrast; watch for clipping at top/bottom.
- **RGB Parade** — white balance and casts. Align the bottoms (blacks) and tops (whites) of R/G/B to neutralize; offsets reveal a cast.
- **Vectorscope** — hue/saturation. Skin tones should sit along the skin-tone line (upper-left I-bar). Over-saturation pushes toward the edge.
- **Histogram** — distribution; quick clip check.

Match shots by eye *and* scope: line up the parade and waveform between shots in a scene, then refine by eye. A calibrated reference monitor is ideal; absent that, scopes are the objective truth.

## Blending a grade across a cut (transition the grade, not the image)

Common ask: two adjacent clips have slightly different grades and the user wants the grade to *transition smoothly* across the cut — without an Edit-page cross-dissolve (which blends the two **images**). The goal is to blend the **grade values only**.

There is **no native "interpolate grade across a cut" button.** Two reliable techniques:

### A) Dynamic keyframes on the Color page (preferred when node trees match)
Cleanest when both clips share an **identical node tree** (parameters map 1:1). Example goal: carry Clip A's grade into the head of Clip B, then ramp to B's own grade over ~1.5 s.

1. **Grab a still of each grade first** (Gallery ▸ Grab Still on Clip A and Clip B) — safety net *and* the source you'll apply later.
2. Work in **Clip** mode, not Timeline mode (the Clips/Timeline toggle). Per-clip grade-copy and keyframes have nothing to land on in Timeline mode.
3. Put the incoming look on Clip B: select Clip B's thumbnail (orange outline = active clip), then **middle-click Clip A's thumbnail** to copy A's whole grade onto B. (More reliable than applying a still; `Apply Grade` via right-click also works.)
4. Open the **Keyframes panel**. On the **Master** track (ramps the entire grade at once), place a **Dynamic** keyframe at the **head** of Clip B — right-click ▸ *Add Dynamic Keyframe*. Static keyframes hold-then-snap (the hard jump you're removing); **dynamic** keyframes interpolate. The add/prev/next keyframe buttons (◀ ◆ ▶) are **hover-revealed** to the right of the track name.
5. Move the playhead to ~1.5 s in and add a **second Dynamic keyframe**.
6. **The key trick:** with the playhead parked *exactly on the second keyframe*, **Apply Grade from Still B**. Resolve writes B's values **into that keyframe** instead of wiping the animation — so a full, complex grade (curves and all) lands on the endpoint with no manual re-dialing. Result: keyframe 1 = A's look, keyframe 2 = B's look, dynamic ramp between them.

> Note: `Apply Grade` normally replaces the whole grade. But when a keyframe exists at the playhead, the applied values record *into* that keyframe. This is the clean way to set a complex grade at a keyframe — manually nudging curves/wheels back to a target is impractical and unnecessary.

Verify: scrub Clip B — head matches A, smooth drift, full B by 1.5 s, and the value **holds flat after the last keyframe** (drift/pop after = a stray extra keyframe). Park one frame each side of the cut: they should look near-identical.

### B) Crossfade two grade-versions of identical footage (when you'd rather not keyframe)
Works for any grades (node trees need not match) and sets exact full-grade endpoints automatically. Because both layers are the *same frames*, only the grade blends — no ghosting.
1. Duplicate Clip B's head onto the track above (**Alt-drag** up to V2), aligned to the cut, trimmed to the transition length.
2. Grade the V2 copy with the **incoming** look (Apply Grade from Still A); leave V1 Clip B on its own grade.
3. **Fade the V2 copy out** over its length (drag the clip's fade handle). The incoming look dissolves to reveal the destination look underneath. Curve the fade handle to ease it.

The same idea inside the Color page is a **Layer Mixer** node with each look on its own (compound) layer, keyframing the top layer's Key ▸ Output Gain — but the V2-duplicate route is far simpler for most users.

## HDR notes (Studio)
- Grade in a wide-gamut managed/ACES pipeline with an HDR output (Rec.2100 PQ/ST2084 or HLG).
- Use the HDR palette / zones for tone-mapped control; deliver Dolby Vision (Studio) when the contract requires it, with the correct mastering display metadata.
- Always confirm a Rec.709 trim/derivative looks acceptable if an SDR deliverable is also required.
