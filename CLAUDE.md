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

Everything lives in `index.html`. One file. React 18.3.1 + Babel standalone
7.26.0, pinned and loaded from jsDelivr, no build step. This was deliberate —
it had to run by double-clicking a file before it had a domain.

Splitting it into a build is planned for later (see "Later: split into a Vite
project" below and FIXES.md → Later).

### What works

- Script import: SRT, WebVTT, `start / end / text` triplets, `[0:00 - 0:04] text`
  inline. Symbols: `#` / `##` headings, `"quoted"` headings, `>` notes,
  `//` ignored. See SCRIPT-FORMAT.txt.
- Gap filling on import: stretches each line's end to the next line's start,
  preserving the real end in `origEnd` so silences stay visible on the waveform.
- Waveform: decoded once at 8 kHz via OfflineAudioContext (the samples stay
  in memory, ~52 MB for 27 minutes) plus min/max summaries per 16/256/4096
  samples. `columns()` builds one min/max pair per screen pixel for the
  visible window only when the view changes, so it's sharp at every zoom
  (1–120×). Drag to select a region, click to seek. Silence detection shades
  low-RMS runs (50 ms windows).
- Tagging: select lines (shift/cmd-click) or drag a waveform region, then press
  1–9. Tags are user-editable (label, colour, order) and stored per project.
- Shot list: tags sharing the same set of lines group into folders. Each tag
  has status (todo/doing/done), a due date and comments; the comment flagged
  `brief` is the main instruction and always shows first. Comments: Enter
  saves, shift+enter new line, esc cancels, double-click edits.
- View modes (per project, `ui.shotMode`): Compact tiles / Comfortable /
  Expanded. Blocks cap at 720px; past ~1000px of shot list (CSS container
  query on `.shots`) lines and comments sit in two columns.
- Annotations (Acrobat-style): "Annotate" on a tag block enters annotation
  mode; select text → toolbar; click a highlight → note / colour / style
  popup; Delete removes it; hover shows the note. Highlights belong to one
  tag. Stored as character offsets per cue.
- In annotation mode each line has an ✕ to detach it from that tag; "＋ N
  lines" attaches the lines selected in the script (`relink()`).
- Re-anchor (`planReanchor()`): "Replace script…" on a project with tags shows
  a preview (clean / ambiguous / no line left / time-only) and moves tags and
  highlights onto the new lines instead of wiping them.
- Interface direction follows the script (`uiDir`): RTL when most lines are
  Arabic. Timecodes, the waveform and the Calendar grid stay LTR.
- Themed `Dropdown` / `DatePicker` / `MenuButton` components (portalled
  popovers) replace every native select and date input.
- Board view (kanban by status), Calendar view (aggregates due dates across
  every project in the browser).
- Tag list import/export: `tag | start | end | status | due | note`.
  See TAGS-FORMAT.txt.
- Undo/redo, 40 steps (header buttons, ⌘Z, and an Undo button on the toast
  after any deletion). Adding a tag is a `patch`, not a `commit`, so it isn't
  undoable.
- Import / Export menus: brief (.md), Premiere/Resolve markers (.csv), tag
  list (.txt), project backup (.json). Tag and status filter chips, ▶ plays one
  shot and stops at its end, the shot under the playhead is marked live,
  `,` / `.` jump between shots, `?` opens help.
- Phone (below 760px, `isPhone`): one pane at a time with a switcher, ☰ menu,
  Select mode for multi-line selection, tag buttons in a scrolling row, More
  for the rest of the transport, 64px waveform, full-screen modals. Pointer
  events drive every drag, fields are 16px on touch (stops iOS zoom), 100dvh.
- Resizable split pane, waveform height, script font size.
- Cloud: Supabase auth (email/password). `syncCloud()` is two-way — it uploads
  every local project the cloud lacks or holds older, pulls anything newer, and
  runs on sign-in, on a reload with a session, and from Sync now. The Cloud
  button is a menu (account, Sync now, settings, Sign out). Audio uploads on
  load and comes back automatically when a project opens (both switchable);
  audio for projects untouched 14 days is deleted on sync, measured from the
  project's own last edit.
- Home (dashboard): projects with progress, what's due and in progress, an
  activity log, and settings (cloud, default fps for new projects, export all
  as a bundle, import, delete local data). The logo opens it.
- Every tag exports on its own (`exportTagText`): timecode, duration, status,
  brief, script lines, highlights with notes, comments — clipboard or .txt.
- Each shot shows how long it runs (`durText`) on folders, tags, tiles, Board
  cards and in the brief export.

### Data model

```
project = {
  id, name, updated, schemaVersion:3, fps, offset, audioName,
  activity:[{id,at,text}],        // newest first, capped at 60 (Home → Activity)
  startDate, dueDate,
  cues:    [{ id, kind:'line'|'head'|'note', level, text, start, end, origEnd }],
  markers: [{ id, tagId, cueIds:[], start, end,
              comments:[{id,text,at,brief?,edited?}],
              annotations:[{id,cueId,s,e,style,color,note, group?, orphan?,text?}],
              status, due,
              note }],               // legacy, always "" after migration
  tags:    [{ id, label, color }],
  ui:      { scriptSize, waveH, splitPct, shotMode },
  cloudAudio, cloudAudioName
}
```

**Migration.** Every project entering the app — localStorage (`readAll`),
Supabase pulls, .json import and drag-drop — goes through `migrateProject()`.
It must stay safe to run repeatedly: each marker is checked on its own
(an old tab that hasn't reloaded can keep writing old-shape data). v2 moved
`marker.note` into `comments` as the first comment flagged `brief:true`;
v3 added `project.activity`.

A highlight that crosses lines is stored as one annotation per line sharing a
`group` id; note, colour, style and delete apply to the whole group, and
`highlightList()` puts the parts back together for exports.
Never drop fields you don't recognise. Bump `SCHEMA` and extend
`migrateMarker` / `migrateProject` for any future shape change.

An annotation flagged `orphan` lost its line during a re-anchor; `text` keeps
the phrase so a later re-anchor can place it again. Orphans don't render.

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

**No secrets are in this repo.** The Supabase URL and the *publishable* (anon)
key are hardcoded as `DEFAULT_CLOUD` in index.html — that key is public by
design and safe only because RLS is on (each user can only touch their own
rows and audio folder). Users can override it in the app; overrides live in
localStorage. The service_role key has never been in any file here and must
never be. If you add a build step,
do not introduce a `.env` that gets committed.

## Later: split into a Vite project

Not started. Do it as its own session with nothing else in flight, and commit
immediately before starting (FIXES.md → Later).

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

- No virtualisation on the script list. ~520 cues is fine; 5,000 would not be.
- The phone layout has only been checked in an emulated 375×812 viewport,
  never on a real iPhone.
- A project holds one script version at a time; loading another re-anchors
  the tags onto it rather than keeping both.
- The browser pane used for testing reports 0 width while the desktop app's
  window is hidden; emulate a viewport size before running layout or drag tests.
- Freehand drawing on the script was requested and deliberately not built:
  strokes can't stay anchored to reflowing RTL text. Highlight/underline/strike
  cover the need. If revisiting, it belongs on video frames, not text.
- No tests of any kind.

## Wanted next

- Teams. Auth already supports unlimited users. Needs a `project_members`
  table and the four RLS policies changed from `user_id = auth.uid()` to a
  membership lookup.
- Several script versions in one project, switchable, with tags following.
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
