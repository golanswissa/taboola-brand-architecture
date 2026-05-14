# MIGRATION.md — Primer for a fresh Claude agent

> Written 2026-05-14. Read this first. Then read `index.html`. Then ask
> the user what they want next.

---

## 1. What this project is

An interactive single-page HTML visualization of the **Taboola Brand
Architecture / Brand Hierarchy** chart. It shows how Taboola's brand
sits above its products — Publisher Solutions (Feed, Newsroom, Header
Bidding, Performance Hub, etc.), DeeperDive, the Realize ad platform
(with its sub-products), and the endorsed brands Shop Your Likes and
Skimlinks (both Connexity / Taboola companies).

- **Source of truth (design):** Figma file linked from the in-page
  "Brand System" button —
  `https://www.figma.com/design/ufrJPFax2cvYDqAagnusKh/Realize?node-id=2250-5477`
- **Live deployment:** `https://taboola-brand-architecture.vercel.app/`
  (auto-deployed from `main` on push)
- **GitHub remote:** `https://github.com/golanswissa/taboola-brand-architecture.git`
- **Goal:** a polished, animated, presentation-ready chart with
  light/dark mode, hover/expand interactions, and click-to-zoom popups
  for the design-system reference images.

---

## 2. Repo / folder layout

The whole "app" is one HTML file. There is no build step, no framework,
no node_modules. Everything is inline `<style>` and `<script>` in
`index.html`.

```
Chart/
├── index.html                                  # THE app (~445 lines, all inline)
├── taboola-brand-architecture.html             # Stale duplicate copy — ignore unless asked
├── package.json                                # Just defines `npm run dev` -> npx serve . -p 3000
├── 1.png, 2.png, 3.png, 4.png                  # Design-system reference popups (clicked from Realize sub-rows / DeeperDive)
├── realizePop.png                              # Default popup image (fallback)
├── shopYoulikes.png                            # Asset, currently unreferenced in code
├── DeeperDive.svg                              # Logo on the DeeperDive card
├── Abby.svg                                    # Logo on Realize "Abby" sub-row
├── SkimLinks.svg                               # Logo on Skimlinks card
├── TaboolaFeed.svg, TaboolaNewsRoom.svg        # Logos in expanded Publisher Solutions pills
├── SYL_Logo_Linear_Classic-Blue-Red2024-1-…png # Shop Your Likes logo
├── cnx-logo.svg                                # Connexity logo (stacked under SYL & Skimlinks)
├── .claude/
│   ├── settings.local.json                     # Permission allowlist for git/gh/preview
│   ├── launch.json                             # Preview config (port 3000)
│   └── worktrees/<branch>/                     # Where Claude worktree branches live
└── .git/                                       # Standard git
```

Branches:
- `main` — what Vercel deploys
- `claude/naughty-joliot-308427` — current worktree branch (this session's work)

---

## 3. Where session memory lives

Claude Code memory is stored on the **local filesystem**, not in your
account. Path:

```
~/.claude/projects/<encoded-project-path>/memory/
```

Where `<encoded-project-path>` replaces every `/` with `-`. For this
project as opened via the worktree, that's:

```
~/.claude/projects/-Users-golanswissa-Downloads-Chart--claude-worktrees-naughty-joliot-308427/memory/
```

And for the project opened at its real root:

```
~/.claude/projects/-Users-golanswissa-Downloads-Chart/memory/
```

**Implications:**
- **Same machine + new Claude account → no migration needed.** Memory
  lives next to project transcripts, indexed by path, not by account.
  Just open the folder with the new account; the same files are still
  there.
- **New machine → manually copy the `memory/` folder** (and ideally
  the whole `~/.claude/projects/<encoded-path>/`) to the same path on
  the new machine.
- **As of this writing, this project has NO `memory/` folder yet** —
  no prior agent saved structured memory. The history lives in
  `~/.claude/projects/-Users-golanswissa-Downloads-Chart--claude-worktrees-naughty-joliot-308427/<session-id>.jsonl`
  (the raw transcript) and in git commits. So this MIGRATION.md is
  doing memory's job for the next agent.

---

## 4. Standing user preferences

Inferred from this session (no `feedback_*.md` notes exist yet, so
treat these as observed defaults, not gospel):

- **Iterate visually, fast.** User sends screenshots with annotations
  more often than written specs. They expect changes to show up in the
  preview / localhost / Vercel quickly and will eyeball them.
- **Keep it one file.** No frameworks, no build step, no dependencies.
  This is a hand-written HTML visualization. Don't propose a refactor
  to React / Vite / a build pipeline unless asked.
- **Prefers a localhost URL on a specific port** when reviewing
  changes (port 8082 was used in this session). Start a Python
  `http.server` on the requested port; don't assume `npm run dev`.
- **Vercel is the canonical preview** — the user often says "this is
  how I show it on Vercel, double check" and means the deployed
  `taboola-brand-architecture.vercel.app` URL.
- **Pixel-precise coordinate work is welcome.** When repositioning,
  the user thinks in terms of "mirror to the left", "to the left of
  X", "exactly like the right but mirrored". Compute the math, don't
  hand-wave.
- **No emojis in code or chat** unless the user explicitly asks.
- **Plan before building when asked.** When the user shows you a
  spec image and says "do not build, what do you understand", they
  want a plain-English summary and clarifying questions, not code.

---

## 5. Critical gotchas / lessons learned

These will bite a new agent:

1. **Fixed canvas, scaled via CSS.** The chart is a 1728 × 1117
   absolute-positioned canvas. JS only sets `--s` (scale), `--tx`,
   `--ty` to fit it to the viewport. **All `top:` / `left:` numbers in
   the markup are in 1728-canvas coordinates, not viewport pixels.**
   Don't try to use `vw` / `%` — it'll break the layout.

2. **Stems are SVG paths in `#SV`, hand-coded for each card.**
   Whenever you move a card you MUST also update the matching `<path>`
   or `<line>` in the `<svg id="SV">` block (lines ~173–180). The
   curves are quadratic / cubic Béziers anchored at Taboola
   `(864, 129)`. Forgetting this leaves a stem dangling at the old
   position.

3. **Light-mode overrides use attribute selectors on inline styles.**
   Look at the top of `<style>`:
   ```css
   body.light .N span[style*="color:#78aced"]{color:#042651 !important;}
   ```
   This means: if you introduce a new inline color, you ALSO need a
   matching `body.light` rule, or it'll look broken in light mode.
   Existing colors that already have light-mode rules: `#c9ff5c`,
   `#6a8735`, `#042651`, `#78aced`.

4. **Coming-Soon pill must be a SIBLING of the grayed-out content,
   not inside it.** The card's body is wrapped in a div with
   `opacity:.45; filter:grayscale(.6)`. The `<span class="cs-pill">`
   sits next to that wrapper so it stays crisp. If you put the pill
   inside the grayed wrapper, opacity bleeds through and the pill
   looks washed-out.

5. **Animations are paused until the entry screen is dismissed.**
   `.au, .ap, .al, .ar, .ln` and `#moat` all have
   `animation-play-state:paused` until `#C.animate` class is added by
   `enterChart()`. If something looks "frozen" in dev, click "View
   Hierarchy" on the entry screen.

6. **There are TWO HTML files.** `index.html` is the live one.
   `taboola-brand-architecture.html` is an older identical copy. Edit
   only `index.html` unless the user explicitly says otherwise. (Worth
   asking the user if the duplicate should be deleted.)

7. **The expanded Publisher Solutions and Realize columns use absolute
   overlays** (`#pub-expand`, `#rlz-expand`) so they don't push other
   layout when opened. If you reposition a card vertically, account for
   this overlay's `top:` value.

8. **Mirror axis = x = 864.** When the user says "mirror this to the
   left side", reflect across x=864 in canvas coordinates:
   `new_x = 1728 - old_x`. For a card with width `W` at `left:L`, the
   mirrored card starts at `left = 1728 - L - W`.

---

## 6. Inventory of what exists

**Top-level UI**
- Entry splash screen with animated aurora background and a "View
  Hierarchy" button (`#entry`).
- Top-right controls: "Brand System" link → Figma, theme toggle
  (light/dark), persisted via `localStorage`.

**Chart cards (current positions, post-restructure)**
| Card | x (left) | y (top) | width | side | notes |
|---|---|---|---|---|---|
| Skimlinks | 54 | 229 | 193 | publisher (was advertiser) | Blue, **grayed + Coming Soon pill** |
| Shop Your Likes | 288 | 229 | 193 | publisher (was advertiser) | Blue, **grayed + Coming Soon pill** |
| Design System pill | 559 | 241 | 136 | (above DeeperDive) | Green outline |
| DeeperDive | 501 | 320 | 252 | publisher | Click → opens `4.png` popup |
| Publisher Solutions | 773 | 229 | 297 | publisher | "+" toggle expands 6 sub-pills |
| Realize | 1247 | 229 | 297 | advertiser (alone on the right) | "+" toggle expands 4 sub-pills; sub-rows click → `1/2/3.png` popups |
| Taboola (top) | 753 | 77 | 222 | center | Logo box, currently NOT expandable |
| Data Signal Moat (bottom) | 45 | 995 | 1629 | full-width | Marching-ants stroke, glow, **Coming Soon pill** |

**Stems (`<svg id="SV">` lines 173–180)**
- Taboola → Publisher Solutions (blue cubic Bézier)
- Design System → Publisher Solutions (horizontal blue line at y=257)
- Taboola → Realize (green cubic Bézier)
- Taboola → Shop Your Likes (blue, mirrored from original right-side)
- Taboola → Skimlinks (blue, mirrored from original right-side)
- Design System → DeeperDive (vertical blue line)

**Interactions**
- `togglePublisher()` — sequential reveal of 6 publisher pills with
  connector segments
- `toggleRealize()` — sequential reveal of 4 Realize sub-product pills
- `openRealizePopup(img)` — full-screen modal with blurred backdrop,
  closeable via background click
- `toggleTheme()` — switches `body.light`, persists in localStorage
- `enterChart()` — hides splash, kicks off the chart's CSS animations

**Removed in this session**
- Center vertical divider line (`#center-line` is now `display:none`)
  because the right side has only Realize and the divider made the
  imbalance look wrong.

---

## 7. What's done and what's not done

**Done this session:**
- Moved Skimlinks + Shop Your Likes from advertiser side to publisher
  side, recolored blue, mirrored their stems exactly across x=864
- Restructured publisher-side x-positions to fit four cards (Skimlinks,
  SYL, DeeperDive, Publisher Solutions)
- Pushed Design System pill + DeeperDive + Publisher Solutions right
- Pushed Realize right (from x=909 to x=1247) so it sits cleanly alone
- Removed the center divider line
- Added a green "Coming Soon" pill inside the Data Signal Moat
- Grayed out (opacity .45 + grayscale .6) Skimlinks and SYL and added
  small gray "Coming Soon" pills above each

**Uncommitted:** all of the above is in the working tree of branch
`claude/naughty-joliot-308427`. `git status` shows `index.html` modified.
Not yet pushed.

**Spec the user shared but did NOT yet ask to build** (image
annotation, last turn before this MIGRATION.md):
1. Hover on **DeeperDive** and **Realize** → small dropdown opens with
   `Design System+` and `Messaging+` options.
2. Hover on **the rest** of the cards → shows a "Coming Soon" state
   (grayed out).
3. **The Moat** → coming soon (already partially done — the pill is
   there, but maybe the whole moat should also be visually subdued).
4. **Taboola** (the top pill) → "Allow expand" — should be clickable
   to expand; what it reveals is unspecified.

The agent (this one) asked clarifying questions about #1, #2, and #4
and is awaiting the user's answers. Don't start building these without
those answers.

---

## 8. How the user works

- **Communicates in screenshots, often hand-annotated.** Treat the
  image as the spec. Read it carefully.
- **Writes quickly with typos** ("teh", "ned", "ithn k") — don't get
  hung up on the typos, parse the intent.
- **Iterates one or two changes at a time.** Doesn't batch big
  feature requests; expects you to ship a small change, see it on
  Vercel or localhost, then come back with the next one.
- **"Do not build, what do you understand?"** is a real signal.
  When given a spec image without a build instruction, summarize and
  ask, don't code.
- **Asks for a localhost URL on a specific port** when they want to
  preview locally. They will visit it themselves.
- **Pushes to Vercel via `git push origin main`** (Vercel watches
  `main`). The user usually does this themselves; don't push without
  being asked.

---

## 9. First moves for the next agent

In order, on turn 1:

1. **Read `index.html` end-to-end** (it's only ~445 lines). The whole
   project is in there. Do this before answering anything substantive.
2. **Run `git status` and `git log --oneline -10`** to see what's
   uncommitted and what was recent. As of 2026-05-14, the working
   branch is `claude/naughty-joliot-308427` with uncommitted edits to
   `index.html` (the restructure described in §7).
3. **Check whether `~/.claude/projects/-Users-golanswissa-Downloads-Chart/memory/`
   exists.** If yes, read `MEMORY.md` first — it supersedes anything
   in this file that has gone stale.
4. **Verify the live URL still matches your understanding:**
   `https://taboola-brand-architecture.vercel.app/`. The user often
   references it.
5. **Ask the user the open questions from §7** before building any of
   the four pending hover/expand specs:
   - For DeeperDive/Realize hover dropdown — are "Design System+" and
     "Messaging+" themselves clickable, and to what?
   - For the "rest of the cards" coming-soon — tooltip, overlay pill,
     or full-card dim?
   - For Taboola "Allow expand" — what content should expanding
     reveal?
6. **Only start a localhost server on a port the user names.** If
   they don't name one, ask. Default Vercel preview is the safer
   review surface.

---

## Honest caveats

- No prior memory files exist; everything in §4 ("standing
  preferences") is inferred from one session of observation. Treat as
  starting point, not as established law. Update as you learn more.
- The duplicate `taboola-brand-architecture.html` file's purpose is
  unclear. Worth asking the user whether to delete it.
- `package.json`'s `dev` script uses port 3000, but in this session
  the user asked for 8082. There's no canonical port; ask each time.
- The Figma node ID in the "Brand System" link points to a Realize
  file, not specifically a brand-architecture file. Whether this is
  the actual source of truth or just a related design doc is not
  confirmed.
