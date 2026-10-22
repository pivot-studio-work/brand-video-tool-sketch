# Brand Video Tool — Test Sketch Spec

## 1. Purpose

Test the foundations of a browser-based brand video tool for Amp: Cavalry templates, branded bumpers and client footage, arranged on a timeline and previewed (later exported) in the browser, deployed on Vercel.

This is a **feasibility sketch**, not a product. Success means answering these questions with evidence:

| # | Question | Pass condition |
|---|----------|----------------|
| Q1 | Can the Cavalry Web Player run inside a Remotion composition? | A `.cv` scene renders in the Remotion `<Player>`, and scrubbing shows the correct frame |
| Q2 | Can **multiple** Cavalry scenes play in sequence? | Cavalry A then Cavalry B play back to back with no blank frames, flashes or stale frames at the cut |
| Q3 | Can Cavalry mix with real video? | Sequence: transparent bumper → Cavalry A → Cavalry B → talking-head .mp4 plays smoothly with audio |
| Q4 | Does transparency composite correctly? | The bumper's alpha shows the layer beneath it, with no black box |
| Q5 | Does it work deployed? | All of the above works on the Vercel URL in Chrome; Safari behaviour is documented |
| Q6 (stretch) | Can it export? | `renderMediaOnWeb()` produces an MP4 that includes the Cavalry frames, not blank ones |

## 2. UI

```
┌──────────────┬─────────────────────────────────────────┐
│  ASSETS      │                                         │
│  ┌──┐ ┌──┐   │            PREVIEW (16:9)               │
│  │  │ │  │   │         Remotion <Player>               │
│  └──┘ └──┘   │                                         │
│  ┌──┐ ┌──┐   │      ◀◀   ▶/❚❚   00:04:12 / 00:21:00     │
│  │  │ │  │   ├─────────────────────────────────────────┤
│  └──┘ └──┘   │ TIMELINE   |playhead                    │
│  [+ Add]     │ V2 ░░[bumper]░░░░░░░░░░░░░░░░░░░░░░░░░  │
│              │ V1 [cav A][cav B][talking head.mp4    ] │
└──────────────┴─────────────────────────────────────────┘
```

- **Asset gallery (left).** Grid of thumbnails with a type badge (MP4 / MOV / CAV) and duration. Transparent assets show over a checkerboard. Items are draggable.
- **Preview (right).** Remotion `<Player>`, 1920×1080 at 30fps, scaled to fit. Play/pause button, spacebar toggles, timecode readout.
- **Timeline (below the preview).**
  - Two tracks:
    - **V1** is the main track and magnetic: items snap end to end in order.
    - **V2** is an overlay track: items sit at a free start frame and draw above V1. That's where transparent bumpers and lower-thirds go.
  - Drag from the gallery onto a track to add an item.
  - Drag within V1 to reorder. Drag horizontally on V2 to move.
  - Select an item and press Delete to remove it.
  - Click the ruler to seek. The playhead follows playback.
  - Clip width = duration × zoom (px per frame). A zoom slider is nice to have.

## 3. Assets

### 3.1 Phase 1 source: seeded files
Put test files in `public/assets/` and describe them in `public/assets/manifest.json`. No uploads at first, which takes storage out of the experiment.

Also support **local drop**: dragging files from Finder onto the gallery creates `URL.createObjectURL()` entries. It's session-only and costs nothing.

### 3.2 Later: uploads
Use Vercel Blob **client uploads** (the browser uploads straight to Blob). Server-function uploads hit Vercel's ~4.5MB request body limit, so they won't work for video. This is out of scope for the sketch unless it's needed to test the deployed URL with real client files.

### 3.3 Asset model
```ts
type AssetKind = 'video' | 'cavalry';

interface AssetSource {
  url: string;
  mime: string;   // e.g. 'video/webm; codecs="vp09.00.10.08"', 'video/mp4', 'video/quicktime; codecs="hvc1"'
}

interface Asset {
  id: string;
  name: string;
  kind: AssetKind;
  sources: AssetSource[];       // video: pick the first one the browser can play (canPlayType). cavalry: one .cv URL
  hasAlpha: boolean;
  width: number;
  height: number;
  fps: number;                  // native fps (Cavalry: getFPS())
  durationInFrames: number;     // at native fps
  thumbnailUrl?: string;        // generated, data: or blob: URL
  cavalry?: {
    sceneUrl: string;
    assetUrls?: Record<string, string>;  // images/fonts the scene needs
    startFrame: number;                  // getStartFrame()
    endFrame: number;                    // getEndFrame()
  };
}
```

### 3.4 Metadata and thumbnails
- **Video:** load a hidden `<video>`, wait for `loadedmetadata` to read duration and size, seek to ~10% (or 1s), draw to a 2D canvas and save it as `toDataURL('image/webp')`.
- **Cavalry:**
  - Load the scene in an offscreen instance and read `getSceneResolution()`, `getFPS()`, `getStartFrame()` and `getEndFrame()`.
  - Call `setFrame(midpoint)` then `render(surface)`, and **in the same task** draw the WebGL canvas into a 2D canvas and save it as `toDataURL`. The WebGL buffer may be cleared once control returns to the browser, which is the same issue Q6 hits.
  - Dispose the instance afterwards.
- Generate thumbnails one at a time, not in parallel, to stay under the WebGL context limit.

## 4. Timeline model (pure TS, `src/lib/timeline/`)

```ts
type TrackId = 'V1' | 'V2';

interface TimelineItem {
  id: string;
  assetId: string;
  track: TrackId;
  startFrame: number;          // composition frames (30fps)
  durationInFrames: number;    // composition frames
  trimStartFrames: number;     // composition frames trimmed off the asset start (0 for the sketch)
}

interface Timeline {
  fps: 30;
  width: 1920;
  height: 1080;
  items: TimelineItem[];
}
```

Functions (all unit-tested):
- `addItem(timeline, asset, track, index | startFrame)`. On V1, insert at index and repack. On V2, place at startFrame.
- `moveItem`, `removeItem`.
- `repackV1(items)`: V1 start frames become the running sum of durations.
- `totalDuration(timeline)`: the latest item end across all tracks, minimum 1 frame.
- `assetFramesToComp(asset, compFps)`: converts the asset's native duration to composition frames.
- `compFrameToCavalryFrame(localCompFrame, compFps, cav)`: returns `cav.startFrame + floor(localCompFrame * cav.fps / compFps)`, clamped to `cav.endFrame`. It must be **deterministic**: the same input always gives the same output.

## 5. Rendering (`src/remotion/`)

### 5.1 Composition
`BrandVideo.tsx` takes the `Timeline` and asset map as `inputProps`:
```tsx
<AbsoluteFill style={{ background: '#000' }}>
  {v1Items.map(i => <Sequence key={i.id} from={i.startFrame} durationInFrames={i.durationInFrames} premountFor={30}><ItemRenderer item={i}/></Sequence>)}
  {v2Items.map(i => /* same, rendered AFTER V1 so it stacks on top */)}
</AbsoluteFill>
```
`durationInFrames` on the Player = `totalDuration(timeline)`.

### 5.2 Video items
`<Video>` from `@remotion/media`. Pass the source URL chosen at runtime and `trimBefore={trimStartFrames}`. Set `objectFit: contain` inside an `AbsoluteFill`.

### 5.3 `<CavalryBlock>`: the core of the experiment
```
mount:
  delayRender('cavalry-load')
  runtime = await getCavalryRuntime()          // singleton: loads CavalryWasm.js + .wasm once
  load scene (+ assets/fonts) into the virtual FS  // copy the official example's pattern exactly
  canvas (1920×1080, a real <canvas> in the DOM)
  surface = Module.makeWebGLSurfaceFromElement(canvas, w, h)
  player  = Module.Cavalry.MakeWithPath(path)
  player.setPlaybackMode(0)                    // Accurate
  continueRender()

every frame (useCurrentFrame → effect):
  f = compFrameToCavalryFrame(frame, ...)
  player.setFrame(f); player.render(surface)
  // if the frame comes out blank: copy WebGL canvas → visible 2D canvas here, same task (see §6)

unmount:
  free player/surface (embind objects usually need .delete(); check the examples)
```
- Never call `play()`, `tick()` or `setLoop()`. Remotion is the clock.
- `premountFor` loads the next Cavalry scene before its cut, so the cut doesn't show a blank frame. **This is what Q2 tests.** If one second of premount isn't enough, measure the load time and raise it.
- Fonts: call `Module.loadFontData(url)` once in the runtime singleton, before any scene that needs them.
- A Cavalry scene shorter than its timeline slot holds its last frame. A longer one is cut off at the end of its slot.

### 5.4 Player wiring
- `<Player component={BrandVideo} inputProps={{ timeline, assets }} compositionWidth={1920} compositionHeight={1080} fps={30} durationInFrames={total} acknowledgeRemotionLicense />`
- Use a Player ref: listen to `frameupdate` to move the timeline playhead, call `seekTo()` when the ruler is clicked, and call `toggle()` from the button and spacebar.
- `inputProps` must be memoized, because recreating it re-renders the whole composition.

## 6. Known risks and what to do about each

| Risk | Symptom | Response |
|------|---------|----------|
| WebGL canvas cleared between frames | Blank or black Cavalry frames in export or thumbnails | Draw the WebGL canvas into a 2D canvas right after `render()`, then display and capture the 2D canvas. Alternatively, check whether the runtime accepts a context created with `preserveDrawingBuffer: true` |
| Too many WebGL contexts | "Too many active WebGL contexts" warning; old canvases go blank | Lower `premountFor`, make sure instances are freed on unmount, and render only the current and next scene |
| Cavalry scene loads slowly at a cut | First frames of scene B blank | Increase `premountFor`; optionally warm-load scene files with `fetch` into a cache |
| Jumping to a frame gives different output than playing up to it | Scrubbing shows different frames than playback | Log it in DECISIONS; test with simulation/dynamics-heavy scenes. Fallback: pre-render that scene to WebM |
| `.mov` won't play in Chrome | Video errors or fallback warnings | Transcode (§7). ProRes is editing-only |
| Alpha lost | Black box behind bumper | Use VP9 alpha WebM; confirm `hasAlpha` sources are selected |
| WASM MIME/headers on Vercel | Runtime fails to load in production only | Mirror `serve.py` headers in `next.config.ts` `headers()` |
| Cavalry Web Player license | Unknown commercial terms (beta) | Keep the repo **private**. Don't ship to clients until Scene Group confirms terms |

## 7. Test assets to prepare

| File | Purpose | How |
|------|---------|-----|
| `bumper-alpha.webm` | Transparent bumper, Chrome/Firefox | `ffmpeg -i bumper_prores4444.mov -c:v libvpx-vp9 -pix_fmt yuva420p -b:v 0 -crf 30 -an bumper-alpha.webm` |
| `bumper-alpha-hevc.mov` | Transparent bumper, Safari | macOS: `ffmpeg -i bumper_prores4444.mov -c:v hevc_videotoolbox -alpha_quality 0.75 -tag:v hvc1 -an bumper-alpha-hevc.mov` (or Compressor/Media Encoder "HEVC with alpha") |
| `talking-head.mp4` | Speaker footage with audio | H.264 + AAC, 1080p, 30fps, 10–20s |
| `opaque.mov` | Checks that .mov in general works | H.264 in a .mov container |
| `scene-a.cv`, `scene-b.cv` | Two Cavalry templates | Saved from Cavalry 2.5+, 1920×1080, 30fps, 3–6s, no JavaScript layers/Jsmath/Smart Folders, fonts collected |
| `scene-sim.cv` (optional) | Determinism stress test | A scene with simulation, dynamics or noise driven by time |

Match fps (30) across everything for the first pass. Test a 24/25/60fps Cavalry scene only after Q1–Q3 pass.

## 8. Build phases

Each phase ends with a commit, a HANDOFF update and a browser check.

**Phase 0 — Scaffold and deploy (prove the pipeline first)**
- Next.js + TS + Tailwind + pnpm. Add the Remotion packages pinned to the same version. Add Vitest.
- Connect to a private GitHub repo, then import into Vercel.
- ✅ "Hello" Remotion `<Player>` with a colored square animating, live on the Vercel URL.

**Phase 1 — Cavalry spike (no Remotion)**
- Unzip the official package into `vendor/cavalry-web-player/` (reference) and copy the runtime files into `public/cavalry/`.
- `/spike/cavalry` page: load `scene-a.cv`, frame slider → `setFrame` + `render`.
- ✅ Scrubbing shows correct frames locally **and on Vercel**. Record load time and WASM size in DECISIONS.

**Phase 2 — The critical test, with a hard-coded timeline (no UI)**
- `src/lib/timeline/` with tests. `BrandVideo` composition. `<CavalryBlock>`. Video item renderer.
- `/spike/sequence` page: Player with a hardcoded timeline: V1 = [scene-a, scene-b, talking-head.mp4], V2 = bumper-alpha at frame 0 (and a second test with the bumper over the A→B cut).
- ✅ Q1–Q4 pass in Chrome. Play through three times plus random scrubbing: no blank frames at cuts, no stale frames, audio in sync. Write the results into HANDOFF.
- **If this phase fails, stop and report. The UI phases don't matter until it passes.**

**Phase 3 — Asset gallery**
- Manifest loader, metadata probing, thumbnail generation (§3.4), local file drop.
- ✅ All test assets show correct thumbnails, durations and badges.

**Phase 4 — Timeline UI**
- dnd-kit: gallery → track drop, V1 reorder, V2 positioning, delete, ruler seek, playhead sync, play/pause + spacebar.
- ✅ A user can build the Phase 2 sequence by hand and it plays the same way.

**Phase 5 — Deployed test pass**
- Test matrix on the Vercel URL: Chrome, Safari, Firefox × (Q1–Q5). Record it in HANDOFF as a table.

**Phase 6 — Stretch: export**
- Export button → `renderMediaOnWeb()` → download MP4. Apply the §6 WebGL fix if frames are blank.
- ✅ Q6. Record render time against video length.

## 9. Folder structure

```
src/
  app/
    page.tsx                 # editor layout
    spike/cavalry/page.tsx   # Phase 1
    spike/sequence/page.tsx  # Phase 2
  components/
    AssetGallery.tsx
    PreviewPanel.tsx
    Timeline/ (Timeline.tsx, Track.tsx, Clip.tsx, Ruler.tsx, Playhead.tsx)
  lib/
    timeline/ (types.ts, model.ts, frames.ts, *.test.ts)
    cavalry/  (runtime.ts, loadScene.ts, types.ts)
    assets/   (manifest.ts, probe.ts, thumbnails.ts, pickSource.ts)
  remotion/
    BrandVideo.tsx
    ItemRenderer.tsx
    CavalryBlock.tsx
    VideoItem.tsx
  store/editor.ts            # Zustand
public/
  cavalry/                   # runtime files served to the browser
  assets/                    # test media + manifest.json
vendor/cavalry-web-player/   # official package, reference only
docs/ SPEC.md, DECISIONS.md, HANDOFF.md
```

## 10. Out of scope (for this sketch)
Auth, database, multi-user, Cavalry Control Centre editing UI (next sketch: auto-generate controls from `getControlCentreLayout()`), brand kit, transitions, audio tracks, trimming handles, server/Lambda rendering, mobile layout.

## 11. Licensing notes
- **Remotion:** free for organisations of 3 people or fewer. Client-side renders send usage data to Remotion, and that can't be turned off. Set `acknowledgeRemotionLicense` on the Player.
- **Cavalry Web Player:** it's in beta, and the docs give no commercial terms. Keep everything private and get terms from Scene Group before any client-facing use.

## References
- Cavalry Web Player: https://cavalry.studio/docs/web-player/ · API: https://cavalry.studio/docs/web-player/api/
- Remotion Player: https://www.remotion.dev/docs/player · `<Video>`: https://www.remotion.dev/docs/media/video
- Remotion client-side rendering: https://www.remotion.dev/docs/client-side-rendering (limitations: /limitations)
- Remotion transparent videos: https://www.remotion.dev/docs/transparent-videos
- Vercel Blob client uploads: https://vercel.com/docs/vercel-blob/client-upload
