# khushali-birthday-card.html — code layout

One self-contained HTML file. No build step, no external requests except the
browser's own mic permission prompt. Everything — markup, CSS, JS, and all
7 images — lives in this single file, which is why it's ~1.5MB.

The file has three parts, in order: `<head>` (meta + one `<style>` block),
`<body>` (the markup), and one `<script>` block at the end. Line numbers
below are approximate (they'll drift a little if you edit), so the
section comments and element IDs are the reliable way to find things —
search the file for them directly.

```
<head>
  <style> ... </style>          all CSS, ~300 lines
</head>
<body>
  <div id="stage">              full-viewport flex wrapper, centers everything
    <div id="stack">            CSS-grid overlay: cover and spread share one spot
      <div id="cover-wrap"> ... </div>
      <div id="spread"> ... </div>
    </div>
  </div>
  <script> ... </script>        all JS, ~260 lines
</body>
```

## CSS (`<style>`, top of file)

Organized under a few comment headers you can search for:

- `:root` — color variables (`--kraft`, `--maroon`, `--cream`, etc.)
- `#stage`, `#stack` — the sizing/centering system (see below)
- `/* ---------- COVER ---------- */` — the closed-cover card and its
  hinge-opening animation (`#cover`, `.cover-half`, `.cover-left/right`)
- `/* ---------- SPREAD ---------- */` — the two-page-open view
  (`#spread`, `#close-btn`, `#pages`, `.page-col`, `.page`)
- `/* ---- Letter page ---- */` — envelope/letter tap interaction
  (`.envelope-wrap`, `.envelope-img`, `.letter-img`)
- `/* ---- Candles page ---- */` — cake, flames, wish message, confetti,
  mic controls (`.cake-wrap`, `.flame`, `#wish-overlay`, `#blow-controls`)
- `/* ---- Responsive ---- */` — the one media query, `max-width: 700px`,
  that switches the spread from side-by-side to stacked on phones

### The sizing system (`--page-w`)

Everything — the closed cover **and** each open page — shares one width,
set once as a CSS custom property on `#stack`:

```css
--page-w: min(46vw, calc(66vh * 0.9229), 740px);
```

That's three limits at once: never wider than 46% of the viewport (so two
pages fit side by side with room to spare), never taller than 66% of the
viewport height (so it never overflows a short laptop screen — the 0.9229
factor is the images' width÷height ratio, 850÷921, used to convert a
height budget into a width), and never bigger than 740px regardless of
screen size. `.page` and `#cover-wrap` both just read `width: var(--page-w)`
and let `aspect-ratio: 850/921` compute the height. The mobile media query
overrides the variable to `min(84vw, 420px)` instead, since phones don't
need the two-pages-side-by-side headroom.

`#stack` uses CSS Grid with both `#cover-wrap` and `#spread` placed in the
same cell (`grid-area: 1/1`) so one fades into the other in place instead
of the page reflowing. Only one of the two is ever actually in the
document flow at a time — see "layout collapse" in the JS section — the
other is `display:none` so it can't inflate the shared cell's height.

## HTML (`<body>`)

```
#cover-wrap
  #cover                        the closed card, split into two halves
    .cover-half.cover-left      each half shows one side of the cover image
    .cover-half.cover-right     via background-position tricks + rotateY
  #cover-hint                   "tap to open" text

#spread
  #close-btn                    "back to cover"
  #pages
    #page-left-col
      #page-left (.page)
        #envelope-wrap
          .envelope-img         always-visible envelope photo
          .letter-img           full letter, hidden until tapped
      .page-caption             "tap the letter to read it"
    #page-right-col
      #page-right (.page)
        #cake-wrap
          .cake-img              cake with candles erased (blank tops)
          #flame1 / #flame2 / #flame3   the 3 cut-out flame PNGs, absolutely
                                         positioned by % over the blank candles
          #wish-overlay
            #confetti-canvas
            #wish-message → #relight-btn
      #blow-controls
        #mic-meter → #mic-meter-fill
        #mic-status
        #enable-mic-btn, #hold-blow-btn
        .note                    HTTPS/hosting explainer text
```

### The 7 embedded images

All `<img src="data:image/...;base64,...">`, inline, no separate files:

| Image | Used as |
|---|---|
| `cover.jpg` | both cover halves (same image, each half shows one side) |
| `envelope.jpg` | the closed/peeking-letter state on the left page |
| `letter.jpg` | the full readable letter, shown when tapped |
| `cake.jpg` | the cake art with the candle flames erased/patched out |
| `flame1.png`, `flame2.png`, `flame3.png` | the 3 flames, cut out from the
original cake art as separate transparent PNGs |

The three flame `<img>` tags have inline `style="left:...%; top:...%;
width:...%"` — those percentages were measured against the *original*
cake artwork's pixel coordinates, so they line up with the blank candle
tops on `cake.jpg` regardless of how large the card renders.

## JS (`<script>`, bottom of file)

One IIFE, four informal sections (each has a `// ----------` comment):

**Cover open/close** — `openCard()` / `closeCard()`. Beyond toggling the
`is-open`/`visible` classes that drive the CSS transitions, these two
functions also flip `display` between `none` and `''` on whichever of
`#cover-wrap` / `#spread` is inactive, timed to happen only after that
element has already faded out. This is the "layout collapse" mentioned
above — without it, the hidden panel would still occupy space in the
shared grid cell and throw off centering.

**Letter/envelope** — `toggleLetter()` just flips one class,
`.is-reading`, on `#envelope-wrap`; the CSS handles the lift-forward/
tuck-back animation.

**Candle blow-out** — the state machine lives in `frame()`, a
`requestAnimationFrame` loop that runs continuously. Key constants near
the top of this section:

```js
BLOW_THRESHOLD = 0.16   // mic/hold level (0–1) counted as "blowing"
BLOW_MS_NEEDED = 500    // how many ms of sustained blowing puts it out
HOLD_LEVEL     = 0.5    // simulated level while the hold-button/spacebar is held
```

`currentLevel` (0–1) is the single shared value both input paths write
to: `pollMic()` sets it from the Web Audio `AnalyserNode` (averaging the
lowest ~12% of frequency bins, since breath noise is low-frequency-heavy)
whenever the hold button isn't overriding it; `startHold()`/`endHold()`
(mouse, touch, and spacebar all wired to the same two functions) set it
directly. `frame()` reads whichever value is current, updates the meter
width, bends the flames (`rotate()`, alternating direction, amplitude
tied to `currentLevel`), and accumulates `blowMs` — once that crosses
`BLOW_MS_NEEDED` it calls `extinguish()`.

`extinguish()` adds `.is-out` to the three flames (CSS fades their
opacity) and shows `#wish-overlay`. `relight()` reverses all of it and
calls `stopConfetti()`.

**Confetti** — plain 2D canvas, no library. `startConfetti()` seeds ~90
particles and runs its own short-lived `requestAnimationFrame` loop
(`step()`) for about 4 seconds, then clears the canvas.

## Making common edits

- **Text on the letter/cover**: these are baked into the JPEGs, not real
  text — there's no HTML text to edit. Changing them means re-exporting
  the source images and re-running the base64 embed step (ask me, since
  I built this by editing images directly, not just markup).
- **Colors**: all in the `:root` block at the very top of `<style>`.
- **Card size**: the `--page-w` line on `#stack` (see above).
- **Blow sensitivity/duration**: `BLOW_THRESHOLD` / `BLOW_MS_NEEDED` in
  the script, described above.
- **Wish message wording, button labels, captions**: plain text in the
  `<body>` markup — safe to edit directly.

## Hosting note

The candle-blowing mic feature requires a secure context (`https://` or
`localhost`) — it's currently live on GitHub Pages, which satisfies that.
Opening the file directly from disk (`file://`) will show the hold-to-blow
fallback instead, which is expected, not a bug.
