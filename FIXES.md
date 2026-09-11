# Fixes and changes

Working list for the دمنة script marker.

**Claude Code: do NOT work through this file top to bottom.**
Only touch an item when I ask for it by number. Use plan mode first.
When one is done and I've confirmed it works in the browser, move it
to Done at the bottom with the date.

Item 11 is not optional and is not last in priority — read it before
you change any data structure in any other item.

---

## Now

### 1. Waveform region tagging doesn't cover all the lines in the range

**What happens now:** I drag a region on the waveform and tag it. The tag
lands on one line only — the last one at the end of the region — instead
of every line inside the range.

**What should happen:** A waveform region tag attaches to every cue whose
time range overlaps the selected region. If the region covers 1:00–3:24
and eight lines fall in that window, all eight get the tag chip, and the
shot list block shows all eight lines of script.

**Where:** `addMarker()` in index.html. Currently a region marker is created
with `cueIds:[]` and raw `start`/`end`, so it never links to script lines at
all. It needs to resolve overlapping cues and populate `cueIds`, while still
keeping the raw `start`/`end` for the part of the region that falls in
silence with no line under it.

**Careful:** don't break tags placed on raw regions where there genuinely is
no script line (the silence-filling case). Both must keep working.

---

### 2. iPhone is unusable — page loads zoomed in

**What happens now:** On iPhone the site opens zoomed in and I can't work.

**What should happen:** Usable on a phone. It's a real use case — I plan
shots on my phone away from the desk.

**Where to look:**
- The panes have `minWidth: 300` / `min-width: 280px` which forces
  horizontal scrolling on a ~390px screen.
- `100vh` on iOS Safari doesn't account for the address bar. Use `100dvh`.
- The header has ~10 buttons in a row that can't fit.
- The transport bar has the same problem.
- Tap targets are sized for a mouse.

Don't just shrink everything. On a phone the sensible layout is one pane at
a time with a switcher, a collapsed header behind a menu, and a shorter
waveform. Show me a plan before building it.

---

### 3. Comments and the brief are wrong

**3a — Enter should save, not make a new line.**
When I type a comment and press Enter it should save that comment, close the
editor, and that's it. If I want another comment I press "add comment"
deliberately. Right now pressing Enter leaves me in a new empty line.

**3b — Remove the separate "brief" button.**
There shouldn't be a brief field and a comments list as two different things.
The **first comment is the brief.** Show it that way in the UI — different
background, a small "BRIEF" label, slightly larger text, whatever reads as
"this is the main instruction" and the rest are follow-up notes.

Migrate existing data: any marker with an old `note` value becomes its first
comment. Don't drop it.

**3c — Annotation should work like Adobe Acrobat. Copy it directly.**

This is the part you got wrong last time. The flow I want:

1. Each tag block has an **expand button**. Pressing it makes the block grow
   and visibly highlight, so it's obvious it's in "annotation mode."
2. Only in that expanded state do the annotation tools appear — highlight
   colours, underline, strikethrough.
3. I select text and apply a tool. The highlight appears.
4. **I then click on the existing highlight** and a small popup appears where
   I add a note. I should NOT have to re-select the same text to attach a
   note — that's what you built and it's wrong.
5. Hovering a highlight shows its note.
6. Clicking a highlight and pressing Delete removes that highlight.

Acrobat's comment behaviour is the reference. Match it.

---

### 4. Dropdowns look cheap — white background, unreadable

**What happens now:** The tag dropdown, the status dropdown ("To do"), and
the date field (`mm/dd/yyyy`) all open with a white background against the
dark theme. It looks broken.

**Why:** native `<select>` option lists and `<input type="date">` pickers
can't be reliably styled by CSS across browsers.

**What should happen:** Replace them with custom dropdown and date-picker
components styled to the theme. Colour swatch next to each tag name in the
tag dropdown, status colour dot in the status dropdown.

---

### 5. Layout should be right-aligned — the content is Arabic

**What happens now:** Timecodes sit on the left of each script line, tag
numbers on the left of each shot list block. Everything reads left-first
while the script itself is Arabic and reads right-first.

**What should happen:** Timecode on the right of the line. Shot list number
on the right. Generally, when the script is Arabic the whole interface
should follow RTL, not just the text inside each line.

Use CSS logical properties (`margin-inline-start`, `border-inline-start`,
`dir="rtl"` on containers) rather than hardcoded left/right, so this stays
correct if I ever load an English script.

---

### 6. Waveform is blocky when zoomed in

**What happens now:** Zoomed out it looks fine. Zoomed in it turns into fat
blocks instead of a real waveform.

**Why:** `analyse()` computes a fixed 4,000-bucket peak array for the entire
file. At 27 minutes that's one bucket per ~0.4 seconds. Zoomed to 100×, each
screen pixel is drawing from a fraction of a bucket, so it stair-steps.

**What should happen:** Sharp at every zoom level, and still light on the
machine when zoomed out.

**Approach:** keep the decoded 8 kHz sample data in memory after
`analyse()` (about 52 MB for a 27-minute file — acceptable), and compute the
peaks for the *visible window* each time the view changes, at roughly one
bucket per screen pixel. Zoomed out you're downsampling hard and it's cheap;
zoomed in you're reading real samples and it's sharp.

Don't re-decode the file on every zoom — decode once, slice many times.

---

### 7. No way to remove one line from a multi-line tag

**What happens now:** I select six lines and tag them. If I then want that
tag to cover only five, I have to delete the tag and redo it.

**What should happen:** In the expanded tag block, each script line has a
small remove control that detaches just that line from this tag. The tag's
time range recalculates from the lines that remain.

Also useful the other way: a way to add a line to an existing tag. Lower
priority than removal but same mechanism.

---

### 8. Selecting multiple lines selects the text instead

**What happens now:** Shift-clicking across lines drags a browser text
selection, which is ugly and gets in the way.

**What should happen:** Clicking and shift-clicking script lines selects
*lines*, never text.

**Note the exception:** text selection must still work inside an expanded
tag block, because that's how annotation works (item 3c). So this is
`user-select: none` on the script pane cues specifically, re-enabled inside
the annotation area — not a blanket rule.

---

### 9. Loading a second version of the script must keep my tags

**The situation:** I'm working from a script broken into ~7 words per line.
I want to load a different version of the same narration broken into ~3
words per line. Same project, same audio. All my existing tags must survive.

**Why this is hard:** markers link to cues by `cueIds`. Replacing the script
creates all-new cue IDs, so every link breaks.

**What should happen:** Importing a new script version into a project that
already has tags offers to **re-anchor by timecode** instead of wiping.
For each existing marker, take its current time range, find the cues in the
new script that overlap that range, and link to those instead.

Show me a preview before applying: how many markers re-anchored cleanly, how
many were ambiguous, how many couldn't be matched. Let me cancel.

Ideally the project can hold more than one script version and I can switch
between them, with the tags following. If that's a much bigger job, tell me
and we do the re-anchor version first.

---

### 10. Shot list layout at wide widths, and more view modes

**10a — Cap the block width.**
When I drag the shot list pane wide, the tag blocks stretch with it and look
bad. Give them a max width. Past that point, move the comments to sit to the
*side* of the script text rather than underneath it — two columns instead of
one tall block.

**10b — More view modes for the shot list.**
Right now there's one density. I want a switcher, something like:

- **Compact** — tags side by side, minimal height, comments hidden. Click a
  tag to expand it and see everything.
- **Comfortable** — roughly what exists now.
- **Expanded** — everything open, all comments and annotations visible.

The mode should stick per project.

---

### 11. Do not destroy my existing data — READ THIS BEFORE ANY OTHER ITEM

**The situation:** I am using the live site right now and adding real tags
and comments. When these fixes ship, everything I've already entered must
still be there, upgraded to the new structure. Not wiped.

**What this means for you:**

- Add a `schemaVersion` field to the project object.
- Write a migration function that upgrades an old project to the new shape
  on load, and run it on everything in `localStorage` and everything pulled
  from Supabase.
- Specifically for item 3b: an old marker's `note` becomes its first comment.
- Specifically for item 9: existing `cueIds` must keep working.
- Never delete a field you don't recognise — carry it through.

**Before shipping any change that touches the data model:**

1. Tell me to export every project to .json first, and wait for me to confirm.
2. Test the migration on a real exported file, not a made-up one.
3. Make it possible to load an old export into the new version.

If you are unsure whether a change is destructive, assume it is and ask me.

---

## Later

Not urgent. Don't start these without asking.

- Teams. Auth already supports multiple users; needs a `project_members`
  table and the RLS policies changed from `user_id = auth.uid()` to a
  membership lookup.
- Word-level gloss translation — Arabic phrase to English meaning, aligned
  to timecode, so non-Arabic-speaking editors can cut from the script
  directly. This is the idea the whole tool grew out of. See CLAUDE.md.
- Splitting index.html into a Vite project. Do this as its own session with
  nothing else in flight, and commit immediately before starting.

---

## Done

Move finished items here with the date. Don't delete them — if something
breaks later this is how we find what caused it.

-
