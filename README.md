<div align="center">

<img src="media/ebee-logo.svg" width="96" alt="eBee logo" />

# eBee

**A calm, macOS-inspired focus space that lives in every new browser tab.**

<!-- Replace with your own recording. GitHub plays GIFs inline; for MP4, link to it instead. -->
   <img src="media/ebee-island.gif" width="820" alt="The Dynamic Island expanding on hover" />

</div>

> **A personal project.** eBee isn't published or distributed; it's the new tab I use every day.
> This repository is a showcase: the source code is private.
> Happy to walk through the code and architecture on request: [github.com/Alch3mist-42](https://github.com/Alch3mist-42).

## What it does

| | |
| --- | --- |
| ðŸŽ§ **Dynamic Island** | A notch at the top of the screen for the focus timer, ambient soundscapes and Spotify, with mute, song-change peeks and calendar reminders. It expands on hover, and its colours follow the album art. |
| ðŸªŸ **macOS-style windows** | Tasks, Notes (block editor with undo/redo), Habits, Sounds, Calendar, Wallpaper, Links, Site Blocker, Shortcuts and Settings, with traffic-light controls, dragging and zoom. |
| ðŸƒ **Card stack** | Weather, today's tasks, upcoming events, habits, focus this month and world clocks, as a one-card-per-minute slideshow with continuous scrolling. |
| â±ï¸ **Focus & Zen** | A Pomodoro timer that keeps running across views, a pop-out timer window, full-screen Zen with a breathing guide, and a Wind Down mode for the evening. |
| ðŸ›¡ï¸ **Site blocker** | Blocks distracting sites always or only during focus, with a calm blocked page and strictly limited bypasses. |
| ðŸŽ¨ **Personal** | Eight accent colours, a Standard or Editorial typeface, a day/night clock face, saved custom wallpapers and a rotating greeting line. |
| ðŸ”’ **Offline-first** | No accounts, no servers. Everything is stored in the browser (IndexedDB and localStorage). |

## A closer look

### The home screen

<img src="media/home.png" width="820" alt="The eBee home screen" />

### The Dynamic Island
<img src="media/island.png" width="820" alt="The Dynamic Island showing the focus timer, soundscapes and Spotify" />
 
### Windows, in either theme

<img src="media/windowA.png" width="410" alt="The Tasks window in the dark theme" /> <img src="media/windowB.png" width="410" alt="The Notes window in the light theme" />

### Zen mode and personalisation

<img src="media/zen.png" width="820" alt="Zen mode with the breathing guide" />

<img src="media/settings.png" width="820" alt="Accent colours and typeface in Settings" />

## Built with

TypeScript Â· React 18 Â· Tailwind CSS Â· Framer Motion Â· Zustand Â· Dexie (IndexedDB) Â·
Chrome Extension Manifest V3 (WXT) Â· Vitest Â· browser-based end-to-end validators

## What I learned building it

- **Measuring beats guessing.** Animations stuttered when windows opened. Rather than guess, I recorded
  every frame and tracked brightness over time, which showed the animation reversing for a single frame.
  The cause was a known rendering quirk, not my own code.
- **Thinking adversarially.** I audited eBee the way a security reviewer would, asking what a malicious
  website, another extension or an automated agent could make it do, and fixed clickjacking, message
  isolation, a missing content security policy, and a way to abuse the site blocker's bypass.
- **Verifying AI-assisted work.** I used AI heavily to move fast during an exam week while owning every
  product and design decision, and verified each change with tests, type checking and measurement.
- **Testing what matters.** 299 unit tests, plus browser validators that drive the real extension through
  the timer, site blocker, greeting, wallpaper and breathing flows.

## Built on Meelio

eBee is built on **[Meelio](https://github.com/zainzafar90/meelio)**, an open-source new-tab app by
Zain Zafar (MIT License), which gave me a solid foundation to design on top of.

## Ownership

Â© 2026 Thando Shabangu. eBee's own design, features, name and logo are all rights reserved.
Meelio's code remains under its MIT License. eBee is not affiliated with or endorsed by Meelio or Spotify.




