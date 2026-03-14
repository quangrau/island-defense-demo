# Island Defense — KiasuDash Demo

A tower-defense-style game built with **Phaser.js 3** and **React**. Part of [KiasuDash](https://kiasudash.com), a gamified learning platform for Singapore primary schools.

Blue warriors vs Red warriors battle across a pixel-art archipelago. Answer questions to spawn your troops and destroy the enemy castle!

## How to Run

Serve the folder with any static HTTP server:

```bash
# Python
python3 -m http.server 8080

# Node.js (npx)
npx serve .

# PHP
php -S localhost:8080
```

Then open `http://localhost:8080` in your browser.

## Controls (Test Panel)

- **Difficulty** — Easy / Medium / Hard (affects AI spawn rate & HP)
- **Start Battle** — Begins the round system (BGM plays automatically)
- **+1 / +3 / +5 Blue** — Spawn blue warriors (simulates correct answers)
- **Banner 1-3** — Show round transition banners
- **Victory / Defeat** — Trigger game-over banners
- **Sound buttons** — Preview BGM and SFX

## In-Game Camera

- **Pinch to zoom** (touch) or **mouse wheel**
- **Drag to pan** (touch or mouse)
- **+/- keys** to zoom

## Inspired By

- [chongdashu/phaserjs-tinyswords](https://github.com/chongdashu/phaserjs-tinyswords/tree/main) — Phaser.js Tiny Swords implementation

## Assets

- Free pixel art assets from [Tiny Swords by Pixel Frog](https://pixelfrog-assets.itch.io/tiny-swords)

## Tech Stack

- [Phaser.js 3](https://phaser.io/) — Game engine
- [React 19](https://react.dev/) — UI control panel
- [Vite](https://vite.dev/) — Build tooling
