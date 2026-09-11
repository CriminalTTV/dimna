# دمنة — Script Marker

Context for Claude Code. Read this before touching anything.

## What this is

A single-page web tool for planning visuals on Arabic-language documentary
videos. The user (Hussain, Kuwait) writes and narrates Arabic historical
documentaries, then needs to mark where AI-generated scenes, map animations,
motion graphics and archive footage go — and hand that plan to animators who
don't read Arabic.

Live at **dimna.org** (Netlify, auto-deploys from this repo's main branch).

## Current state

Everything lives in `index.html`. One file. React 18 + Babel standalone,
both from CDN, no build step. This was deliberate — it had to run by
double-clicking a file before it had a domain.

**This is now the main thing to fix.** See "First job" below.

### What works

- Script import: SRT, WebVTT, `start / end / text` triplets, `[0:00 - 0:04] text`
  inline. Symbols: `#` / `##` headings, `"quoted"` headings, `>` notes,
  `//` ignored. See SCRIPT-FORMAT.txt.
- Gap filling on import: stretches each line's end to the next line's start,
  preserving the real end in `origEnd` so silences stay visible on the waveform.
- Waveform: decoded at 8 kHz via OfflineAudioContext to keep memory sane on
  27-minute files. Zoom 1–120×, drag to select a region, click to seek.
  Silence detection shades low-RMS runs.
- Tagging: select lines (shift/cmd-click) or drag a waveform region, then press
  1–9. Tags are user-editable (label, colour, order) and stored per project.
- Shot list: tags sharing a span group into expandable folders. Each tag has a
  brief, unlimited comments, status (todo/doing/done) and a due date.
- Annotations: select text inside an expanded tag block → highlight, underline,
  strikethrough, or highlight-with-note. Stored as character offsets per cue.
- Board view (kanban by status), Calendar view (aggregates due dates across
  every project in the browser).
- Tag list import/export: `tag | start | end | status | due | note`.
  See TAGS-FORMAT.txt.
- Undo/redo, 40 steps.
- Resizable split pane, waveform height, script font size.
- Cloud: Supabase auth (email/password), project sync, audio storage.
  Audio for projects untouched 14 days is deleted on sync.

### Data model

```
project = {
  id, name, updated, fps, offset, audioName,
  startDate, dueDate,
  cues:    [{ id, kind:'line'|'head'|'note', level, text, start, end, origEnd }],
  markers: [{ id, tagId, cueIds:[], start, end, note,
              comments:[{id,text,at}],
              annotations:[{id,cueId,s,e,style,color,note}],
              status, due }],
  tags:    [{ id, label, color }],
  ui:      { scriptSize, waveH, splitPct },
  cloudAudio, cloudAudioName
}
```

Markers anchor to `cueIds` (script lines), to raw `start`/`end` (waveform
regions, tag-list imports, playhead tags), or to **both**. Region tags made since
FIXES item 1 carry both: `cueIds` lists the lines under the region (chips, script
text in the block) and `start`/`end` is the span that was dragged. `range(marker)`
uses `start`/`end` when both are numbers, otherwise derives the span from the
lines. `cuesInRange()` decides which lines are "under" a region. Don't break that.

### Storage

- Local: `localStorage` key `dimna:projects` — a map of id → project.
  `dimna:last` holds the last opened id. `dimna:cloud` holds the Supabase
  URL and anon key the user pasted in.
- Cloud: Supabase table `public.projects` (see supabase-setup.sql), private
  storage bucket `audio`, files at `<user_id>/<project_id>.<ext>`.
- RLS is on. Every policy is `auth.uid() = user_id`. Storage policies check
  `(storage.foldername(name))[1] = auth.uid()::text`.

**No secrets are in this repo.** The Supabase URL and anon key are entered by
the user in the app and live in their browser's localStorage. The service_role
key has never been in any file here and must never be. If you add a build step,
do not introduce a `.env` that gets committed.

## First job

Split `index.html` into a real project. It's past 100 KB in one file and Babel
standalone compiles it in the browser on every page load, which is slow on
mobile.

Suggested: Vite + React, TypeScript if you want it. Split roughly along the
existing component boundaries — App, Waveform, ScriptPane, ShotList,
GroupCard, MarkerBlock, AnnotatableLines, Board, Calendar, the modals — plus
`lib/` for parseScript, parseTagList, time helpers, storage, and the Supabase
client.

Netlify needs its build command and publish directory set after this. The site
currently has no build step at all.

**Verify nothing regresses**, specifically:
- Arabic RTL rendering per line (direction is detected per cue via
  `/[\u0600-\u06FF]/`, not set globally)
- Annotation character offsets surviving the refactor
- The 8 kHz OfflineAudioContext decode — a naive `new AudioContext()` will
  blow up memory on a 27-minute file
- Local dates: `localISO()` exists because `toISOString()` shifts the day
  backwards in UTC+3. Don't replace it with `toISOString().slice(0,10)`.

## Known rough edges

- The waveform redraw effect depends on `time`, so it recreates its
  ResizeObserver several times a second. Works, but wasteful.
- No virtualisation on the script list. ~520 cues is fine; 5,000 would not be.
- Mobile layout stacks at 860px but is cramped. Worth real attention — the
  user edits on his phone.
- Freehand drawing on the script was requested and deliberately not built:
  strokes can't stay anchored to reflowing RTL text. Highlight/underline/strike
  cover the need. If revisiting, it belongs on video frames, not text.
- No tests of any kind.

## Wanted next

- Teams. Auth already supports unlimited users. Needs a `project_members`
  table and the four RLS policies changed from `user_id = auth.uid()` to a
  membership lookup.
- Better mobile.
- Eventually: word-level gloss translation (Arabic phrase → English meaning,
  aligned to timecode) so non-Arabic-speaking editors can work from the script
  directly. This was the original idea the tool grew out of. Forced alignment
  via WhisperX gives word-level timestamps, but Arabic needs a wav2vec2 model
  sourced separately — it isn't in WhisperX's defaults.

## How the user works

Direct answers, concrete numbers, no motivational filler. He'd rather hear
that something is a bad idea than get agreement. Explain unfamiliar tooling
plainly — he's self-taught and moves fast, but hasn't done a build pipeline
before. Arabic and RTL correctness is not a nice-to-have; it's the product.
