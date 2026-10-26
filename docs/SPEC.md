You are building **cpp-shorts**: a local CLI that turns a topic into a finished vertical YouTube Short teaching one C++ concept, with a human review gate before anything is published. Work in PHASES. After each phase, stop, run the acceptance check, show me the result, and wait for my "go" before starting the next phase. Do not build everything at once.

## 0. Context and non-negotiables

- Output: 1080x1920, 30 fps mp4, target 45-90 s (hard cap 170 s), one C++ concept per video.
- Narration language is Vietnamese (configurable `NARRATION_LANG=vi`). Code and identifiers stay in English.
- Code in videos must be REAL: every snippet is compiled and executed with g++, and the output shown on screen is the real program output, never LLM-written.
- The mascot is a rigged SVG pufferfish already provided in `./mascot/`:
  - `pufferfish.svg`: static, layered. Layer ids: `float, breath, puff, tail, fin, spines, marks, eyes, lid, happy, brows, browL, mouth, m-smile, m-a, m-o, m-big, m-flat`.
  - `pufferfish-rig-reference.html`: working HTML/CSS reference showing exactly how it animates. Treat it as the spec. Port it to Remotion as a `<Mascot pose mouth />` React component. Do NOT redraw or restyle the fish; keep its colors, outline and proportions.
- Mascot behavior to reproduce: idle breathing + bobbing, tail wag, fin flap, random blink every 2-5 s, poses `idle | talk | wow | think | happy`. `wow` = body puffs to 1.15x with spines pushed out, eyes bigger, brows raised, short shake (used for bugs/crashes/segfaults). `talk` = mouth cycles through `a/o/smile/flat` driven by the narration audio loudness (precompute an amplitude envelope with ffmpeg or Python and pass it as props; no live audio analysis).
- Everything runs locally and must be free: Python 3.11+, Node 20+, Remotion, ffmpeg, g++, edge-tts. Gemini API only for text (script writing, fixing code). Never hard-code a model name; read `GEMINI_MODEL` from `.env`. Add `python main.py models` that lists models my API key can use.
- Secrets in `.env` (`GEMINI_API_KEY`, `GEMINI_MODEL`, `TTS_VOICE=vi-VN-HoaiMyNeural`, `NARRATION_LANG=vi`). Provide `.env.example`. Never commit `.env`.
- Every step caches its output in `./out/<slug>/` and is idempotent, so any stage can be re-run alone: `python main.py run <slug> --from tts`.

## 0.5 Host environment: Windows 11 (PowerShell)

- Use `python` (not `python3`), a venv at `.venv` (activate with `.venv\Scripts\Activate.ps1`), and `pathlib` for every path. No bash-only syntax in scripts or docs; give PowerShell commands.
- Always read/write text files with `encoding="utf-8"` and set `PYTHONUTF8=1` in the CLI entry so Vietnamese text is safe.
- g++ comes from MSYS2 UCRT64 (`C:\msys64\ucrt64\bin` on PATH). Compile through `subprocess` with an argument list (no shell strings), output `.exe` into a temp dir. There is no `ulimit` on Windows: enforce a 5 s timeout (kill the process tree), cap captured output (e.g. 64 KB) and run in a fresh temp dir.
- Open videos with `os.startfile`. `ffmpeg` and `ffprobe` are expected on PATH (Gyan build via winget).
- Remotion: bundle fonts locally in `remotion/public/fonts` (Lexend for subtitles and titles, JetBrains Mono for code; both must cover Vietnamese diacritics where needed). Do not depend on system fonts.
- `doctor` must detect missing tools and print the exact `winget` or `pacman` command to fix each.
- Git: line endings are handled by `.gitattributes`; never commit `.env`, `out/`, `ready/`, or media files.

## 1. Project layout

```
cpp-shorts/
  main.py                  # CLI entry (typer or argparse)
  curriculum.yaml          # ordered lessons (provided in the repo; you load it)
  state.json               # per-lesson status, see section 3
  pronunciations.yaml      # TTS text replacements (cout -> "xi-out", nullptr -> "nơ pơ-tờ"...)
  pipeline/
    script.py              # Gemini -> validated script JSON (pydantic)
    verify.py              # compile + run snippets, fix loop
    tts.py                 # edge-tts -> per-scene audio + durations + subtitle timing
    envelope.py            # audio -> mouth amplitude envelope
    checks.py              # automatic quality gate
    render.py              # calls Remotion CLI
    meta.py                # title, description, tags, chapters
  prompts/script_system.md # I provide it; load as-is
  remotion/
    src/Root.tsx, Short.tsx
    src/components/{Mascot,CodeTyper,Terminal,MemoryBoxes,Subtitles,TitleBar,SafeZones}.tsx
    public/mascot/pufferfish.svg
  out/<slug>/              # script.json, audio/, frames cache, draft.mp4, thumb.png, review.md, meta.json
  ready/<slug>/            # approved: final.mp4, thumb.png, meta.json
```

## 2. Pipeline stages

1. **script**: read the lesson from curriculum.yaml, call Gemini with `prompts/script_system.md`, request JSON only, validate with pydantic, retry up to 3 times on schema errors. Script schema:
   `{title, hook, scenes:[{id, narration, tts_text?, code?, code_mode: "append"|"replace", highlight_lines?, expected_output?, visual?, mascot_pose, mascot_note?}], outro, tags[]}`.
   Rules enforced in code, not just in the prompt: hook <= 3 s of speech, 4-6 scenes, total code <= 25 lines, <= 14 words per subtitle chunk.
2. **verify**: concatenate the scene snippets into compilable programs, compile with `g++ -std=c++20 -Wall -Wextra`, run in a temp dir with a 5 s timeout and no network. Replace `expected_output` with the REAL stdout. On compile error or timeout, send the error + code back to Gemini to fix (max 3 retries); if still failing, mark status `failed` and stop with a clear message. Scenes that intentionally show a bug (UB, segfault) must be flagged `intentional_error: true` and are shown as text with a `wow` pose, never executed for output.
3. **tts**: `edge-tts` per scene using `tts_text` if present (else narration) after applying `pronunciations.yaml`. Save mp3, exact duration (ffprobe), and subtitle timing: use edge-tts boundary events if available for the voice, otherwise distribute words across the scene duration proportionally by character count. Add 250 ms padding between scenes. Normalize loudness to about -14 LUFS with ffmpeg loudnorm.
4. **envelope**: for each scene audio, compute a 30 fps amplitude array (0-1) saved as JSON for the mascot mouth.
5. **render**: Remotion composition `Short` takes the script JSON, audio files, envelopes. Scene length = audio duration + padding. Components:
   - `CodeTyper`: syntax-highlighted typing animation (keywords, types, numbers, comments), current line highlight, scene-driven reveal speed; dark window styled like `pufferfish-rig-reference.html`'s page palette (plum/rose/cream, amber accent).
   - `Terminal`: shows real output as a footer strip inside the code window. `Visual`: animated mini-illustration of what the code does, from a fixed template library (section 2.5).
   - `Subtitles`: word-level highlight, large, bold, readable. Default style is a speech bubble coming from the mascot (section 2.5); keep a `SUBTITLE_STYLE=bubble|bar` setting.
   - `Mascot`: lives in the underwater scene at the bottom (section 2.5), pose per scene; the hook scene may show it bigger.
   - `SafeZones`: respect YouTube Shorts UI. Everything lives inside the content frame defined in 2.5. Add a debug flag `--debug-safe` that overlays the zones.
   - Add a 2-second outro card (mascot happy + "Bài tiếp theo: <next topic>").
   Export `draft.mp4`, plus `thumb.png` (mascot + big topic title, 1080x1920 and a 1280x720 crop).
6. **checks** (automatic gate, runs before review; any hard fail blocks review):
   - hard: duration 20-170 s, all snippets verified, no text overflow outside safe zones (check via a Remotion still at each scene midpoint and a simple bounding-box test of the text elements), audio not silent, loudness in range.
   - soft (warnings written into review.md): hook longer than 3 s, narration speed above 3.2 words/s, code line over 40 chars, scene longer than 18 s.
7. **meta**: title (<= 60 chars, topic + hook), description (summary, the final code in a block, chapter list, #Shorts #cpp #lậptrình), tags. Written to `meta.json`.

## 2.5 Layout (from my hand sketch; follow it exactly)

Canvas 1080x1920. All content lives inside a **content frame**: x 36-936, y 130-1530. The right ~144 px (YouTube action buttons), the bottom ~390 px (channel/caption UI) and the top ~130 px (Shorts header) stay empty of text and mascot.

```
y130  +-------------------------------+
      | Học C++ · Ngày {n}            |  title bar: lesson number + {title}
y280  +-------------------------------+
      |  CODE WINDOW (typing)         |  syntax highlight, current line highlighted,
      |  ...                          |  max ~14 lines
      |  $ output (terminal strip)    |  real program output, appears when relevant
y800  +-------------------------------+
      |  VISUAL (minh họa code =      |  animated mini-illustration of what the
      |  animation)                   |  code does (templates below)
y1140 +-------------------------------+
      |  ~ ~ ~ underwater scene ~ ~ ~ |  waves at top, bubbles rising, seaweed swaying,
      |  (bubble)  <  PUFFERFISH      |  sand + shells at bottom. Fish floats here.
      |  o #channel                   |  speech bubble = subtitles, left of the fish
y1530 +-------------------------------+
```
(Interpretation of my sketch: the blob left of the fish is the fish's speech bubble used for subtitles. `#channel` handle and a small round avatar sit at the bottom-left of the underwater scene. Make the handle text and avatar configurable in `.env`: `CHANNEL_HANDLE`, `CHANNEL_AVATAR`.)

Underwater scene: deep teal/navy gradient that contrasts with the rose/cream fish, subtle light rays, 8-12 bubbles on loop, 2-3 seaweed strands swaying, gentle wave line at the top edge. Keep it light so rendering stays fast (SVG + CSS-like interpolation, no heavy filters).

**Visual template library** (the script picks one per scene via `visual: {type, params, steps}`; steps are timed highlights as a fraction of the scene, e.g. `{t: 0.4, do: "highlight", target: "p"}`). Implement these first, each as a Remotion component with a pydantic schema:
1. `memory`: named boxes with values and addresses, arrows for pointers/references, highlight on read/write.
2. `array`: indexed cells, moving index marker, out-of-bounds flash (uses the fish `wow`).
3. `loop`: counter variable + iteration trail.
4. `callstack`: frames pushed/popped with local variables.
5. `branch`: if/else flow where the taken path lights up.
6. `text`: big label or comparison (fallback, e.g. "int = 4 bytes"). If a `visual` fails validation, fall back to `text` instead of failing the render.
Visual animations must be deterministic and derived from the real verified program (e.g. real values), not invented by the LLM.

## 3. State machine and the REVIEW GATE (important)

Statuses per lesson in `state.json`:
`planned -> scripted -> verified -> voiced -> rendered -> awaiting_review -> approved -> published`, plus `failed` and `rejected`.

The automatic part stops at `awaiting_review`. A human (me) must approve.

Commands:
```
python main.py make "Con trỏ là gì"          # one-off topic (not in curriculum): runs stages 1-7 -> awaiting_review
python main.py next                           # next unfinished lesson in curriculum.yaml -> awaiting_review
python main.py batch 7                        # next 7 lessons, sequential, each stops at awaiting_review
python main.py review <slug>                  # opens draft.mp4 and prints review.md checklist; optional: `npm run studio` to tweak in Remotion Studio
python main.py approve <slug>                 # copies final.mp4, thumb.png, meta.json to ready/<slug>/, status approved
python main.py reject <slug> --note "..." [--from script|tts|render]
                                              # saves my note, re-runs from the chosen stage; if --from script, the note is appended to the Gemini prompt as editor feedback
python main.py status                         # table of all lessons and statuses
python main.py models                         # list available Gemini models
```
`review.md` is generated per video with a checklist for me:
- [ ] Explanation is technically correct (list each claim made in narration, one per line)
- [ ] Code and real output shown match what the narration says
- [ ] Pronunciation of technical terms sounds right (list the terms detected)
- [ ] Mascot poses fit the moment, nothing covered by YouTube UI
- [ ] Title/description OK
Include the soft-check warnings and the verified code at the end.

No upload code in the first versions. `approved` videos are uploaded by me manually or scheduled in YouTube Studio. Only after I ask, add an optional `upload.py` using YouTube Data API v3 (OAuth), and warn me that an unaudited API project forces uploads to private.

## 4. Phases (stop after each one)

**Phase 1: skeleton + proof of mascot.** Project layout, `.env`, `models` command, Remotion project with the `Mascot` component ported faithfully. Acceptance: `npm run studio` shows the pufferfish with all 5 poses animated, and a 10 s test mp4 `out/_test/mascot.mp4` renders. Compare it side by side with `pufferfish-rig-reference.html` and tell me any visual differences.

**Phase 2: script + verify + tts.** Lesson 1 only. Acceptance: `script.json` valid, snippets compile, real output captured, audio files and timings exist. Print the script in a readable form.

**Phase 3: full render.** CodeTyper, Terminal, Subtitles, SafeZones, outro, thumbnail, checks, meta. Acceptance: `python main.py make "Con trỏ là gì"` produces a draft.mp4 that I can watch, a review.md, and status `awaiting_review`.

**Phase 4: review gate + curriculum.** `approve`, `reject`, `status`, `next`, `batch`, `curriculum.yaml` loading, state.json, resume after crash. Acceptance: run `batch 3`, approve one, reject one with a note and watch it regenerate.

**Phase 5 (only if I ask):** scheduling helper (cron/systemd timer or a simple `schedule.md` for manual posting), optional upload.py, optional hook/outro clips from an AI video generator placed manually in `assets/clips/` (the pipeline must work fine without them).

## 5. Engineering rules

- Pin all dependency versions (Remotion exact version, `edge-tts`, `google-genai`, pydantic). Do not use any npm/pip package published in the last 14 days.
- Remotion rendering uses `--concurrency` set from CPU count; show progress.
- Clear, short error messages with the next action ("Fix X, then run: python main.py run <slug> --from verify").
- Write install steps for Windows 11 into the repo README, and a smoke test (`python main.py doctor`) that checks g++, node, ffmpeg, ffprobe, edge-tts, API key, and the mascot file.
- Keep code small and readable; no premature abstractions. Add tests only for: schema validation, verify loop, state transitions.
- If something in this spec is ambiguous or impossible, ask me ONE concise question or state your assumption and continue.

