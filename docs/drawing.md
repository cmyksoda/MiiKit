# Drawing Miis

[miidraw.h](../miidraw.h) and [miidraw.c](../miidraw.c) draw a Mii with GX: its
head as the Wii draws it, from the console's own face models, and a body to
go with it. HBC's avatar ([channel/channelapp/source/avatar.c](https://github.com/Monsterray/hbc-reborn/blob/master/channel/channelapp/source/avatar.c))
is the worked example.

```c
#include "mii.h"
#include "miidraw.h"

static mii_res res;
static mii_model model;
static mii_body body;
mii m;
int ok;

/* Once, outside the frame loop (about 0.1 s; any thread). */
ok = mii_res_load(&res, NULL) == MII_OK;           /* sd:/mii/RFL_Res.dat */
ok = ok && mii_get(0, &m) == MII_OK;               /* or any Mii you have */
ok = ok && mii_model_make(&res, &m, 128, &model) == MII_OK;
mii_res_free(&res);                                /* the model keeps what it needs */
if (ok && mii_body_load(&body, m.sex == MII_FEMALE ? MII_BODY_FEMALE_PATH
                                                   : MII_BODY_MALE_PATH) == MII_OK)
    mii_model_set_body(&model, &body);             /* the Wii's own; else the library's */

/* Each frame, in your 3D pass: a viewport where the Mii stands, the camera
 * that frames it there, then draw. Set your GX state again after. */
if (ok) {
    Mtx44 proj;
    Mtx view;
    mii_pose pose = { 0 };

    pose.anim = 1 + mii_body_anim(&body, "bodyWait");   /* 0: stand still */
    pose.frame = seconds * 60;
    pose.expression = MII_EXPR_SMILE;
    GX_SetViewport(x, y, 128, 208, 0, 1);
    mii_view(&model, &pose, 128.0f / 208, 0.04f, proj, view);
    GX_LoadProjectionMtx(proj, GX_PERSPECTIVE);
    mii_draw(&model, &pose, view, NULL);
}
```

- **The face models are the user's.** `RFL_Res.dat` is Nintendo's: never
  ship it. HBC's HOME > HBC > Save Mii faces (or `hbc.py mii-faces`) copies
  the Wii's own to `sd:/mii/RFL_Res.dat`, where `mii_res_load(&res, NULL)`
  looks. `MII_E_NO_FILE` means: ask the user to save it.
- **Or straight from the Wii, with nothing on SD.** `mii_res_load_nand(&res)`
  reads the Wii Menu's own `RFL_Res.dat` into memory, and
  `mii_bodies_load_nand(&male, &female)` the Mii Channel's bodies (either
  may be NULL; one `Character.szs` is decoded for both). IOS refuses these
  files to apps, so `miinand.c` lifts its NAND permission check for the
  reads, with the patch Save Mii faces uses, and puts back exactly what it
  changed. That needs AHBPROT: `<ahb_access/>` in your `meta.xml` (Wiiload
  gives it too), and no IOS reload before the call. Without it the calls
  return `MII_E_DENIED` and touch nothing, so try the SD copy next:

  ```c
  if (mii_res_load_nand(&res) < 0 && mii_res_load(&res, NULL) < 0)
      ;                                            /* no faces: ask the user */
  ```

  About 0.1 s for the faces and 0.1 s for the bodies in Dolphin; call them
  outside the frame loop.
- **`mii_model_make`** builds everything once: the head's shapes placed as
  RFL places them, the face composited for all seven expressions
  (`MII_EXPR_*`), the parts RFL colours with TEV tricks baked into plain
  textures, a body in the Mii's colours (shirt in the favourite colour,
  trousers grey or gold for a special Mii) scaled by height and build, and
  each part as a GX display list. As RFL keeps them: textures RGB5A3,
  shapes as s16 vertices drawn indexed. About 280 KB with 128 px faces;
  most of it is the seven faces.
- **`mii_model_build`** takes `mii_model_opts`: the face size (32 to 256
  px) and which expressions to build (`MII_EXPR_BIT`; the normal face is
  always built, and one not built draws as it). With 128 px faces and the
  normal one only, a Mii takes about 67 KB; with 32 px, about 37 KB.
- **Ways to draw it:**
  - In place, each frame: `mii_view()` gives the projection and camera
    that frame the Mii in a viewport of the aspect you pass; set your
    viewport there and `mii_draw()`. Give the Mii a wide depth range: its
    face's layers break up on a real Wii in a thin one (Dolphin hides it).
    HBC's avatar does this (channel/channelapp/source/avatar.c): the whole
    range in front of its slot, then its rectangle's depth put back.
  - `mii_capture()` into a texture with alpha, framed whole (or the head:
    `pose.head_only`). It draws twice the size and box-filters down, so the
    edges are smooth. For icons and stills.
  - `mii_icon()`: a head icon framed as the Wii Menu's.
  - `mii_draw()` into your own 3D scene, with your projection and camera.
- **`mii_pose`**: turn, head yaw/pitch/roll, arms out and forward, legs,
  a bob, breathing, the expression. All zero stands still.
- **The Wii's own bodies.** With `sd:/mii/maleBody.brres` and
  `femaleBody.brres` (Save Mii faces copies them from the Mii Channel too),
  `mii_body_load` reads one and `mii_model_set_body` puts it under a model:
  Nintendo's skeleton, the shirt and trousers skinned on the GPU from the
  Mii Channel's own display lists (a matrix per bone a frame), and its
  animations. Pose it with `pose.anim = 1 +
  mii_body_anim(&body, "bodyWait")` (or `bodyGreeting`, `bodyWalk`,
  `bodyRun`, `bodyTalk`, ...) and `pose.frame` at 60 a second; 0 stands
  still. When you change animation, cross-fade instead of snapping: keep
  the one before in `pose.from_anim` / `from_frame` with
  `pose.from_weight = mii_fade_weight(seconds_since, 0.4f)`. A looping
  animation plays on as it fades; a finished one-shot (a wave) holds its
  last frame, since frames wrap and it would otherwise fade toward its own
  start. Each `mii_anim` says which it is (`loop`, from the file: `bodyWait`,
  `bodyWalk`, `bodyRun`, `bodyTalk`, `bodyTalk_01` and `bodyCatnap_00` go
  round; the rest play once), and `mii_anim_frame(&body, anim, seconds,
  &done)` gives the frame for either: wrapped for a looping one, held at
  the end for a one-shot, with `done` set once it has ended.
  Height and build scale each bone as Nintendo's later libraries do.
  Without the files the model keeps a body of the library's own.
- **The Mii Channel's look** (1.4.0), read from the body's own materials:
  - **Shading.** The shirt and trousers use no lamp. Each one's colour is
    lifted by the square of `env0`, a 32 × 32 light texture, looked up by
    where each normal points as the camera sees it. This gives the soft
    highlight on the chest. A body file without `env0` is lit as before.
  - **Shadow.** `pose.shadow = 1` draws the body's own shadow under the
    feet: a 75-unit square with a radial texture, in the material's colour,
    on the floor at any height and build (1.5.1; before, it rose and sank
    with the leg length, up a tall Mii's legs, under a short one's feet).
    The animation sizes it: it shrinks to nothing as the Mii is lifted
    (`bodyGetDown`) and stretches as the Mii falls flat (`bodySlip_00`). For
    a Mii carried above the ground, set `pose.shadow_drop` to the distance
    down to the ground; the shadow stays there and is smaller the higher the
    Mii is. `mii_body_shadow` gives its placement, for drawing it yourself.
  - **Faces.** `mii_anim_expression(&body, pose.anim, pose.frame, face)`
    gives a face for the animation: eyes shut asleep (`bodyCatnap_00`), a
    smile for `bodyGreeting`, surprise when flung or slipping, the mouth
    going for `bodyTalk`. Where the animation calls for none, it returns
    your `face`. The file does not say which face goes with which animation,
    so these choices are the library's own (see
    [For someone to add](#for-someone-to-add)).

  ```c
  static mii_body body;
  if (mii_body_load(&body, m.sex == MII_FEMALE ? MII_BODY_FEMALE_PATH
                                               : MII_BODY_MALE_PATH) == MII_OK)
      mii_model_set_body(&model, &body);           /* body outlives model */
  pose.anim = 1 + mii_body_anim(&body, "bodyWait");
  pose.frame = seconds * 60;
  ```
- **Effects** (`miifx.c`). The Mii Channel's particle effects: a sneeze's
  splash, speech balloons, dust from running, slipping and landing, the
  Zzz of a nap, the sparkles of a favourite and of a changed face part. They
  are in its `Character.szs/effect` as `effect_nigaoe.breff` (14 emitters)
  and `effect_nigaoe.breft` (6 textures). These are Nintendo's, like the
  bodies: the user copies their own to `sd:/mii`, and nothing ships them.
  `mii_fx_load` reads them. A `mii_fx_state` (about 25 KB, no allocation
  after that) plays up to 16 emitters and 128 particles; when it is full,
  the oldest goes, and `fxs.replaced` counts the emitters that went that
  way, by `mii_fx_play` or by the children particles start in
  `mii_fx_step` (1.5.0). `mii_fx_play` starts an emitter, placed by a
  matrix in the world that you give. `mii_fx_place(&model, &pose, head,
  at)` makes it in the Mii's own space: on the head
  bone as posed (`head` 1: the sneeze in front of the face and where it
  points, talk balloons before the mouth, zzz over the head; the same
  place the animations give as `pSphere14`), or on the ground under the
  Mii's lowest bone (0: dust where the feet, or the back, hit), turned with
  the Mii and sized by its height. For a `head_only` pose the chin is the
  origin. That is the world only for a Mii drawn without a `place`: for
  one drawn with `mii_draw(&model, &pose, view, place)`, use
  `mii_fx_place_in(&model, &pose, head, place, at)` (1.5.0), or its
  effects play at the world's origin. Placed after the Mii is drawn, with
  `pose.reuse = 1`, they take the bones it was drawn with and pose nothing
  again (`mii_body_pose` too). `mii_icon_fx` draws an icon with the effects over it: play them
  on a `head_only` pose's place, scaled to about half, to keep them inside.
  `mii_fx_step` moves the effects on, at 60 frames a second. `mii_fx_draw`
  draws them as squares that face the camera, coloured and blended as the
  file says, and leaves Z writes on (1.5.0: an emitter's Z mode with
  writes off, left to the frame's copy, kept that copy from clearing Z,
  and the next frame drew nothing where the Miis had stood: a crowd of
  them in the clear colour). Draw them after the opaque parts of the
  scene. `Nc_zzz01`
  goes on until `mii_fx_stop`. On the PC, `mii_fx_particles` lists each
  particle (`tests/test_miifx.py`).

  ```c
  static mii_fx fx;
  static mii_fx_state fxs;
  if (mii_fx_load(&fx, NULL, NULL) == MII_OK)      /* sd:/mii/effect_nigaoe.* */
      mii_fx_init(&fxs, &fx, 1);
  float head[3][4];
  mii_fx_place_in(&model, &pose, 1, place, head);  /* place: as mii_draw's, or NULL */
  int zzz = mii_fx_play(&fxs, mii_fx_find(&fx, "Nc_zzz01"), head);
  /* each frame, after mii_draw(&model, &pose, view, place): */
  pose.reuse = 1;                                  /* the bones just drawn */
  mii_fx_place_in(&model, &pose, 1, place, head);
  mii_fx_move(&fxs, zzz, head);                    /* move with the head */
  mii_fx_step(&fxs, 1);
  mii_fx_draw(&fxs, view);                         /* after the Miis */
  ```
- **Many Miis.** A crowd is what the Wii's CPU and memory allow, not its
  GPU, until the CPU's part is small. What helps, all measured on the
  bench Wii (REVIEW.md, 1.11.6 to 1.11.10):
  - One `mii_cache` for the crowd (`mii_model_opts.cache`): Miis share
    each part that comes out the same, byte for byte (the same hair, face
    shape, textures). With 32 px faces and the normal one only, a random
    Mii then takes about 12 to 20 KB; the bench Wii held 5,855.
  - `mii_body_make_lods(&body, NULL, 0)` once per body,
    `mii_model_opts.lods = MII_HEAD_LODS` for each Mii, then each frame
    `pose.lod = mii_body_lod_for(&body, pixels_tall)`: four coarser levels
    of body and head for Miis under 90, 39, 30.5 and 25 pixels tall (chosen
    by eye on a real Wii), each changing the Mii by under half a pixel
    there, made on the Wii from the user's own files. A head of about 490
    triangles goes to about 380, 340, 270 and 180 (the hair folds at half
    the tolerance: its thin pieces cracked), the body's 750 to about 490,
    410, 320 and 240, all drawn as strips. The GPU's time in a crowd is
    vertices. `tests/mii_app parade` walks Miis through the levels over
    lines on the grass where each begins.
  - Drawing is the hardware's work: the bones on the paired-single unit,
    each head level a display list the GPU fetches itself, the matrix
    palette's cache lines claimed with `dcbz`. A Mii far off costs the CPU
    about 16 us a frame. Build with `-DMII_PROFILE` to see where
    (`mii_profile`).
  - `mii_model_opts.gpu_faces = 1`: faces drawn by the GPU, as RFL draws
    them (`RFLiMakeTexture`): about 0.08 ms each against 1 to 6 ms on the
    CPU, within 0.1 of 255 of the CPU's on average. They use the EFB, so
    call `mii_model_make_faces` between frames on the thread that draws.
  - Faces by distance: `mii_model_set_mask_size` (32 px far, 128 near);
    with GPU faces it marks them, and `mii_model_make_faces` draws them
    between frames.
  - Walk in step: Miis at the same frame of the same animation pose once
    (the body remembers its last 16 poses), so give a crowd a few phases,
    not one each.
  - `pose.reuse` for a Mii far off: draw it with the bones its last draw
    posed, posing every second or fourth frame. Place its effects after
    the draw with `reuse` set too: they take those bones, not posed again.

  ```c
  /* Once: a cache for the crowd, the bodies' levels. */
  mii_cache *cache = mii_cache_new();
  mii_body_make_lods(&body, NULL, 0);
  mii_model_opts o = { 32, MII_EXPR_BIT(MII_EXPR_NORMAL), cache, MII_HEAD_LODS, 1 };
  for (i = 0; i < n; ++i)
      if (mii_model_build(&res, &miis[i], &o, &crowd[i]) == MII_OK)
          mii_model_set_body(&crowd[i], &body);

  /* Each frame, before drawing: faces still to draw (the GPU, between frames). */
  for (i = 0; i < n; ++i)
      if (crowd[i].faces_pending)
          mii_model_make_faces(&crowd[i], &res, clear_colour);
  /* ... set your GX state, projection and camera, then for each Mii: */
  float px = mii_pixels_tall(&crowd[i], view, place, proj, rmode->efbHeight);
  pose.lod = mii_body_lod_for(&body, px);
  pose.reuse = pose.lod >= 3 && (frame + i) % 2;   /* far: pose every other frame */
  mii_draw(&crowd[i], &pose, view, place);

  /* At the end: the models, then the cache and the body. */
  ```
- **Your GX FIFO in MEM1.** Make it a static array
  (`static u8 fifo[SIZE] ATTRIBUTE_ALIGN(32);`), not `memalign` from the
  heap: once MEM1's part of the heap runs out, the heap gives MEM2, and a
  GPU fed its commands from MEM2 stalled at random on a real Wii (never in
  Dolphin). HBC lost days to this when linking this library grew its .bss
  (REVIEW.md, 1.10.26). A Mii adds under 2 KB of commands a frame (its
  lists and arrays stay in memory, where the GPU reads them).
- **Builds with libogc 3 and libogc2.** HBC uses libogc 3; every function
  here also compiles and links with devkitPPC r41-2 and libogc2.
- **Between frames.** `mii_capture` and `mii_icon` draw in the EFB's corner
  (RGBA6 while they draw), copy it out and clear it: call them after your
  last screen copy and before you draw the next frame, then set your GX
  state again (pixel format, viewport, scissor, projection, copy filter,
  vertex format, colour channel). `mii_draw` changes the vertex format,
  lighting, TEV, Z, culling and blending.
- **Z writes on at your copy.** A copy that clears (`GX_CopyDisp(xfb,
  GX_TRUE)`) clears Z only while Z writes are on (`GX_SetZMode`), on the
  Wii and in Dolphin. `mii_draw`, `mii_fx_draw` and `mii_model_make_faces`
  leave them on; if your own drawing ends with them off, turn them on
  before the copy, or the next frame is drawn against this one's Z.
- **Testing in Dolphin**: an EFB copy stays on Dolphin's GPU unless
  `Graphics.Hacks.EFBToTextureEnable=False`, so a CPU read of `mii_capture`'s
  texture sees zeros there; drawing it works either way.
- **On the PC**: with `-DMII_NO_OGC` everything but drawing builds
  (`tests/miidraw_host` writes a Mii's meshes as OBJ and its faces as RGBA;
  with `MII_BODY`, a Wii body posed by `MII_ANIM` and `MII_FRAME`;
  `tests/miifx_host` plays an effect and prints its particles).
- **Files**: `miidraw.c` (faces, the built body, drawing), `miibody.c`
  (the Wii's bodies), `miifx.c` (the effects) and `miinand.c` (faces and
  bodies read from NAND) with `mii.c`.

The head's numbers come from RFL as the CC0 decompilations show it; the
code is this library's own. [docs/research/mii-rendering.md](https://github.com/Monsterray/hbc-reborn/blob/master/docs/research/mii-rendering.md)
has the sources.

## Your own matrix maths

libogc's `guMtxConcat` (paired-single `ps_guMtxConcat`) loads a constant with
a `psq_l` through `r13`, whose offset has 12 bits. devkitPPC's linker scripts
put the small-data base 32 KiB past `.sdata`, so the offset is cut short and
the load reads the wrong word: every product it makes is off. The library uses
the C one. For your own calls, use `c_guMtxConcat`, or set the base at the
start of `.sdata` in your linker script (`_SDA_BASE_ = .;`, as HBC's
`newrvl.ld` does). Check with `powerpc-eabi-objdump -d app.elf | grep
'psq_l.*(r13)'`: the last-but-one number must be 0.

## For someone to add

MiiKit uses everything in the Mii Channel's body and effect files.
These are the parts it does not have yet. Each one needs knowledge from
outside those files.

- **Height and build as the Wii does them.** Bones are scaled with the
  formula of Nintendo's later Mii library, the Wii U's (FFL), because the
  Wii's own formula is not in the data. It lives in the Mii Channel's
  program, or in RFL as games use it. To check it: pose a tall, thin Mii
  and a short, wide one in the Mii Channel in Dolphin, and compare with
  `mii_body_pose`.
- **Wii U and 3DS Miis.** `mii_import` takes only the Wii's 74- and 76-byte
  forms. The Wii U and 3DS store a Mii in 96 bytes (FFLStoreData, Ver3),
  with more hair styles, eye and lip colours, and a different creation ID.
  Converting one to a Wii Mii means choosing the nearest Wii part for each
  new one. Nintendo's own Mii transfer between Wii and Wii U is a reference
  for those choices.
- **Wii U drawing.** FFL lights each part with its own material: a rim
  light, a specular light on the hair, and textures in more formats. The
  Wii has none of these, so drawing a Wii U Mii as the Wii U does would
  need FFL's shader values (FFLShaderData) and a TEV version of them.
- **The Mii Channel's faces and effects timing.** No file says which face
  or effect goes with which frame of an animation. `mii_anim_expression`
  and HBC's effect cues are this library's choices, by eye (HBC finds the
  landings from the bones: dust where the lowest one comes down). The real ones
  are in the Mii Channel's program, as is the matrix each effect is placed
  by. Talk and the sneeze fit the head, run smoke the body's root, and zzz
  neither exactly.
- **pSphere14.** Nineteen animations move a node that the body does not
  have. It is the head bone's place in model space, baked in, and the
  effects on the head are placed by it: `mii_fx_place` uses the head bone,
  which gives the same place without reading it.
