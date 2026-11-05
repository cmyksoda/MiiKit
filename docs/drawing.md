# Drawing Miis

[miidraw.h](../miidraw.h) draws a Mii with GX: its head as the Wii draws it, from the console's own face models, and a body to go with it. The README has a whole example, and `miidraw.h` documents every call. This page is the overview.

## Face models and bodies

- `mii_res_load_nand(&res)` reads the Wii Menu's `RFL_Res.dat`, and `mii_bodies_load_nand(&male, &female)` the Mii Channel's two bodies. Each takes about 0.1 s, so call them outside the frame loop. They need AHBPROT (`<ahb_access/>` in `meta.xml`, or Wiiload) and no IOS reload before the call, and return `MII_E_DENIED` without it.
- From the SD card instead: `mii_res_load(&res, NULL)` reads `sd:/mii/RFL_Res.dat`, and `mii_body_load(&body, MII_BODY_MALE_PATH)` (or `MII_BODY_FEMALE_PATH`) a body from `sd:/mii/`. hbc-reborn's Save Mii faces copies all three there.
- A model keeps what it needs from `res`, so free `res` once the models are built. A body has to outlive the models it's under.

## Building a Mii

- `mii_model_make(&res, &m, 128, &model)` builds everything once: the head placed as RFL places it, the face in all seven expressions (`MII_EXPR_*`), and a body in the Mii's colors, scaled by its height and build. That's about 280 KB with 128 px faces, mostly the seven faces.
- `mii_model_build` takes `mii_model_opts`: the face size (32 to 256 px) and which expressions to build. With 128 px faces and only the normal one, a Mii takes about 67 KB.
- `mii_model_set_body(&model, &body)` puts the Wii's own body under it: Nintendo's skeleton and the Mii Channel's shirt and trousers, skinned on the GPU, with its animations and its shading. Without one, the model keeps a plainer body of MiiKit's own.

## Ways to draw it

- **In place, each frame:** `mii_view()` gives the projection and camera that frame the Mii in a viewport of your aspect, then `mii_draw()` draws it. Give the Mii a wide depth range: on a real Wii, its face's layers break up in a thin one (Dolphin hides this).
- **Into your own 3D scene:** `mii_draw()` with your projection and camera, and a `place` matrix for where the Mii stands.
- **Into a texture:** `mii_capture()` draws the whole Mii (or the head, with `pose.head_only`) with alpha, at twice the size and filtered down, for stills. `mii_icon()` draws a head icon framed as the Wii Menu frames it.

## Posing it

- `mii_pose` holds the turn, the head's yaw, pitch and roll, the arms, the legs, a bob, breathing and the expression. All zero stands still.
- With the Wii's body, set `pose.anim` to `1 + mii_body_anim(&body, "bodyWait")` (or `bodyGreeting`, `bodyWalk`, `bodyRun`, `bodyTalk`, ...) and `pose.frame` to 60 a second. `mii_anim_frame()` gives the frame to use, wrapped for a looping animation and held at the end for one that plays once.
- To change animation without a jump, keep the one before in `pose.from_anim` and `pose.from_frame`, with `pose.from_weight = mii_fade_weight(seconds_since, 0.4f)`.
- `mii_anim_expression()` picks a face to go with an animation: eyes shut for a nap, a smile for a wave. The files don't say which face goes with which animation, so these are MiiKit's own choices.
- `pose.shadow = 1` draws the body's shadow under its feet, and `pose.shadow_drop` keeps it on the ground under a Mii that's carried.

## Effects

`miifx.c` plays the Mii Channel's particle effects: a sneeze, speech balloons, dust from running and landing, the Zzz of a nap, and sparkles. `mii_fx_load(&fx, NULL, NULL)` reads them from `sd:/mii/effect_nigaoe.breff` and `.breft`. Then:

1. `mii_fx_init()` sets up a state that plays up to 16 emitters and 128 particles, dropping the oldest when it's full.
2. `mii_fx_place_in()` gives a place on the Mii's head, or on the ground under it, and `mii_fx_play()` starts an emitter there.
3. Each frame, after drawing the Mii, `mii_fx_step()` moves the effects on and `mii_fx_draw()` draws them.

`mii_icon_fx()` draws a head icon with effects over it.

## Many Miis

The Wii's CPU and memory limit a crowd before its GPU does. These help:

- **One `mii_cache` for the crowd** (`mii_model_opts.cache`): Miis share every part that comes out the same. With 32 px faces and only the normal one, a Mii then takes 12 to 20 KB.
- **Coarser levels for Miis far off:** `mii_body_make_lods()` once per body, `mii_model_opts.lods = MII_HEAD_LODS` for each Mii, then `pose.lod = mii_body_lod_for(&body, mii_pixels_tall(...))` each frame.
- **Faces drawn by the GPU** (`mii_model_opts.gpu_faces = 1`): about 0.08 ms a face, against 1 to 6 ms on the CPU. Call `mii_model_make_faces()` between frames, on the thread that draws.
- **Posing less often:** Miis at the same frame of the same animation pose once, and `pose.reuse = 1` draws a far-off Mii with the bones its last draw posed.

## GX rules

- **Keep your GX FIFO in MEM1,** as a static array, not from `memalign`. Once MEM1's part of the heap runs out, the heap gives MEM2, and a GPU fed its commands from MEM2 stalled at random on a real Wii (never in Dolphin).
- **Call `mii_capture`, `mii_icon` and `mii_model_make_faces` between frames,** after your last copy to the screen. They draw in a corner of the EFB, so set your GX state again after them (pixel format, viewport, scissor, projection, copy filter, vertex format, color channel). `mii_draw` changes the vertex format, lighting, TEV, Z, culling and blending.
- **Leave Z writes on at your copy.** A copy that clears the screen clears Z only while Z writes are on. MiiKit's calls leave them on.
- **Read a captured texture after `GX_DrawDone()`.** In Dolphin, the CPU sees it as zeros unless Graphics > Hacks > "Store EFB Copies to Texture Only" is off; drawing it works either way.
- **Use `c_guMtxConcat` for your own matrices,** or set `_SDA_BASE_ = .;` at the start of `.sdata` in your linker script. With devkitPPC's linker scripts, libogc's paired-single `guMtxConcat` loads a constant from the wrong address, and every product it makes is off. MiiKit uses the C one itself.

The head's numbers come from RFL as the CC0 decompilations show it, and the code is MiiKit's own. hbc-reborn's [mii-rendering.md](https://github.com/Monsterray/hbc-reborn/blob/master/docs/research/mii-rendering.md) lists the sources.

## Not done yet

- **Height and build as the Wii scales them.** The bones are scaled with the formula of the Wii U's Mii library (FFL), because the Wii's own isn't in the data.
- **Wii U and 3DS Miis.** `mii_import` takes only the Wii's 74- and 76-byte forms, not the 96-byte form the Wii U and 3DS use.
- **Wii U-style drawing,** with FFL's rim light, the specular light on hair, and more texture formats.
- **The Mii Channel's own timing** for faces and effects during animations. MiiKit's are chosen by eye.
