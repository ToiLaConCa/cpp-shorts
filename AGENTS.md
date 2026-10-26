# AGENTS.md (read this first)

Project: **cpp-shorts**, a local CLI that turns a C++ topic into a finished vertical YouTube Short (pufferfish mascot, real verified code, human review gate before publishing).

## Read order
1. `docs/SPEC.md` is the authoritative spec. Follow it exactly.
2. `docs/PROGRESS.md` shows which phase we are on.
3. `mascot/pufferfish.svg` and `mascot/pufferfish-rig-reference.html` define the mascot. Do not redraw or restyle it.

## Working rules
- The owner is on **Windows 11 (PowerShell)**. See section 0.5 of the spec.
- Work **one phase at a time**. After each phase: run its acceptance check, summarize results and visual differences, update `docs/PROGRESS.md`, commit locally with a clear message, then STOP and wait for the owner to say "go". Never start the next phase on your own.
- Never read, print, or commit `.env` or any API key. If a key is missing, tell the owner what to put in `.env`.
- Do not `git push` unless asked.
- Pin dependency versions; avoid packages published in the last 14 days.
- If the spec is ambiguous, ask ONE concise question or state your assumption and continue.
- Keep the code small and readable.

## First action
Check the toolchain (`node -v`, `python --version`, `ffmpeg -version`, `ffprobe -version`, `g++ --version`, `git --version`), report anything missing with the exact install command, then begin **Phase 1**.
