# Plan board page

The page this skill publishes. The plugin ships no HTML, so write the page from this spec each time
you create a board (step 4), as one self-contained `.html` file in the scratchpad. Load the
`artifact-design` skill first if it's available; this spec wins where the two differ. The page is
read-only: it only reads the database and never writes.

## Data

Read the board's database with `const db = await window.claude.use("db")` and keep it live with
`onSnapshot`, so every `ArtifactData` write shows without a reload:

- `db.doc("plan/meta")`: `{goal, orchestrator, example?}`.
- `db.collection("tasks")`: one doc per task, id = doc id, fields as in the skill's step 3 table.
- `db.collection("log").orderBy("at", "desc").limit(30)`: `{at, text}`.

Derived, never stored:

- **Shown status.** A task whose `status` is missing or `todo` shows as **Ready** when every
  `blockedBy` id is `done` (or unknown), else **Waiting**. Any other status shows as stored.
- **Wave.** 0 for a task with no blockers, else 1 + the highest wave among its blockers. Guard
  against cycles (a task already on the visiting path counts as 0) so a bad graph can't hang the
  page. Shown to people as wave number + 1.
- **Unblocks.** The tasks that list this task in their `blockedBy`.

States: no `window.claude` or no db → "Sign in to claude.ai to see this plan." A read error →
"The plan couldn't be read. Reopen the page to try again." Nothing after 10 seconds → "The plan
didn't load. Reopen the page to try again." No `plan/meta` → "No plan yet". No tasks → an empty
box saying "No tasks yet. The orchestrator writes them here when it splits the work."

## Layout

Title `Plan Board`. One column, max width about 1080px, 16px side gutters, four parts:

1. **Header.** A small line with "Orchestrator: <orchestrator>" (plus " · example data" when
   `meta.example`) and "Updated <latest task updatedAt>". The goal as the `h1`. A thin progress
   bar split by shown-status counts, and a pill per status with its count ("3 done").
2. **Order and dependencies.** An SVG graph in a horizontally scrollable card: one column per
   wave, left to right, nodes sorted by `order` then id and centred vertically in their column.
   Each node is a rounded box (about 188 by 54) with a status-coloured border and left stripe,
   "<id> · <status>" on top and the title below (cut at about 26 characters with "…", full title in
   an SVG `<title>`). Curved arrows run from each blocker to its dependent: solid while the
   blocker isn't done, dashed and faint once it is. A legend of status pills sits under it.
3. **Tasks.** A list grouped under wave headings ("Wave 1 · can start now", "Wave 2 · after wave
   1"), sorted by wave, `order`, id. Each task is a `<details>` row: a `›` chevron that turns down when open, id, title, status pill, and a
   hint line ("Waiting on T1, T2" for open blockers, else "Agent: <owner>"). Expanded, it shows
   `details` as a paragraph, then a definition list of Done when, Wave, Blocked by and Unblocks
   (each linked task as id, title and its status pill; "Nothing, can start any time" and "Nothing"
   when empty), Approach, Evidence, Agent, Branch (monospace), Pull request and Last updated, and
   `note` in a shaded box. Skip empty fields.
4. **Activity.** The log, newest first, time and text per line; "Nothing has happened yet." when
   empty.

Clicking or pressing Enter or Space on a graph node opens that task's row and scrolls it into
view. Open rows stay open when the data updates.

## Look

Font: Instrument Sans for text, IBM Plex Mono for ids and times (Google Fonts). Status colours,
light and dark:

| status      | label       | light text / fill   | dark text / fill    |
|-------------|-------------|---------------------|---------------------|
| done        | Done        | `#1f8a5b` / `#e3f4ec` | `#5cc995` / `#173226` |
| in_progress | In progress | `#2f5bd3` / `#e5ecfc` | `#7d9cf0` / `#1c2a4a` |
| review      | In review   | `#7a4cc2` / `#efe8fa` | `#b393ea` / `#2b2140` |
| ready       | Ready       | `#9a6200` / `#fbf0d9` | `#e0b057` / `#3a2e14` |
| waiting     | Waiting     | `#6b7383` / `#eceef2` | `#a0a8b8` / `#252c38` |
| failed      | Failed      | `#c23a3a` / `#fbe5e5` | `#ec7b7b` / `#3d1d1d` |

Page `#f5f6f8` / `#12161d`, cards `#ffffff` / `#1a2029`, text `#1b2230` / `#e6e9ef`, muted
`#5c6678` / `#98a1b2`, lines `#d9dde5` / `#2c3442`, accent `#2f5bd3` / `#7d9cf0`. Define them as
tokens on `:root`, switch to dark under `prefers-color-scheme: dark` unless `data-theme="light"`,
and under `data-theme="dark"`; redraw the graph when either changes, since SVG colours are read
from the tokens. Status order in the bar, pills and legend: done, review, in progress, ready,
waiting, failed.

## Must hold

- Build every element with `textContent` and `createElement`, never `innerHTML`: task fields are
  free text.
- Link `pr` only when it starts with `https://github.com/`, with `target="_blank"` and
  `rel="noopener"`; otherwise show nothing.
- Works at phone width: rows stack, the graph scrolls inside its card, no page-wide horizontal
  scroll. Visible focus rings on rows and nodes. No motion when `prefers-reduced-motion`.
