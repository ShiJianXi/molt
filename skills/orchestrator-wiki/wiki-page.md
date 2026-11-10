# Wiki page

The page this skill publishes. The plugin ships no HTML, so write the page from this spec when you
create a wiki (step 2) or upgrade one (step 3), as one self-contained `.html` file in the
scratchpad. Load the `artifact-design` skill first if it's available; this spec wins where the two
differ. The page is read-only: it only reads the database and never writes. It is for a person, so
it shows plain sentences and folds every technical value away.

## Data

Read with `const db = await window.claude.use("db")` and keep it live with `onSnapshot`:

- `db.collection("wiki")`: doc `meta`, doc `review`, and every doc whose id starts with
  `archive-` (the archive months).
- `db.collection("pages")`: `charter`, `decisions`, `state`, `gotchas`, in the shapes of the
  skill's step 3 table.
- `db.collection("log").orderBy("at", "desc").limit(30)`: `{at, text, kind?, topic?}`.

Treat every list as possibly missing (`[]`). Dates stored as `YYYY-MM-DD` are calendar days: build
them as local dates so they show on the right day in every timezone; full ISO times show with the
time ("Nov 3, 4:05 PM").

States: no `window.claude` or no db → "Only the wiki's owner can read it. Sign in to claude.ai as
the owner." A read error (the data is owner-only) → "Only the wiki's owner can read it." Nothing
after 10 seconds → "The wiki didn't load. Reopen the page to try again." No `meta` → "No wiki
yet".

**Old pages.** A page doc that has a `body` and neither `items` nor `summary` was written before
the structured format: show its `body` as preformatted text in a card, in place of that section's
normal view, until Claude rewrites it.

## Layout

Title `Orchestrator Wiki`. One column, max width about 860px, 16px side gutters, in this order:

1. **Header.** `h1` = `meta.domain` ("Orchestrator wiki" if empty). Under it `charter.summary` as
   a lede. A small line: "Kept by “<orchestrator>”", "Started <createdAt>", "Last updated
   <latest page updatedAt>", and one link per `meta.boards` entry (its `title`, else "Plan board
   N"), opening in a new tab.
2. **Needs you · N** (hidden when N is 0). A shaded amber box listing every `state` item's
   `needsYou`, then "Review proposal: <text>" for each `review.proposals` entry.
3. **Now.** A card per `state` item: title, `ref` in monospace, and a pill: "Waiting on you"
   (amber) when `needsYou` is set, else Running and In review (blue), Done (green), Paused (grey).
   `next` below. "Nothing running." when empty.
4. **Decisions.** Rows sorted newest `date` first: the date on the left ("Nov 3"), then `text`,
   `why` muted, and `ref`. Stacks on phones. "No decisions yet."
5. **Gotchas.** A card per item: "**<topic>:** <text>". "No gotchas yet."
6. **Charter.** Two boxes side by side, In scope and Out of scope, then full-width Rules, then
   Learned rules (only when there are any) with the line "Rules this orchestrator gave itself
   after a review. Say “forget that rule” to remove one." and each rule's `why` muted under it,
   then Skills as chips. An empty list shows "None."; no charter shows "No charter yet."
7. **Last review** (only when `wiki/review` exists). A card: "Reviewed <at>", then the `changed`
   sentences as a list, or "Nothing needed changing."
8. **Recent changes.** The log, newest first, time and text. "Nothing written yet."
9. **Archive** (only when there are archive months). One collapsed `<details>` per month, newest
   first (sort ids with numeric compare so `-2` parts order right), summary "<month> · N items".
   Each entry: a small uppercase tag from `from` (decisions → Decision, gotchas → Gotcha, state →
   Work, learned → Learned rule, log → Log), the item's `text`, `title` or `topic`, and `reason`
   muted under it.

**Technical details.** Wherever an item, the charter, a learned rule or the review has a `details`
object with keys, add a collapsed `<details>` "Technical details" holding a definition list of
key and value (values in monospace, wrapping anywhere). Never show `details` values anywhere else.

## Look

Font: Instrument Sans, IBM Plex Mono for refs and technical values (Google Fonts). Light and dark
tokens:

| token | light | dark |
|---|---|---|
| page / card | `#f4f5f7` / `#ffffff` | `#12151b` / `#1a1f27` |
| text / muted / line | `#1a2130` / `#5d6676` / `#dde1e8` | `#e7eaf0` / `#9aa3b2` / `#2c333f` |
| accent | `#2f5bd3` | `#8aa6f2` |
| needs you (text / fill) | `#9a5b00` / `#fdf1dc` | `#e8b45c` / `#3a2c12` |
| done | `#1f7a52` / `#e2f3ea` | `#63c99a` / `#173126` |
| running | `#2f5bd3` / `#e8eefc` | `#8aa6f2` / `#1e2a47` |
| paused, chips | `#636b7a` / `#eceef2` | `#a3abba` / `#262c37` |

Define them on `:root`, switch to dark under `prefers-color-scheme: dark` unless
`data-theme="light"`, and under `data-theme="dark"`. Section headings are small, uppercase and
muted; cards have a 1px line border and 10px corners.

## Must hold

- Build every element with `textContent` and `createElement`, never `innerHTML`: every field is
  free text.
- Only `meta.boards` URLs become links, with `target="_blank"` and `rel="noopener"`.
- Works at phone width with no page-wide horizontal scroll. Visible focus rings on summaries.
