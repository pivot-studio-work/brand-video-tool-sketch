# Brand Video Tool — Test Sketch

Browser-based brand video editor sketch. Left: asset gallery (.mp4, .mov, Cavalry .cv). Right: preview with play/pause. Below: drag-and-drop timeline. Deploys to Vercel.

**The one thing this sketch must prove:** two or more Cavalry scenes on the timeline play in sequence alongside a transparent bumper video and a talking-head .mp4, frame-accurately, in one preview player.

Full spec: `docs/SPEC.md`. Decisions log: `docs/DECISIONS.md`. Session handoff: `docs/HANDOFF.md`.

## Stack
- Next.js (App Router) + TypeScript (strict) + Tailwind
- Remotion: `remotion`, `@remotion/player`, `@remotion/media` (+ `@remotion/web-renderer` for the export stretch goal)
- `@dnd-kit/core` + `@dnd-kit/sortable` for drag and drop
- Zustand for editor state
- Vitest for unit tests on timeline math
- Cavalry Web Player (WASM, beta) vendored in `public/cavalry/`
- Package manager: pnpm

## Commands
- `pnpm dev` — local dev
- `pnpm build` — must pass before any commit
- `pnpm test` — Vitest
- `pnpm lint` / `pnpm typecheck`

## Architecture rules (do not break)
1. **Remotion owns time.** The Remotion `<Player>` is the only clock. Cavalry scenes are never played with `play()`/`tick()`; each frame, `<CavalryBlock>` calls `setFrame()` then `render()` based on `useCurrentFrame()`.
2. **The timeline is data.** `src/lib/timeline/` holds pure TypeScript (no React) for the timeline model and frame math. The UI edits that data; `src/remotion/` renders it. Everything in `src/lib/timeline/` has unit tests.
3. **One Cavalry WASM module, many scene instances.** Load the runtime once (singleton in `src/lib/cavalry/runtime.ts`). Each block gets its own canvas and its own `Cavalry` instance, created on mount and freed on unmount.
4. **Mount only what's needed.** Every timeline item is a Remotion `<Sequence>` with `premountFor`. Never keep every Cavalry instance alive (browsers cap WebGL contexts at about 16).
5. **Stacking = DOM order.** Render track V1 first, then V2 overlays. Don't rely on `z-index` (Remotion's web renderer ignores it).
6. All Remotion and Cavalry code is client-only (`'use client'`, dynamic import with `ssr: false` where needed).
7. Pin every `remotion` / `@remotion/*` package to the **same exact version** (no `^`).

## Cavalry Web Player: ground truth
- The API is beta and thinly documented. **Do not guess method signatures.** Before writing Cavalry code, read the examples in `vendor/cavalry-web-player/` (the unzipped official package) and copy their loading pattern exactly. That includes how scene files and assets get into the virtual filesystem, `locateFile`, and the `cavalryAutoLoadAsset` event.
- Docs: https://cavalry.studio/docs/web-player/ and https://cavalry.studio/docs/web-player/api/
- Known limits: no audio; JavaScript layers, Jsmath and Smart Folder layers are unsupported; scenes must be saved from Cavalry 2.5+.
- Use playback mode `0` (Accurate). Frame mapping lives in `src/lib/timeline/frames.ts`.
- Check the bundled `serve.py` for any special headers (MIME types, COOP/COEP). Anything it sets must be mirrored in `next.config.ts` headers / `vercel.json`.

## Video formats
- Chrome/Firefox can't play ProRes .mov. Transparent assets need **WebM VP9 + alpha** (Chrome/Firefox) and optionally **HEVC-with-alpha .mov** (Safari). An asset can have multiple `sources`; pick one at runtime with `canPlayType`.
- Opaque .mp4/.mov must be H.264 or HEVC.
- Video playback uses `<Video>` from `@remotion/media`.

## How to work in this repo
- Work one phase at a time from `docs/SPEC.md` §8. Start each phase in plan mode, propose the plan, then build.
- A phase is done only when its acceptance checks pass **in the browser**, `pnpm build` passes, and it's committed.
- After each phase, update `docs/HANDOFF.md` (state, what works, what's next, known issues) and log non-obvious choices in `docs/DECISIONS.md`.
- When something about Cavalry behaves unexpectedly, write the finding in `docs/DECISIONS.md` under "Cavalry findings". That knowledge is the main output of this sketch.
- Keep it a sketch: no auth, no database, no styling polish beyond clean and usable.
