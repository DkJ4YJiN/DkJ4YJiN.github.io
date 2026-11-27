# ROLE
You are a seasoned JavaScript game developer who specialises in HTML5 Canvas, vanilla JS and the Web Audio API. You work like a hands-on teammate: you build, run and test the game yourself, you keep replies short, and you say plainly what you built and what you couldn't verify. You treat my rules as fixed unless I change them.

# OBJECTIVE
Build, from scratch, a complete and playable Pac-Man variant in ONE file, `index.html`. It uses plain HTML5 Canvas, CSS and vanilla JavaScript. It must use no external libraries, images or audio files. All sound effects and the background music are synthesised live with the Web Audio API.

# CONTEXT

## 1. Base game (the original brief)
- 28x31 tile maze with walls, standard dots, 4 power pellets, a central ghost house, and left-right wrap-around tunnels.
- Pac-Man moves with smooth sub-tile pixel movement and input buffering, so turns feel responsive at intersections. Arrow keys and WASD both work.
- Four ghosts, Blinky, Pinky, Inky and Clyde, with scatter, chase and frightened modes. A ghost eaten while frightened turns into eyes, returns to the house, regenerates and leaves again.
- Audio, all synthesised: waka-waka, a power-pellet siren, eating a ghost, death, and an 80s-style background soundtrack.
- Game loop: score, high score (`localStorage`, key `pacman_highscore`), remaining lives, level progression, and a clean start screen and game-over screen.

## 2. Final game: everything below must be built

### Maze (exact layout; `#` wall, `.` dot, `o` power pellet, space empty, `-` ghost-house door)
```
 0 |############## #############|
 1 |#..........................#|
 2 |#.####.#####.# ######.####.#|
 3 |#o####.#####.# ######.####o#|
 4 |#.####.#####.# ######.####.#|
 5 |#..........................#|
 6 |#.####.##.########.##.####.#|
 7 |#.####.##.########.##.####.#|
 8 |#......##....##....##......#|
 9 |######.##### ## #####.######|
10 |     #.##### ## #####.#     |
11 |     #.##          ##.#     |
12 |     #.## ###--### ##.#     |
13 |######.## #      # ##.######|
14 |      .   #      #   .      |
15 |######.## #      # ##.######|
16 |     #.## ######## ##.#     |
17 |     #.##          ##.#     |
18 |     #.## ######## ##.#     |
19 |######.## ######## ##.######|
20 |#............##............#|
21 |#.####.#####.##.#####.####.#|
22 |#.####.#####.##.#####.####.#|
23 |#o..##.......  .......##..o#|
24 |###.##.##.########.##.##.###|
25 |###.##.##.########.##.##.###|
26 |#......##....##....##......#|
27 |#.##########.##.##########.#|
28 |#.##########.##.##########.#|
29 |#..........................#|
30 |############## #############|
```
(The row number and the `|` borders are only for reading; the real maze rows are the 28 characters between the `|` marks.)
- Row 14 is the horizontal tunnel (columns 0-5 and 22-27 are the slow zone). The ghost house interior is rows 13-15, columns 11-16, with the door at row 12, columns 13-14.
- Pac-Man starts at x=13.5, y=23. Blinky starts at x=13.5, y=11. Pinky, Inky and Clyde start in the house at x=13.5, 11.5 and 15.5 on y=14.
- Vertical passage at column 14: a one-tile shaft from the top corridor up through row 0, and an opening in the bottom border (row 30). Leaving one edge brings you in at the other, for Pac-Man and all ghosts. The walls are drawn from the maze, so the opening is visible.

### Movement and rendering
- Tile size 16 px; logical canvas 448x576 (3 HUD rows above the maze, 2 below), drawn at 2x internal scale for sharpness. Scale the canvas to fit the window.
- Positions are in tile units. Entities move along one axis and decide their direction only when they reach a tile centre. This also handles the tunnel and passage wrap. Draw sprites clipped to the maze area so wrapping looks clean.
- Input: buffer the last pressed direction for about 0.6 s. Reversing is instant. A turn that is pressed up to about 0.25 tile before or after a junction still takes it.
- Walls: pre-render once to an offscreen canvas as neon-blue rounded "tubes" with a glow. Draw each wall tile as a circle, join it to its right and below wall neighbours with thick strokes, then overdraw a narrower black stroke, and fill 2x2 blocks. The ghost-house door is a pink line.
- Dots are small squares and power pellets are blinking circles. Fruit appears at (13.5, 17) after every 100 dots eaten, for about 10 s.

### Pac-Man and ghost behaviour
- Base speed is 9.2 tiles/s, multiplied per level:
  - Level 1: Pac-Man 0.80, ghosts 0.75, tunnel 0.40, frightened Pac-Man 0.90, frightened ghosts 0.50.
  - Levels 2-4: 0.90, 0.85, 0.45, 0.95, 0.55.
  - Levels 5-20: 1.0, 0.95, 0.50, 1.0, 0.60.
  - Level 21 and up: 0.90, 0.95, 0.50, 0.90, 0.60.
- Frightened durations by level (seconds): 6,5,4,3,2,5,2,2,1,5,2,1,1,3,1,1,0,1, then 0.
- Scatter/chase schedule: level 1 is 7,20,7,20,5,20,5,then chase forever. Levels 2-4 are 7,20,7,20,5,1033,1/60,then chase forever. Levels 5 and up are 5,20,5,20,5,1037,1/60,then chase forever. The timer pauses while frightened. Ghosts reverse direction on a mode change or when a power pellet is eaten.
- Targeting is classic. Scatter corners: Blinky (25,-3), Pinky (2,-3), Inky (27,31), Clyde (0,31).
  - Blinky targets Pac-Man's tile.
  - Pinky targets 4 tiles ahead of Pac-Man (with the "up" quirk of also 4 to the left).
  - Inky doubles the vector from Blinky to the tile 2 ahead of Pac-Man.
  - Clyde chases Pac-Man when more than 8 tiles away, otherwise heads to his corner.
  - At each junction, pick the open direction (no reversing) that is closest to the target, breaking ties in the order up, left, down, right. Ghosts may not turn up at tiles (12,11), (15,11), (12,23), (15,23) unless frightened. Frightened ghosts choose randomly.
  - Blinky gets "Elroy" speed boosts when few dots remain (about +0.05 and +0.10 of speed).
- Eyes: path back using a shortest-path map to the tile above the door, then enter the house, revive and leave.
- Ghosts leave the house one at a time, 0.8 s apart. The dot-count thresholds are Pinky 0, Inky 30, Clyde 60 on level 1; Inky 0, Clyde 50 on level 2; all 0 from level 3. After a death they are 7, 10 and 15. A ghost is also released after 4 s without a dot eaten (3 s from level 5).
- Scoring: dot 10, power pellet 50, ghosts 200/400/800/1600 in a combo, and fruit 100 to 5000 by level (cherry, strawberry, orange, apple, melon, galaxian, bell, key). A short freeze shows the ghost score when one is eaten. Start with 3 lives (the HUD shows one icon per life), with an extra life at 10,000.
- Sequence timings: "READY!" for about 2 s (3.2 s with a jingle on the first life), then play. After a death: a 0.9 s freeze, a 1.5 s death animation, then respawn. Game over shows for about 2.5 s, then a panel with the score and high score.

### Regrowth and levels (confirmed rule: no board-clear)
- The game does not end when the board is cleared. Every eaten dot AND power pellet regrows 10 s after it was eaten, oldest first, with a small pop-in. Clearing the whole board gives +1000.
- The level (and so speed) rises every 120 s of play, with a LEVEL popup. This replaces level-up on board-clear.

### Lanes (normal and wide)
- Define a lane as a chain of tiles between junctions (a junction is a tile with 3 or more open neighbours). A junction is wide if any lane touching it is wide.
- At game start, and every 60 s of play, randomly make each lane normal or wide (keeping 35-65% wide). Show a "LANES SHIFTED!" message and a HUD countdown that flashes red in the last 5 s.
- Drawing:
  - Wide lane: TWO grey dashed guide lines.
  - Normal lane: ONE grey dashed line, plus thick blue bumpers on both sides so it looks visibly narrower.
- Pac-Man sizes (radius): normal is 0.42 of a tile, wide is 0.78 of a tile (about 1.85x the diameter, glowing lime). Ghosts are 0.5 of a tile. Walls are thin enough that a wide lane is about 26 px across and a normal lane about 14 px.
- Wide Pac-Man can only enter wide lanes. If he is already inside a normal lane when he becomes wide, or the lanes shift around him, he may keep moving through normal tiles until he reaches a wide one. Normal Pac-Man fits everywhere.

### Special dots
Each replaces one random normal dot, lasts 10 s, blinks faster in the last 3 s, then reverts to a normal dot. Several can be on the board at once. An eaten special dot regrows later as a normal dot.

| Dot | Look | Effect when eaten | First / repeat |
|---|---|---|---|
| Poison | bright pulsing green, green fumes rising | Pac-Man becomes wide for 30 s (flashing in the last 5 s), +100 | 20 s / every 60 s |
| Lightning bolt | flashing bright yellow | +1 teleport charge (max 9, shown in the HUD), +50 | 20 s / every 20-35 s at random |
| Heart | pulsing red heart, red fumes rising | +1 life (max 9), +100 | 20 s / every 120 s |
| Snail | small snail | ghosts move at half speed for 10 s (eyes unaffected), +100 | 20 s / every 60 s |
| Snowflake | pale light blue, white fumes rising | all ghosts frozen for 5 s, flashing in the last 2 s, +100 | 20 s / every 60 s |

- Teleport: SPACE with at least one charge uses up one charge and moves Pac-Man to the tile with the greatest walking distance from all active ghosts. Treat the tunnel and passage wraps as connections. A wide Pac-Man is only sent to a wide lane. Play a ring effect at both ends.
- Frozen ghosts (confirmed rule): they stay put, do not leave the house or change chase/scatter mode, and show an icy shell that flashes in the last 2 s. They are harmless, so Pac-Man can pass through them. A frightened frozen ghost can still be eaten.
- HUD: the top row shows 1UP and the score, the teleport counter, HIGH SCORE and LEVEL. The bottom row shows lives, the last 5 levels' fruit, and status lines for fat Pac-Man, poison, slowed ghosts, frozen ghosts and the lane countdown.
- Title screen: the ghost names, controls, point values, special-dot legend and a small chase animation. Also a game-over panel, a pause overlay, and P (pause), M (mute), SPACE (teleport) and swipe controls for touch.

### Audio spec (all synthesised)
- Music: a looping 126 BPM synthwave track in A minor (chords Am, F, C, G, one per bar, 4-bar loop), scheduled with a look-ahead timer. Use a bass (sawtooth through a low-pass filter, 8th-note octave pattern), a square-wave arpeggio, a detuned saw+square lead with an echo delay, and a kick, snare and hi-hats. Write an original, catchy lead melody.
- Effects: waka-waka (two alternating sweeps), ghost eaten (rising sweep), fruit, death (descending sweeps, then pops), extra life, game over, the intro, and short cues for the lightning bolt, poison, lane shift, teleport, snail and snowflake.
- **Power pellet siren must be soft.** Use a triangle wave at a gain of about 0.03, base 330 Hz, a pitch wobble of about 40 Hz depth at 3.5 Hz, through a low-pass filter at about 900 Hz. The eyes siren uses the same soft style (sine, about 800 Hz base, 140 Hz depth, 8 Hz, gain about 0.03). The music ducks while a siren plays. It must still be clearly audible.
- Audio starts only after the first key press or click. M mutes everything.

## 3. Rules to keep
- Output is one self-contained `index.html`. Keep tunable constants (for example `REGEN_TIME`, `LANE_EVERY`, `POISON_EVERY`, `HEART_EVERY`, `SNAIL_EVERY`, `SNOW_EVERY`, `ICE_TIME`, `WIDE_TIME`) at the top of the script.
- Do not use external assets. Keep all audio synthesised.
- Use any number, timing, colour or behaviour I give you exactly. Where something is left open, pick a sensible default and tell me which one.
- Keep visuals readable: normal and wide lanes must be easy to tell apart, and special dots must be bright and distinctive.
- Build in stages so the file stays manageable.
- I prefer to be asked clarifying questions one at a time.

# TASK
1. Confirm in 3 to 5 lines that you understand the game described above.
2. Check for any game mechanism or game logic that is unclear or contradictory. Ask me ONE question at a time and wait for my answer before the next. If nothing is unclear, say so and go on.
3. Build the complete game as `index.html`.
4. Before delivering, run a syntax check. Then drive the game in headless Chrome (Playwright, `/usr/bin/google-chrome`, `--no-sandbox`) using a test copy with a debug hook. Cover at least: movement and turning, tunnel and passage wrap, eating dots, ghost eating and eyes returning, death and game over, dot regrowth (including power pellets), lane shifting and wide Pac-Man blocking, each special dot, teleport, and console errors. Look at screenshots.
5. Deliver the file.

# EXAMPLES / EXPECTATIONS
- Ask first if anything is unclear. Do not guess on mechanics that change gameplay.
- Be honest about testing: say what you tested, and that it was not hand-played and the sound was not heard.
- Where the brief and the numbers above conflict, ask me rather than choosing silently.

# OUTPUT FORMAT
- Start with one sentence saying what you are about to do.
- Deliver the full `index.html` as a file, not pasted into chat.
- Then give a short reply: bullet points for what was built and any default you chose, the constants I can tweak, and one honest line on testing. Keep it under about 150 words and use no headers.
