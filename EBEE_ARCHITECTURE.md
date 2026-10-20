# eBee: architecture and handoff

**For: anyone picking up eBee, including a first-year CS student.**
Written to be read top to bottom. Each section says *what* was built, *why that approach*, and
*what was rejected and why* — because the reasoning is the part that is expensive to rediscover.

> **Status note.** Sections marked **[VERIFY]** were built in a later session and are described from
> release notes rather than from reading the code. Check them against the repository before relying on them.

---

## 1. What eBee is

eBee replaces the new-tab page in Brave and Chrome with a calm, macOS-inspired focus workspace:
a clock, weather, tasks, notes, habits, ambient sound, Spotify, a Pomodoro timer, a Zen mode and a
site blocker. It is a **personal project**, built on the open-source
[Meelio](https://github.com/zainzafar90/meelio) by Zain Zafar (MIT), and is not published or distributed.

**Three constraints shaped every decision:**

1. **No backend.** No server, no accounts, no sync. Everything lives in the browser.
2. **It must feel like macOS.** Frosted glass, traffic-light windows, unhurried easing.
3. **One person maintains it.** Favour boring, well-trodden solutions over clever ones.

---

## 2. How a browser extension works (the 2-minute version)

A Chrome/Brave extension under **Manifest V3** has three kinds of code, which cannot see each other
directly:

| Part | Where it runs | What it can do |
| --- | --- | --- |
| **Extension pages** (eBee's new tab) | A normal web page at a `chrome-extension://` address | Full DOM, extension APIs it declares |
| **Background service worker** | On its own, no page | Reacts to browser events (tabs, alarms, blocking rules). **Can be stopped at any time.** |
| **Content scripts** | Injected into other websites | Reads/changes that page. **Least trusted: a hostile site shares its world.** |

They communicate by **passing messages**. Two consequences run through this document:

- The background worker **cannot hold state in memory**, because the browser shuts it down when idle.
  Anything that must survive goes into storage.
- Content scripts must be treated as **untrusted input**, because any website they run on can
  influence them.

The `manifest.json` file declares permissions. **Every permission is a promise to the user**, so eBee
asks for as few as possible (see §9).

---

## 3. The stack, and why each piece

| Choice | Why | What was rejected, and why |
| --- | --- | --- |
| **TypeScript** | Types catch whole classes of bug before the browser runs anything. On a one-person project, the compiler is the code reviewer. | Plain JavaScript: faster to write, but with ~40 feature folders a rename becomes guesswork. |
| **React 18** | Describe what the screen *should* look like for a given state, instead of hand-patching the DOM. Meelio already used it. | Vanilla DOM, Vue, Svelte: all workable, but switching would mean abandoning Meelio's foundation for no user-visible gain. |
| **Zustand** (state) | A few lines per store, no boilerplate, and readable outside React (`useStore.getState()`), which matters for the sound engine and keyboard shortcuts. | Redux: more ceremony than a solo project needs. React Context: re-renders every consumer, which is visible at 60fps. |
| **Dexie** (IndexedDB) | IndexedDB is the only browser store that holds real volume (notes, tasks, photos). Dexie makes it bearable. | `localStorage`: ~5 MB, synchronous (it blocks painting), strings only. Fine for settings, hopeless for photos. |
| **Tailwind CSS** | Styles sit next to the markup, so there is no hunt for which rule applies. Makes a consistent design system practical. | Plain CSS or CSS modules: the indirection costs more than the tidiness is worth here. |
| **Framer Motion** | Declarative animation that interrupts gracefully — if you re-open a window mid-close, it blends rather than jumping. | CSS transitions: used where they suffice (see §6); they cannot express interruptible, state-driven motion as cleanly. |
| **WXT** (build) | Handles MV3 manifest generation, hot reload and bundling. | Hand-rolled Webpack: solved problems, re-solved badly. |
| **Vitest** | Same config as the bundler, fast enough to run constantly. | Jest: needs separate configuration for the same result. |

### Why fork Meelio rather than start from scratch

Meelio supplied an audio engine, a wallpaper library, 33 sound files, a working MV3 setup and an
i18n scaffold — months of work, MIT-licensed. Starting from zero would have meant re-solving all of
it to reach the point where the *interesting* work (the island, the window system) could begin.

**The cost of that choice:** eBee carries Meelio's architectural decisions, including some bugs
(see §10). The rule adopted was: **keep changes to Meelio's files small and commented, and build
eBee's own features in `features/`**, so the two stay separable. `NOTICE.md` records which is which.

---

## 4. Code layout

```
apps/extension/src/
  entrypoints/      newtab, popup, background, content script
  features/         eBee's own code (one folder per feature)
  components/       Meelio's original components
  stores/           Meelio's original stores
packages/shared/    Meelio's shared code (stores, UI, i18n) — changed sparingly
packages/infrastructure/  Dexie database models
scripts/validate/   browser end-to-end validators
docs/               this file, SECURITY.md, PUBLISHING.md, brand assets
```

**Why feature folders rather than `components/`, `hooks/`, `utils/`:** a change to notes touches
`features/notes/` and nothing else. Grouping by *technical kind* scatters one feature across five
directories; grouping by *feature* keeps a change local, which is what makes a codebase safe to
work on alone.

---

## 5. The main systems

### 5.1 Windows (`features/windows/`)

`app-window.tsx` is one shell — traffic lights, drag, zoom, frosted glass — that every window reuses.
`windows-host.tsx` enforces **one window open at a time**.

**Why one at a time:** overlapping windows need z-order, focus rings and "which is active?" logic.
On a new-tab page you are doing one thing. The simpler rule removed an entire class of bugs.

**Why reuse Meelio's dock flags** (`isTasksVisible` and friends) rather than a fresh store: anything
that opened Meelio's old sheets — dock, island, Help, keyboard shortcuts — opens eBee's windows
automatically. No audit of every call site.

**Two non-obvious details:**

- **Content renders one frame late** (`startTransition` + `requestAnimationFrame`). A heavy window
  rendering in the same frame it animates in causes a visible stall.
- **Drag stops with ~⅓ of the window on screen**, and never above the top edge, so a window can
  always be grabbed again.

### 5.2 The Dynamic Island (`features/island/`)

A notch at the top centre that shows the timer, soundscapes, Spotify and calendar reminders, and
expands on hover.

**Why one island rather than separate widgets:** Meelio put the calendar in its own pill, which
pushed eBee's island off-centre. Multiple floating widgets compete for the same space and for
attention. One surface that changes size is the whole point of Apple's design.

**Resting widths are deliberate** (idle 140px, one activity 152px centred, both 212px, song-change
peek 380px). A notch that is always wide looks like a toolbar; one that sizes to its contents reads
as alive.

**Two bugs worth knowing about, because both recur in hover UIs:**

- **Stuck expanded after Zen.** Zen is often started *from* the open island, so the island
  disappeared under the pointer and the browser never sent "mouse left". It now resets on hide and
  on return, and polls pointer position as a safety net.
- **Hidden behind the wallpaper.** It returned while Zen's backdrop was still fading out on top of
  it. It now waits for Zen to finish.

### 5.3 Sound (`features/sounds/`)

**The engine lives at page level, not in the Sounds window.** In Meelio, Shuffle and Drift run inside
the window's buttons, so they stop when you close it. Mounting `sounds-engine.tsx` once for the page
means they keep running — which is what a user expects from something described as "shuffle".

**Why Spotify uses PKCE OAuth:** the standard flow needs a client *secret*, and an extension has no
secret — its files are readable by anyone who installs it. PKCE exists exactly for clients that
cannot keep secrets.

**[VERIFY] Spotify sync.** Each UI piece polled Spotify separately every 5 seconds, which triggered
rate-limiting (the "it stops updating for a while" symptom). Now one shared poll per tab, honouring
Spotify's own back-off, with "nothing playing" requiring two consecutive empty responses so the
island does not blank between songs.

### 5.4 Notes **[VERIFY]**

Rebuilt twice. The sequence matters, because it is a good lesson:

1. **Markdown stored as text.** Bold was stored as `**bold**`, and the editor showed the asterisks.
   Wrong model: the user should never see the mechanism.
2. **TipTap (ProseMirror).** Formatting is stored as *marks on the text*, not characters inside it.
   Chosen over hand-rolling because rich-text editing — selection, undo, paste, composition —
   is a decade of edge cases.
3. **Photos.** Pasted photos broke on reload. The root cause was in Meelio's save code: **notes were
   silently truncated to 10,000 characters**. A base64 photo is hundreds of thousands, so it was cut
   mid-stream. Now photos are stored as real JPEG/PNG blobs in IndexedDB and the note holds a
   reference; the limit is removed.

**The lesson:** small test images never hit the limit. Tests that use unrealistically small or clean
data can pass while the feature is broken for real input.

### 5.5 Site blocker **[VERIFY]**

Blocks sites always, or only during focus sessions. Two bugs were found in Meelio's version: rules
were only installed with full host access, and "focus only" only knew about Zen, not the timer.

**Bypasses are enforced in the background, not the page.** A page can be opened by hand, framed, or
driven by a script; only the background knows whether a tab was genuinely just blocked. Limits:
**15 minutes maximum, one bypass per blocked visit**, and open tabs are sent back to the blocked
page when a bypass ends — because blocking rules only stop *new* loads.

### 5.6 Translation **[VERIFY]**

Meelio's i18n covers only Meelio's screens; eBee's ~1,050 strings were written in English in the code.
Rather than refactor every component, a **page-wide translator** walks the DOM and swaps fixed text,
with user content (tasks, notes, habits, song titles) explicitly marked "do not translate".

**Trade-off, stated honestly:** this is unusual. The orthodox fix is to route every string through a
translation function, which is cleaner but touches every file. The translator was chosen for reach
with limited time. Its risk showed up immediately: it must track every text node it changes, or
repeated switching strands text in the previous language — which is exactly the bug that appeared,
and was fixed.

Languages complete: **English, isiZulu, Español, 中文**. Others have navigation translated only.
isiZulu needs **a native speaker's review** — all wording is in `features/i18n/strings.ts`.

---

## 6. Motion, and why flicker happened

The design language: ease `[0.16, 1, 0.3, 1]`, 0.32s in, 0.16s out, **no springs** except the island
(critically damped, so it never wobbles past its size).

**Flicker was diagnosed by measurement, not guesswork.** Every frame of each animation was recorded
and its brightness tracked. A smooth fade moves in one direction; a flicker shows as a *reversal*.
Before: 6 reversals. After: 0, across three runs.

Four causes, all worth remembering:

1. **Frosted glass that faded in after landing.** A performance optimisation that read as a second
   "settle". Glass is now present from the first frame.
2. **Springs overshooting.** Menus bounced past their size and swung back.
3. **Menus sliding on exit.** Closing is now a pure fade, as macOS does.
4. **A rendering quirk in Framer Motion.** Simple fades handed to the browser's hardware animation
   engine can show their old value for one frame at the end. Such fades now run frame by frame.

**The frosted-glass trap, worth its own warning:** `will-change: opacity`, a leftover `filter` (even
`blur(0px)`), or `opacity < 1` on an **ancestor** silently kills `backdrop-filter` on its children.
The cards looked tinted but were not blurring at all, and had probably never been.

---

## 7. Data and storage

| What | Where | Why |
| --- | --- | --- |
| Notes, tasks, habits, photos | **IndexedDB** (Dexie) | Large, structured, asynchronous |
| Settings, prefs, saved mixes | **localStorage** | Small, needed synchronously on first paint |
| Spotify tokens | **`chrome.storage.local`** | Extension-only, not reachable from page scripts |
| Weather | **localStorage, 20-minute cache** | Be a good citizen of a free API |

**Prefs are versioned** (`ebee-prefs`, currently v3) with a migration function, so a new field never
breaks an existing install.

---

## 8. Testing: three layers, each for a different question

| Layer | Question it answers | Example |
| --- | --- | --- |
| **Unit tests** (~332, Vitest) | Is this logic correct, including edge cases? | Is 18:12 exactly "night"? Does one trackpad flick turn exactly one card? |
| **Browser validators** (5) | Does the real extension still work end to end? | Start the timer, block a site, change the wallpaper |
| **Frame measurement** | Does it *look* right? | Brightness per frame to detect flicker |

**Why pure functions are extracted for testing:** `clock-theme.ts`, `card-nav.ts` and
`focus-cycle.ts` hold no React, no DOM, no clock. Pass input, assert output. Logic entangled with a
component can only be tested by rendering it, which is slower and flakier. **The tests were written
before the code** in each case, which forced the edge cases to be decided first.

---

## 9. Security **[VERIFY]**

Audited as a reviewer would: *what can a website, another extension, or an automated agent make eBee do?*

| Finding | Fix |
| --- | --- |
| Any site could frame the blocked page and trick a click on Bypass (clickjacking) | Framing refused, by headers and by the page itself |
| Bypass could be granted on a hand-opened blocked page | Only the tab that was genuinely just blocked qualifies |
| URL text was inserted as HTML | Always inserted as plain text |
| Outside sites (including `localhost:4000`) could message eBee | No external messaging at all |
| Content scripts could send any command | Activity tracking only |
| No content security policy | Strict policy on all eBee pages |
| Scripted clicks on Bypass | Requires a deliberate 1.2s press; background limits are the real backstop |

**Stated plainly in `SECURITY.md`:** no extension can defend against someone who controls the
computer. Claiming otherwise would be false security. What *can* be defended is the boundary between
eBee and hostile websites, extensions and agents.

---

## 10. Bugs found in Meelio's own code

Useful because they show where to look when something inexplicable happens:

1. **Notes truncated to 10,000 characters** on save, silently. Corrupted every pasted photo.
2. **Site blocker installed no rules** without full host access, so nothing was blocked.
3. **"Focus only" blocking** only knew about Zen, not the timer.
4. **Shuffle and Drift** stopped when the Sounds sheet closed.
5. **A `blur(0px)` filter** left on the dashboard after its entrance animation, silently disabling
   frosted glass everywhere inside it.

---

## 11. Working on eBee

```bash
pnpm install
pnpm build:extension          # → apps/extension/.output/chrome-mv3
pnpm --filter extension test  # unit tests
pnpm --filter extension lint
pnpm validate:extension       # browser validators
```

`apps/extension/.env` (never committed): `WXT_API_URL`, `WXT_WEB_URL`, `VITE_CDN_URL`,
`WXT_SPOTIFY_CLIENT_ID`.

**The working agreement**, from the AI code quality guide this project follows:

1. Read the exact code involved; don't paste whole files.
2. State constraints before writing (no new dependencies, don't touch unrelated code).
3. Explain the approach before the code.
4. Write tests first for testable logic.
5. Verify every import exists; run types, lint, unit tests and validators.
6. Review the diff for unintended changes.
7. **Measure** visual and timing problems; don't guess.

---

## 12. What is deliberately not built

| Not built | Why |
| --- | --- |
| Accounts, sync, collaboration | No backend, by design |
| Sign in with Google/Apple | Both need a registered app and a server; Brave blocks Chrome's built-in Google sign-in |
| Arabic | Correct Arabic needs the whole layout mirrored right-to-left |
| isiZulu Bible verses | Published translations are copyrighted; inventing scripture would be dishonest |
| Live wallpapers from Meelio's site, "Ask AI", night wallpaper pairs | Only on Meelio's live site, not in the published code |
| Publishing eBee | It would compete with Meelio, whose foundation it is built on |

---

## 13. Ownership

- **Meelio's code**, including files eBee modified: MIT, © Zain Zafar. The notice stays.
- **eBee's own work** (everything in `features/` except `content/`, `site-blocker/`, `zen-mode/`;
  eBee's tests and docs; the eBee name, logo and icons): © 2026 Thando Shabangu, all rights reserved.

See `NOTICE.md`, `LICENSE-EBEE.md` and `docs/PUBLISHING.md`.
