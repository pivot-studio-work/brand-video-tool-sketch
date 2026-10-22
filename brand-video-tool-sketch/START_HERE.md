# Start Here

How to kick off this project in VS Code with Claude Code.

## 1. Before opening Claude Code (about 30 min)
1. Create a new empty folder for the repo, e.g. `~/Code/brand-video-tool`.
2. Copy `CLAUDE.md` and the `docs/` folder from here into it.
3. Download the Cavalry Web Player package (`cavalry-web-player.zip`) from https://cavalry.studio/docs/web-player/ and unzip it to `vendor/cavalry-web-player/` inside the repo.
4. Prepare the test assets listed in `docs/SPEC.md` §7, and put them in a folder called `test-assets/` in the repo (Claude will move them into `public/assets/`).
5. Create a **private** GitHub repo. Have a Vercel account ready.

## 2. Session rhythm (best practices)
- **One phase per session.** Run `/clear` between phases. CLAUDE.md, SPEC.md and HANDOFF.md carry the context forward.
- **Plan first.** Start each phase in plan mode (Shift+Tab) and approve the plan before any code is written.
- **Verify in the browser yourself.** Claude can't see the preview play. You are the check for "no blank frames at the cut".
- **Commit after each green phase.** If a phase goes sideways, `git reset` to the last commit instead of patching it.
- **Feed it errors verbatim.** Paste console errors and screenshots of glitches.
- If Claude starts guessing at Cavalry API calls, stop it and point it to `vendor/cavalry-web-player/` examples.

## 3. First prompt (paste into Claude Code)

> Read CLAUDE.md, docs/SPEC.md and docs/HANDOFF.md. We're starting Phase 0. Also skim vendor/cavalry-web-player/ (examples + serve.py) and summarize: how the runtime loads, how scene files and assets get into the virtual filesystem, how instances are freed, and any headers serve.py sets. Then propose a plan for Phase 0 that sets up headers/MIME for those runtime files from day one. Don't write code until I approve.

## 4. Next prompts
- Phase 1: "Phase 0 is committed and live on Vercel. Plan Phase 1 (Cavalry spike) per SPEC §8."
- Phase 2: "Plan Phase 2, the critical test. Hard-coded timeline only, no UI. Include unit tests for src/lib/timeline first."
- After Phase 2: report results in HANDOFF before touching UI.
