# Sasha & Aman's Wedding Site — Project Status

**Live site:** https://aman1051-coder.github.io/sasha-aman-wedding/
**GitHub repo:** https://github.com/aman1051-coder/sasha-aman-wedding
**Working file:** `Sasha_Aman_Wedding_v8.html` (edit this one — see "How to make changes" below)
**Wedding:** 4th & 5th December 2026, Bogmallo Beach Resort, Goa

---

## How to make further changes

1. This is a **single self-contained HTML file** — all CSS and JS are inline, all photos/the
   monogram are embedded as base64 data URIs directly in the HTML. No build step, no
   dependencies.
2. **Edit `Sasha_Aman_Wedding_v8.html`**, not `index.html` directly — `index.html` is just a
   copy of it (GitHub Pages requires that exact filename to serve as the site root).
3. **To deploy a change:**
   ```
   copy Sasha_Aman_Wedding_v8.html index.html
   git add index.html
   git commit -m "describe the change"
   git push origin main
   ```
   `git push` will pop up a browser window asking to log into GitHub — that's normal, it's the
   credential manager, not a bug. GitHub Pages rebuilds automatically, usually within a minute
   or two.
4. **Two other files are deployed alongside it and must stay in the repo:**
   `og-image.jpg` (link-preview image) and `wedding-music.mp3` (background track).
5. **`.gitignore` is deliberately strict** — only `index.html`, `og-image.jpg`,
   `wedding-music.mp3`, and `.md` docs are tracked. This folder has a lot of personal photos and
   source files; don't `git add -A` or `git add .`, add files by name.

---

## What's built

**Pages:** Home (hero + countdown + section cards), Our Story, Itinerary, Wardrobe, Venue,
Gallery, RSVP — single-page app style, switched via `go(id, navIndex)` in JS, no page reloads.

**Opening animation:** ink-bloom loader — three soft blooms bleed in around the monogram, which
sits in its own rotating rings; the monogram then morphs (measured live via `getBoundingClientRect`)
into its resting spot in the hero section, so the loader becomes the hero rather than just
disappearing. Plays in full on every visit by design (no skip, no shortcut). Respects
`prefers-reduced-motion` with a plain instant crossfade instead.

**Itinerary:** Day 1 (Marriage Nuptials → Arrival/Cocktail → Grand Entrance → Cake Cutting →
Toasts & Speeches → First Dance → Dance Floor Opens → Reception Dinner) and Day 2 (Ganesh Pooja →
Mehendi & Marigolds → Lunch → Baarat & Welcome → Wedding Pheras & Ceremony → Gala Dinner → After
Party), each with a line that draws itself in and staggered event reveals.

**Wardrobe:** merged Marriage Nuptials / Reception cards for Day 1 with a bold "wear anything but
white" caution card; merged Baarat/Pheras/Gala into one card for Day 2.

**Venue:** real photos of Bogmallo Beach Resort (you uploaded these), Google Maps embed + "Open
in Maps" link using Google's official documented URL formats (robust, not dependent on a
hand-typed place ID).

**Gallery:** masonry grid + lightbox with swipe/arrow-key navigation. Live-sync scaffolding is
in place but **not wired up** — see "Open items" below.

**RSVP:** Accept/Decline toggle (decline hides the guest-count/events fields, relabels the
button to "Send Our Regrets"), submits to Formspree (`https://formspree.io/f/xwpbgvnq`).

**Extras:** Add-to-Calendar (Google Calendar link + downloadable `.ics` for Apple/Outlook),
background music with a mute toggle (see notes below), a hamburger menu for mobile/tablet that
*coexists* with a horizontally-scrollable nav row (both always available, per your request), and
a favicon reusing the already-embedded monogram (zero extra bytes).

**Performance:** file size went from **2.89MB → ~1MB** (66% smaller) through:
- De-duplicating the monogram, which was embedded 10 separate times (nav, hero, 5 page headers,
  footer, 2 closing sections) — now embedded once and copied via JS on load.
- Discovering the masonry gallery was secretly re-embedding the same 4 photos already used as
  page-header backgrounds, plus a 5th internal duplicate — deduplicated the same way.
- Converting two venue photos from PNG to JPEG (PNG is a poor format for photographic content;
  this alone cut ~730KB with no visible quality loss).
- Resizing/recompressing the decorative floral graphic and the background music (128kbps →
  96kbps, stripped embedded album art and metadata).

---

## Real bugs found and fixed (not just guessed at)

These were diagnosed with actual verification — a local test server + real browser automation,
not just static code reading — so they're higher-confidence than typical "should be fixed" fixes:

- **Mobile horizontal overflow** ("empty strip on the right") — the hero photo's intentionally
  oversized background (used for the slow zoom effect) was tripping a browser quirk around
  negative offsets and `scrollWidth` reporting, even though it was already visually clipped.
  Fixed with `overflow-x:hidden` on `<html>` (only `<body>` had it before).
- **Mobile nav wouldn't scroll at all** — `justify-content:flex-end` pushed overflowing content
  off the *left* edge, where `scrollLeft` can never reach it. This was a **regression** — it had
  been fixed once before and silently came back when the nav was rebuilt for the hamburger menu.
  Fixed with `flex-start`, verified by actually moving `scrollLeft` and confirming every link
  becomes reachable.
- **RSVP hamburger menu showing transparent/broken** — the overlay used a 98%-opacity background
  with `backdrop-filter: blur()` and relied only on `inset:0` for sizing. Made it fully solid
  (no alpha), dropped the blur, and gave it explicit `width`/`height` (`100dvh`, the modern unit
  built for exactly this kind of mobile viewport inconsistency) and its own `z-index`.
- **Music not starting on tap** — two contributing issues: (1) the unlock-listener cleanup ran
  after the *first* interaction regardless of whether `play()` actually succeeded, and since
  `scroll` doesn't count as a valid browser "user gesture," a scroll firing first could burn the
  only attempt on something guaranteed to fail; (2) a single tap fires several events
  (`pointerdown`, `touchstart`, `touchend`, `click`) in quick succession, and without a guard,
  multiple overlapping `play()` calls could cause the browser to abort one in favor of another.
  Both fixed; also switched the `<audio>` tag to an explicit `<source type="audio/mpeg">` instead
  of relying solely on GitHub Pages' non-standard `Content-Type: audio/mp3` header.
- Removed a duplicate `<img>` element and stray characters left over from earlier edits (a BOM
  byte, an embedded newline inside a base64 string) that briefly corrupted the file during
  editing — all caught and fixed before ever being shipped, via the same JS-syntax/CSS-brace/
  div-balance verification run after every single edit in this conversation.

---

## Known open items — things you asked about but aren't done yet

- **Gallery live-sync with Google Photos**: the site-side code is in place (`GALLERY_SYNC_URL`
  constant near the top of the `<script>` block), but it requires a **one-time setup in your own
  Google account** that only you can do (a Google Apps Script + Cloud Console configuration).
  Full step-by-step instructions are in **`GALLERY_SYNC_SETUP.md`** in this folder. Until that's
  done, the gallery just shows the photos already built into the page — nothing is broken by
  leaving it unset.
- **Registry/gift info section**: you said yes to this early on, then priorities shifted to
  fixes — never actually built.
- **Custom domain**: you chose the free `github.io` subdomain for now with the explicit plan to
  add a custom domain later — GitHub Pages supports adding one at any time without needing to
  redo anything else.
- **Travel & accommodation / FAQ section**: discussed as an option, declined at the time — worth
  revisiting if you want it.
- **Song-request field on RSVP**: mentioned as a nice-to-have, not added.

---

## Notes for whoever (human or Claude) picks this up next

- The file is large (photos are base64-embedded), so a plain `Read` on the whole file will fail
  — read it in narrow line ranges and skip/avoid lines longer than ~2000 characters (those are
  the embedded images). `grep`/`awk` by line number work fine on the whole file.
- After **any** edit, verify before shipping: `node --check` on the extracted `<script>` block,
  and a brace-count / `<div>`-count sanity check on the `<style>` block and whole file. This
  caught several real corruption bugs during this build — don't skip it.
- A local test server (`node` one-liner serving this folder) + the browser automation tools were
  what actually caught the regressions above — static code reading alone repeatedly missed them.
  Worth doing the same for anything nontrivial.
- Don't `git add -A`/`git add .` — this folder has personal photos that must never reach the
  public repo. Add tracked files by exact name.
