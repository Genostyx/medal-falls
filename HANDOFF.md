# Medal Falls: handoff

A 3D arcade coin pusher that runs in the browser. You drop medals from a dropper on the back glass, a pusher shoves them toward the front edge, and medals that fall off the front are winnings. Threading a drop through a moving red ring spins a slot machine that showers extra medals and prizes onto the table.

- **Live:** https://genostyx.github.io/medal-falls/ (GitHub Pages, served from `main`, repo root)
- **Repo:** https://github.com/Genostyx/medal-falls
- **Owner:** Genostyx. The owner play-tests everything, and their feedback drives what gets built.

## How it's built

- **One file:** `index.html`, about 1,370 lines of HTML, CSS and one ES module. There is no build step, bundler or package.json.
- **Libraries** load through an import map from jsdelivr, with pinned versions:
  - `three@0.169.0` for rendering, plus `three/addons/environments/RoomEnvironment.js` for reflections.
  - `@dimforge/rapier3d-compat@0.14.0` for physics. It's WASM, embedded as base64.
- **Fonts:** Bungee (display) and Chivo Mono (UI), from Google Fonts.
- **Persistence:** `localStorage` only. There is no backend.

**Run locally.** It must be served over http, because the modules won't load from `file://`:

```
python -m http.server 8767
```

Then open http://localhost:8767/. Append `?v=N` to bust the browser cache between edits.

**Deploy.** Commit to `main` and push. Pages rebuilds in about a minute. Browsers cache the page for up to 10 minutes, so testers need a hard refresh (Ctrl+F5), or a fresh tab on phones.

## Map of `index.html`, top to bottom

1. **CSS.** A single dark arcade theme with tokens on `:root`. Responsive rules:
   - `max-width: 640px`: portrait phones.
   - `max-height: 520px`: landscape phones. This hides the footer how-to text, which also lives in the Menu.
2. **HUD markup:**
   - **Left:** medal count, `+` shop button, Collect button, free-medal timer, fever label.
   - **Right:** paid-out total, level and XP bar.
   - **Footer:** how-to text, paytable, Sound and Menu buttons.
   - Also: the big round DROP button, `#pops` (floating +N text), `#banner`, `#modal` and `#loading`.
3. **`#errbox` plus a plain (non-module) script.**
   - Shows the first few runtime errors on screen along with the browser's user agent, because phones have no console.
   - Writes crash checkpoints ("crumbs") to `localStorage['medalfalls.crumbs']`. If the last visit never fired `pagehide`, the next visit reports the stage it stopped at.
   - Exposes `window.reportError` and `window.crumb`.
4. **Module constants.** Layout, tuning, and the lite-graphics flag. See the tuning table below.
5. **Save and load:**
   - `medalfalls.v1` holds bank, won, tray and the `prog` object.
   - `medalfalls.field.v1` holds every medal's position and rotation, the pusher phase (`simTime`), the pending shower queue, queued spins, fever time left, and any spin that was mid-reel.
   - Saves run every 2 s, and on `visibilitychange` and `pagehide`.
6. **Audio.** WebAudio synth: drop, clink, win, slot ticks, reach, pay. It only starts after the first user gesture.
7. **Renderer, scene and camera.**
   - `resize()` handles portrait phones by lifting the camera overhead and fitting the table's width. It deliberately keeps the side glass and pockets in frame.
   - **Quality tiers:**
     - **Desktop:** MSAA on, resolution up to 2×, 2048 shadow map.
     - **Touch devices:** no MSAA, resolution up to 1.25×, no shadows.
     - **`?lite` or saved lite mode:** no reflections (a hemisphere light instead), no shadows, 1× resolution, metals softened.
     - **Software renderer detected:** same as lite.
     - **During play:** the only adjustment is dropping resolution to 1× when frame time stays above 1/24 s.
8. **Canvas textures:** coin face and reeded edge, the playfield (chevrons and hazard stripes at the pockets), the back-glass logo, and the bulb backdrop.
9. **Static scene:**
   - Table and payout edge light.
   - Side glass built from `SIDE_PIECES`: full-length walls plus a low open window in each side, which forms the side pockets.
   - Chrome framing, the back glass, the pusher, the dropper and its track, and the red ring (`gate`).
10. **Medal meshes.** Two `InstancedMesh`es, silver and gold. There is no cap: `ensureCoinCapacity()` rebuilds both at double size when needed. Prizes are ordinary meshes stored on `c.mesh`.
11. **Physics:**
    - `buildPhysics()` creates the fixed colliders mirroring the scene, plus a kinematic pusher.
    - `spawnCoin(x, y, z, { kind, vy, tilt, rot })`. Kinds are `coin`, `gold`, `stack` and `ball`.
    - `stepSim()` runs at a fixed 1/120 s step:
      - moves the pusher
      - damps rocking
      - checks for ring hits
      - plays clinks
      - classifies medals as won or lost
      - feeds the shower queue
12. **Payouts, slot, progression, shop, input, loop and boot**, each covered in the next section.

## Game systems

- **Dropping:**
  - The DROP button, the Space key, or a mouse click on the table each drop one medal.
  - Holding repeats a drop every 170 ms, starting after 350 ms.
  - Touching the table only aims (drag to move the dropper), so a stray tap never spends a medal.
  - Medals drop from the back glass onto the pusher's top shelf, and the back wall scrapes them off as the pusher retracts.
- **Winning and losing:**
  - A medal that falls off the front edge adds its value to `tray` and `won`.
  - Medals pushed out through the side pockets (z 0.8 to 3.8) are lost.
- **Collect:** winnings wait in `tray`, shown as the **Collect +N** button, until the player presses it, which moves them into `bank`. Level rewards, the daily bonus and shop purchases go straight to `bank`. If the bank is empty while winnings are waiting, the Collect button shakes instead of the shop opening.
- **Ring and slot:**
  - A dropped medal that passes the ring at `GATE_Y` within `0.52` of the ring's x counts as a hit. During fever the window is `0.9`.
  - A hit adds a queued spin (maximum 4). The outcome is decided when the spin starts. About half of losing spins are staged as near misses ("REACH!").
  - The slot panel is a canvas texture on the back glass, redrawn only while animating or when something changes.
- **Prizes:**
  - **Medal stack:** worth 25.
  - **Bonus ball:** worth 3 free spins.
  - At most 6 prizes can be on the table at once. Any extra prize pays out directly instead.
- **Jackpot:** progressive. It starts at 30 and grows by 1/50 of a medal per drop (a float; display it with `floor`). A 777 pays out the whole pool:
  - up to 60 medals rain onto the table
  - the rest goes to the tray
  - 3 gold medals and 2 bonus balls drop as well
- **Fever:** after a BAR or 777, the ring grows 1.7× and is easier to hit for 8 s.
- **Progression:**
  - Levels need `30 + 20 × level` XP, at 1 XP per drop. Each level pays `3 + level` medals and a free spin, and every 5th level also drops a stack.
  - Free medals trickle back at 1 per minute, up to 30.
  - The daily streak pays 20/30/40/60/80/100/200. A new game skips day 1, so it starts at exactly 100 medals.
- **Shop:** a demo only; buying just adds medals and no payment is taken.
  - Packs: 100 for $0.99, 550 for $4.99, 1,200 for $9.99, 3,000 for $19.99.
  - A one-time starter pack (500 for $1.99) appears when the player first runs dry, with a real 15-minute timer.
  - Real payments would need Stripe or app-store billing. Note the legal risk: paid spins for in-game currency can count as gambling in some places. Washington's 2018 *Kater v. Churchill Downs* ruling on social casinos is the usual reference.
- **Menu:**
  - stats and a how-to-play section
  - a Graphics Full/Lite toggle, remembered per device in `medalfalls.lite`
  - Reset game, which asks for confirmation. It sets `resetting = true` before clearing storage, because otherwise the autosave writes the old state back before the reload.

## Tuning (current values)

| What | Value |
|---|---|
| Gravity / step | −38 / 1/120 s |
| Medal | radius 0.6, half-thickness 0.045 (quarter proportions); round-cylinder collider |
| Medal damping | linear 0.4, angular 3, restitution 0, friction 0.4 |
| Pusher | front edge between z −2.8 and −0.4, period 3.2 s |
| Side pockets | z 0.8 to 3.8, height 0.9 |
| Slot odds per spin | 777 1.2% · BAR 4% · ★ 8% · ◆ 14% |
| Slot pays | 777 = jackpot + 3 gold + 2 balls · BAR 6 + stack · ★ 3 · ◆ 1 |
| Lucky drop | 1% of drops turn gold (gold worth 5) |
| Start | 100 medals |

**Economy status.** Measurements from an offline Rapier replica put the pusher alone at about 89% return (coins back per coin dropped) with open sides. Wider gutters brought it to about 71%. The layout has changed since then: the dropper moved to the back glass, and the side pockets replaced the open sides. So **current payback is unmeasured**. The owner's goal is roughly 85–90% total, so that the shop matters. The side pocket length (`POCKET_Z0` and `POCKET_Z1`) is the biggest lever.

## Hard-won lessons (don't undo these)

- **Rocking medals.** Medals leaning on each other rock forever. `stepSim` halves the tilt spin of slow medals every step, but deliberately skips any medal at a drop-off edge: the front, the side pockets, or the pusher step. Without that exception, medals hang on edges.
- **Never toggle shadows or rebuild materials mid-game.** Recompiling every material at once made phone browsers drop the WebGL context. It happened on a Google Pixel.
  - Decide quality before the first frame.
  - The context-loss handler switches the device to lite and reloads, because browsers block 3D for a site after repeated crashes.
- **No medal cap.** An earlier 400-medal cap locked the game. Keep the instanced meshes growable.
- **Refresh safety.** A spin interrupted by a refresh pays its stored result (`slot.pending`), so a refresh can't be used to re-roll.
- **Saving.** When restoring the table, put the pusher back at the saved phase before spawning medals.
- **Pages cache.** Updates take up to 10 minutes to show without a hard refresh, and "it didn't change" is usually this.
- **Encoding.** Keep `index.html` as UTF-8 (it uses ★ ◆ ♣ ▸). Some Windows tools rewrite text files in other encodings.

## Owner preferences

- Keep replies short: say what changed, not how it works.
- Don't make changes the owner didn't ask for. An offhand remark isn't a request, so ask first.
- The owner likes gambling-style mechanics, meaning random outcomes with visible spread (jackpot, reach, bonus balls) over flat rewards.
- The owner does the play-testing. Don't run long simulations or test loops unless asked. A quick load check for errors before pushing is fine.
- After each change, commit it, push it to `main`, and tell the owner to hard-refresh.
- Visual decisions the owner made:
  - dropper on the back glass
  - slot panel on the glass above the logo
  - full-length side glass with side pockets, because it has to look fair
  - thin, quarter-like medals
  - the Collect button
  - the big DROP button

## Open items and ideas

- Re-measure and tune the payback now that the layout has settled, aiming for about 85–90%.
- The red ring sits partly in front of the "M" in the logo. Nudge the logo or ring if the owner asks.
- Only tested on Chrome (desktop and Pixel) and Edge. Safari and iOS are untested.
- The shop is a demo only. Real money needs a payment provider plus a legal check.
- Nothing is shared between players: no leaderboard and no cloud save.
