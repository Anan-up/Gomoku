[English](README.md) | [简体中文](README_Simplified_Chinese.md) | [繁體中文](README_Classical_Chinese.md)

# Gomoku

A fully local, browser-based Gomoku (Five in a Row). Double-click `gomoku.html` and you are playing — no network, no installation, no third-party dependencies.

The computer plays back, and **as Black it is essentially impossible to beat** — it carries an Alpha-Beta engine (9-cell pattern lookup table + incremental potential evaluation + transposition table + VCF continuous-four threats). It does not win by memorizing openings.

![Interface preview](assets/preview.jpg)

---

## Quick start

1. Unzip the whole folder (**do not take `gomoku.html` on its own** — the background images and music live in `assets/`).
2. Double-click `gomoku.html`; it opens in Chrome / Edge / Safari / Firefox.
3. Click any intersection on the board to place a stone. The computer answers automatically.

Background music starts on its own the first time the page opens. If the browser blocks autoplay, click anywhere on the page to start it.

---

## Directory structure

```
gomoku/
├── gomoku.html          The game itself (HTML + CSS + JS + engine, all inlined)
├── README.md            This file
└── assets/
    ├── bgm.mp3          Background music: "Ascetic Mountain on the Rooftop"
    ├── avatar-ai.png    The computer's avatar
    ├── preview.jpg      Interface preview (README only, safe to delete)
    ├── audio/
    │   ├── end-laugh.mp3 End-of-game sound (played when the game ends)
    │   └── undo.mp3      Undo sound (played when you click "Undo")
    ├── video/           Video background
    │   ├── bg-loop.mp4  Seamless-loop background video (1600×900 / 30fps / ~7MB)
    │   ├── poster.jpg   Video poster frame
    │   └── thumb.jpg    Video thumbnail for the wallpaper wall
    └── bg/              17 background images
        ├── bg01.jpg
        ├── bg02.png
        └── …
```

---

## Interface layout

```
┌───────────────────────────────────────────────────────────┐
│  Gomoku                                       ♪  🖼       │
├──────────┬───────────────────────────┬────────────────────┤
│ Computer │                           │ You play           │
│  avatar  │   Frosted-glass board     │  Black/White/Random│
│ Kushan   │  (wallpaper shows thru)   │ New game / Undo    │
│  status  │                           │ Move list          │
│   VS     │                           │ 17 wallpapers+video│
│ You (●)  │                           │                    │
└──────────┴───────────────────────────┴────────────────────┘
```

- **Top left** is your opponent, "Kushan" — the only side with an avatar; the ring around it spins while it thinks.
- **Bottom left** is you, marked by a single stone instead of an avatar, coloured according to whether you hold Black or White.
- **Centre** is the frosted-glass board: `backdrop-filter` blurs the wallpaper or video behind it in real time so the colours bleed through, with a hazy, highlighted glass edge.
- **The side cards are frosted glass too**: translucent fill + live blur + a very fine frosted grain.
- **Right** holds every control.

---

## How to play

### Do you take Black or White

Pick one of three on the right-hand panel; **switching starts a new game**:

The three buttons choose which side **you** take, and the computer takes the other. The page defaults to "White" — meaning **the computer has Black and moves first**, opening at the centre point (tengen).

| Option | Meaning |
|---|---|
| **Black (you move first)** | You play Black and lead; the computer is White. Pick this if you want to open yourself |
| **White (computer moves first)** | **The default.** The computer has Black, moves first, and calculates every move after that |
| **Random** | Assigned randomly each game; the panel shows the result for this game |

### Rules

- 15×15 board, **no forbidden moves**, five in a row horizontally, vertically or diagonally wins.
- A full board with no winner is a draw.

### Controls

| Action | How |
|---|---|
| Place a stone | Click an intersection |
| Undo | Click "Undo", or press `U` |
| New game | Click "New game", or press `N` |
| Change wallpaper | Click any thumbnail under "Wallpaper" on the right; the 🖼 button in the top bar advances to the next one |
| Video background | The first cell of the wallpaper wall, "▶ Video"; once selected it loops and auto-rotation stops |
| Auto-rotation | The "Auto-rotate" switch at the top right of the wallpaper panel; when on, it changes every **1 minute** (no rotation in video mode) |
| Music on/off | The ♪ button at the top right |
| End-of-game sound | Plays automatically on a win, loss or draw; nothing to do |
| Undo sound | Plays when you click "Undo"; clicking repeatedly restarts it each time |

**Undo** takes back two moves — the move you just made plus the computer's reply — returning the position to just before your last placement. You can also undo after the game has ended; it re-enters the playing state.

---

## What the computer is thinking (text view)

No halos or glow rings are drawn on the board. The computer's thoughts are written straight into the text box on the left card and update as it calculates:

```
▸ Move 5 · 36 candidate points
▸ Depth 8 reached · 62.0k nodes
▸ Currently liking G6
```

| Line | Meaning |
|---|---|
| Line 1 | Which move this is, and how many candidate points near the stones are being evaluated |
| Line 2 | How deep iterative deepening has gone, and how many nodes have been searched |
| Line 3 | Mid-search it reads "currently liking"; once done it becomes "playing X" |

**No forced waiting**: the engine drops its stone as soon as it finishes, and that line of text stays until the next move, so a fast calculation does not flash past.

The engine prefers to run in a `Web Worker`, so the interface never stutters. When you open `gomoku.html` directly (`file://` protocol) the browser refuses to start a `Blob Worker`, so it falls back to **sliced in-page computation**: the main thread is yielded once per depth increment, so those three lines still refresh level by level — the deep layer just pauses briefly.

---

## Algorithm and strength

The engine is a pure front-end searcher: no opening book, no network, no server.

### Pattern recognition: every empty point, every direction

Before moving, it answers one question — **if I place here, what does this direction become?** For each point it takes the 9 cells centred on it in each of the 4 directions and encodes them as a base-3 number (0 empty / 1 own / 2 opponent or wall), then looks up a precomputed table of 19,683 entries to get the pattern grade directly:

| Grade | Pattern | Meaning |
|---|---|---|
| 6 | Five | Placing here wins |
| 5 | Open four | Two winning points; cannot be blocked |
| 4 | Closed four | One winning point; the opponent must answer |
| 3 | Open three | Can become an open four next move |
| 2 | Sleeping three | A three blocked at one end |
| 1 | Open two | Still developing |
| 0 | None | — |

This recognises **broken threes** "●●_●", **broken fours** "●●_●●" and these **blocked-at-both-ends** shapes that statistics over a 5-cell window cannot see — the fundamental reason it defends better than the previous version.

### Placing a stone only recomputes affected directions

Every empty point maintains "grade if Black plays here, grade if White plays here" × 4 directions — 1,800 numbers in all. Placing one stone only affects empty points within ±4 cells on the same lines, so a move recomputes a single direction for about thirty points and leaves the rest untouched; the evaluation score is adjusted incrementally at the same time, so leaf nodes fetch a score in O(1).

### Evaluation: aggregated per cell, so closed fours cannot farm points

Each empty point takes "strongest direction + 0.45 × second strongest direction" as its potential, then all empty points are summed:

| Pattern | Weight |
|---|---|
| Open four | 1,500,000 |
| Closed four | 26,000 |
| Open three | 1,200 |
| Sleeping three | 80 |
| Open two | 10 |

Aggregating per cell rather than adding the 4 directions directly matters because the latter lets a point that "produces several closed fours at once" score over and over — the computer would then wallow in a pile of closed fours that the opponent casually blocks. "Strongest + 0.45 × second strongest" keeps the value of double-threes and four-three combinations without letting four low-value directions inflate the score.

Five-points **do not enter the potential**; they are counted per cell separately, because "the opponent can make five next move" is not the same as "I win" — it depends on whose turn it is. In evaluation:

- If it is my turn and I can make five → award a win score outright
- If the opponent has two or more five-points (an open four) → award a loss score outright
- If the opponent has exactly one five-point → I must block it, and the initiative bonus is deducted

Every stone also carries **positional value** (higher toward the centre). Without that term, the centre point and a stone hugging the edge would evaluate the same at the opening, the computer would undervalue the initiative, and playing Black would get worse and worse.

### Search

| Mechanism | Effect |
|---|---|
| Iterative deepening | Starts at 2 plies and deepens; stops when time runs out, keeping only the complete result of the last finished depth |
| Transposition table | 64K-entry Zobrist table caching "position → score + best move" |
| PVS | Full window for the first move, null window for the rest, re-search on a fail-high |
| Killer moves | Two moves that recently caused a cutoff are remembered per ply and tried first |
| Candidate pruning | Only within 2 cells of a stone, sorted by "my grade here + 1.05 × opponent's grade here", top 8–16 kept |
| Forced moves | Can make five → play it; opponent can make five → only block; can make an open four → play it |
| Sliced execution | The main thread is yielded once per depth increment, so the UI can refresh "depth N reached" level by level |

**Defensive weight is slightly above offensive** (1.05). A point where the opponent could make an open four always sorts to the front of the candidate list and cannot be crowded out by my own attack — this is the key to "never missing a defence".

### Killing sequences

VCF (continuous fours) runs before the main search: up to 18 plies, 420 ms cap. If a forced line of "four after four until five" exists, it plays it at once. During recursion it checks whether the opponent can counter with five, so it does not compute a false kill.

Before moving there is one more self-check: play the chosen move, then run VCF once more from the opponent's point of view. If the opponent can kill with continuous fours, it switches to the second and third candidates. This specifically cures the "looks dominant, actually walks into a kill" move.

### Measurements

Self-play in the same process, no random noise, time limit per move:

| Opponent | Result |
|---|---|
| Rule-based defensive AI (make five / block five / block open four; almost never misses a kill) | As Black 5 : 0 |
| Rule-based + its own continuous-four kill (more like a human who calculates kills, randomised) | As Black 18 : 0 |
| This engine against itself, White given 2 extra seconds (3200 vs 5200) | As Black 3 : 0 |
| This engine against itself (White 1800 ms) | As Black 2 : 0 |
| The previous engine (1600 ms) | As Black 4 : 0 |
| Black given only 1200 ms, against "rules + four-kill" | As Black 6 : 0 |

Two more rounds were run in the browser:

- A script simulating a player who **places stones at random**: killed in 11–13 moves.
- A script simulating a **rule-abiding player** (make five when you can, block when I could make five, otherwise play by proximity): four games in a row all won by the computer, killed in 9–11 moves.

### About "unbeatable"

- The page gives Black to the computer by default (you are White), so on opening you see it take the centre point; the first 6 moves get an extra 0.9 s so the opening does not scatter.
- This is a **strong engineering guarantee**: across more than forty games against "rule-based defensive AI / rule-based AI that calculates four-kills / the previous engine / itself (with White given 2 extra seconds)", the computer won every game as Black — not one was drawn or lost.
- It is not a mathematical proof. Gomoku without forbidden moves is a theoretical win for Black, and exhaustively verifying that in JS within 3 seconds is unrealistic; in principle a move sequence aimed specifically at it may still exist. In everyday play, don't count on beating it.
- The computer is **not** guaranteed to win as White — the initiative belongs to Black, so being able to win as White would be unreasonable.

## Customisation

Everything lives in `gomoku.html`; search for the corresponding spot in a text editor:

| What you want to change | What to search for | Notes |
|---|---|---|
| Thinking time | `aiThinkTime` | Black 3200 ms / White 1800 ms, plus 900 ms for the first 6 moves; larger = stronger and slower |
| Search depth cap | `maxDepth: 14` | The depth cap passed from the page to the engine; the real depth is limited by time |
| VCF depth / time cap | `findVCFMove(me, 18, Math.min(420, …))` | Ply count and time limit for continuous fours |
| Music volume | `audio.volume` | Default `0.34`, range 0–1 |
| Board palette | `paintScene` | Currently translucent neutral glass with a deep blue-grey grid; edit the gradients here for a wood tone |
| Board transparency | `glassAlpha` | 0.06–0.15 for images, fixed 0.13 for video; lower is more transparent |
| Board blur | `backdrop-filter` on `.board-box` | Default `blur(26px) saturate(1.38) brightness(1.01)` |
| Card frosted glass | `backdrop-filter` on `.card` | Default `blur(32px) saturate(1.65) brightness(1.06)`, with `background-color` around 0.38 |
| Wallpaper rotation interval | `60000` | Milliseconds for auto-rotation; currently 1 minute |
| Background blur | `filter:blur(3.5px)` on `.bg-layer` | Set to 0 for a fully sharp background (it will compete with the board) |
| Change background images | The `BG_LIST` array | Drop your images into `assets/bg/` and edit the filenames in the array |
| Change the background video | `VIDEO_BG` | Defaults to `assets/video/bg-loop.mp4`; replace it under the same name |
| Change the music | `assets/bgm.mp3` | Replace under the same name |
| Change the end-of-game sound | `assets/audio/end-laugh.mp3` | Replace under the same name; any length works |
| Change the undo sound | `assets/audio/undo.mp3` | Same as above; currently a 0.86 s blip |
| End-of-game sound volume | `sfx.volume` inside `playEndSfx` | Default `1`; drop to around 0.5 if it is too loud |
| Undo sound volume | `undoSfx.volume` inside `playUndoSfx` | Default `0.9` |
| How far the music ducks | `audio.volume = 0.09` inside `playEndSfx` | The level the background music is lowered to during the laugh |
| Board size | `var N = 15` | Note: star-point coordinates are hard-coded in `paintScene`, so change them together |

The background choice is stored in the browser's `localStorage` (key `gomoku_bg`); images are stored by index and the video as `video`, so the next time you open it you get the same one.

---

## FAQ

**Blank page / board does not show?**
Use a modern browser (Chrome 80+, Edge 80+, Safari 14+, Firefox 78+). IE is not supported — it needs `Web Worker` and `Canvas`.

**No background image, no music?**
Most likely only `gomoku.html` was copied. Put the `assets/` folder alongside it. The game still plays fine without the resources — just no background and no sound.

**Video background does not play / stutters?**
The video is muted and looping, so browsers allow it to autoplay; if you paused it manually or the system's power-saving mode restricts it, click the page once to resume. At 1600×900 it is far friendlier to older machines than 4K; it pauses automatically when the tab goes to the background and resumes when you come back.

**The game ended but I heard no laugh?**
The sound plays about 0.24 s after the result is revealed (waiting for the stone animation to settle). If the browser has blocked audio on a page that was never clicked, click the board or "Play again" to unlock it; clicking "Play again" cuts the laugh off immediately, which is expected.

**No sound on undo?**
Two normal cases: there is no stone of yours on the board yet (only the computer's opening move), so there is genuinely nothing to undo; or the browser has not unlocked audio yet, which the first click on the board fixes.

**Music does not autoplay?**
That is the browser's autoplay policy — click anywhere on the page and it starts. Expected behaviour, not a bug.

**Does it work on a phone?**
Yes. On narrow screens it switches to a stacked layout, the board fits the width, and the buttons reposition. Landscape or portrait both work.

**The frosted board turned into a plain translucent panel?**
Frosted glass relies on CSS `backdrop-filter`. Safari 14+, Chrome 76+ and Firefox 103+ support it; older browsers degrade to a translucent panel — the stones still play, just without the blur.

**Why are the Black stones grey-blue dots in the move list?**
The panel background is dark, so pure black dots would be invisible. The Black stones on the board are still properly black.

---

## Technical notes

- Single-file front end: zero dependencies, zero build, zero network requests.
- Rendering: `Canvas 2D`, scaled by `devicePixelRatio` so high-DPI screens stay crisp; the board is drawn translucent and the outer `.board-box` uses `backdrop-filter` for the frosted glass.
- Wallpapers: two `div`s cross-fading, with the dimming mask and glass thickness adjusted automatically from each image's average brightness.
- Video background: `<video muted loop playsinline>` filling the viewport, with a 1.5 s cross-fade at head and tail so the loop has no visible jump; entering video mode stops image rotation, and going to the background pauses it.
- Computation: `Web Worker` (engine code inlined as `text/js-worker`, started via a `Blob`); when the Worker is blocked under `file://` it degrades to sliced in-page computation (`searchBegin` / `searchStep`, yielding the main thread per depth).
- Algorithm: 9-cell pattern lookup + incremental threat potential + Alpha-Beta/PVS + iterative deepening + transposition table (Zobrist 64K) + killer moves + VCF, all pure JS, no dependencies.
- Audio: `<audio loop>` with `preload="auto"`; the end-of-game and undo sounds are two separate `<audio>` elements. While the laugh plays, background music drops from 0.34 to 0.09 and restores when it ends or a new game starts; the undo sound is only 0.86 s, so the music is not ducked for it.
- Assets: 17 background images (jpg/png/webp mixed), a ~7 MB background video, ~7 MB of music, an 80 KB end-of-game sound and an 8 KB undo sound — about 18 MB in total.

---

## License

[MIT](LICENSE)
