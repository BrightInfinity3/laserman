# Laserman

## Overview
Laserman is a hangman-style word guessing game with a cyberpunk/neon aesthetic, built as a single HTML file. Players guess letters to reveal a hidden word before running out of lives. Wrong guesses trigger laser animations that hit body parts of the character.

## Architecture
- **Single file**: `Laserman.html` — contains all HTML, CSS, and JavaScript
- **No build system** — open directly in a browser
- **Canvas-based** character rendering with laser animations
- **localStorage** for saving preferences (character, color, performance tier)

## Game Modes
- **Solo**: Random word from categorized dictionary (Technology, Space, Science, Gaming, Cinema, Adventure, Nature)
- **Multiplayer (local)**: One player types a word, another guesses
- **Online**: WebRTC peer-to-peer multiplayer via PeerJS
- **Laserlink**: Async multiplayer — sender creates a word and shares a URL; receiver opens link and plays immediately. No live connection needed.

## Characters
- **Laserman** (6 lives) — head, torso, 4 limbs. 6 turrets at 30° rotation (0=upper-right … 5=upper-left). Head always hit last (turret 0 or 5).
- **Laserwoman** (6 lives) — Laserman body + **hair** (solid trapezoid behind head, drawn as a character accessory) and **flower** (5-petal + center, drawn on top of head). Hair and flower are NOT body parts — they animate during the final head-shot cascade. See "Laserwoman final combo" below.
- **Laserkid** (7 lives) — 2/3 scale of Laserman, with a **yo-yo** (new 'yoyo' body-part type) hanging from a string below one hand. Side (left/right) is chosen randomly each round via `laserkidYoyoSide`. 7 turrets at 0° rotation with turret 0 directly at the top (hits the head from above). The turret on the yo-yo's side (turret 5 for left, turret 2 for right) hits the yo-yo; arms swap upper-side turrets accordingly.
- **Lasercat** (9 lives) — Laserman body + 2 ears (triangle, no bottom) + tail (bezier). Ears render behind the head via clipping.
- **Laserbug** (10 lives) — ladybug design: `bugBody` (large circle), small `circle` head tangent to the body top, 6 `bentLine` legs (3 per side — two-segment with a knee bend), 2 `antenna` beziers from the head with tip-balls. **Spots** are hollow outlined circles drawn over the body (outline in palette color before body is etched; switches to red / gray-for-red-theme after body hit). Shot order: 8 legs+antennae randomized → body (9th) → head (10th).

### Laserwoman final combo
The head is always the final shot. Before generating the shot order, a mode A/B/C/D is chosen at random and controls what the primary laser hits and which cascade follows.

| Mode | Primary target           | Which top turret fires                   | Cascade (450ms) after primary completes          |
|------|--------------------------|------------------------------------------|--------------------------------------------------|
| A    | Head circle              | random top-left (5) or top-right (0)     | Hair fills top→bot + flower goes red             |
| B    | Flower                   | random top-left (5) or top-right (0)     | Head fills via arc backfill + hair top→bot       |
| C    | Hair — bottom-LEFT corner| top-left (5) only                        | Head fills via arc backfill + flower goes red    |
| D    | Hair — bottom-RIGHT corner| top-right (0) only                      | Head fills via arc backfill + flower goes red    |

Cascade visuals are driven by globals `lwHairAnim`, `lwFlowerAnim`, `lwHeadBackfill`. Once `lwHairAnim.fillProgress >= 1`, the hair trapezoid draws entirely in red (with red glow only), dropping the palette-green base so the halo around the hair shifts to red (or gray for the red theme).

### Turret-path constraints (Laserwoman only)
Turrets can't cross body parts (except torso — which goes through the hair from a side turret per the user's design). Allowed turret sets per part, enforced via backtracking:
- L-arm: {3, 4, 5}  (left turrets)
- R-arm: {0, 1, 2}  (right turrets)
- L-leg: {3, 4, 5, 2}  (left turrets + bottom-right)
- R-leg: {0, 1, 2, 3}  (right turrets + bottom-left)
Other characters don't enforce these constraints.

### Body-part types (drawBodyPartShape)
- `circle` — head / yo-yo ball / bug head. Traces clockwise from `part.startAngle` (set in `setupBeamFromBlink` to the turret-contact angle so etching starts at the hit point).
- `line` — torso, arms, legs. Etches from `x1/y1` (outer tip) to `x2/y2` (attachment).
- `bentLine` — Laserbug legs. Two-segment with a bend at `(bendX, bendY)`. Etches tip → knee → body.
- `solidHair` — Laserwoman hair trapezoid. Fills bottom-to-top or top-to-bottom depending on `fillDir`.
- `flower` — Laserwoman hair accessory. Petal reveal.
- `ear` — Lasercat ears (triangle minus bottom).
- `bezier` — Lasercat tail.
- `antenna` — Laserbug antennae (bezier + tip-ball on completion).
- `bugBody` — Laserbug body (large arc with `part.startAngle`).
- `yoyo` — Laserkid yo-yo (ball + string). Two-phase etch: 0→0.5 ball arc; 0.5→1 string.

All circle-type parts (`circle`, `bugBody`, `yoyo`) use **closest-point-on-circle** targeting in `setupBeamFromBlink`: the beam hits the silhouette point nearest the firing turret, and `part.startAngle` is set so the red trace arc begins there.

## Key Technical Details
- Uses `onTap()` helper for cross-platform touch/click handling (iOS compatibility)
- Performance tiers: Low/High (controls scanlines, star particles, animations)
- **Color themes (6)**: Yellow, Green (default), Blue, Orange, Red, Purple. Options layout puts red top-left, yellow top-middle, blue top-right, orange bottom-left, green bottom-middle, purple bottom-right.
- **Red theme** — turrets start in the palette (red) color and FLASH to gray during the firing animation (the existing `flash` phase alternates `COL.red` [= gray in red theme] and `COL.green` [= palette red], and the `firing`/`red` turret states use `COL.red` = gray). Laser beam, etching, hit body parts are all drawn gray. Implemented via palette overrides on `COL.red`/`bodyRed`/`redGlow`/`laserCore`/`laserGlow`/`pink`/`traceBright`/`traceCore` plus `grayLaserMode` flag. `COL.turretGreen` always matches palette primary. Fire-up target color uses `[85,85,85]` instead of `[255,0,68]`.
- **Purple theme** — failure message renders gray (`.msg-line.lose.gray-fail`) via `grayFailMode` flag.
- **Constant trace colors** — `COL.traceBright` (replaces hardcoded `#ff6677`) and `COL.traceCore` (replaces `#ff8899`) so theme overrides apply consistently.
- Scanlines/vignette rendered via `.screen::before/::after` pseudo-elements (not overlay divs) to avoid iOS touch blocking
- CSS variables defined in `:root` for theming

## Layout & Centering
- All `.screen` containers use `display:flex; flex-direction:column; align-items:center; text-align:center` — every screen is centered by default
- Fixed-width elements (`.word-input-row`, `.entry-actions`) use `margin: 0 auto` to center within block parents
- Online sub-containers (`host-word-entry`, `host-role-choice`, etc.) have explicit `text-align:center`
- Always ensure new UI elements are centered — never left-align content within screens

## Dictionaries
- **`laserman-solo-dictionary.xlsx`** — curated word list used by Solo mode (categorized by Technology / Space / Science / Gaming / Cinema / Adventure / Nature). Mirrors the `WORDS` object hard-coded in `Laserman.html`.
- **`laserman-master-dictionary.xlsx`** — master English word list (~370k words, 2\u201320 chars, all inflections). Sheets: About / All Words / By First Letter / Stats. Source: dwyl/english-words (words_alpha.txt).
- **`laserman-master-dictionary.txt`** — plain-text mirror of the master Excel (one word per line, UPPERCASE). The game fetches this file at runtime when the "USE LASERMAN DICTIONARY" toggle is on. Must be deployed alongside `laserman.html` (GitHub Pages, wbcgamez/public/laserman/, ladybug-gamez/public/laserman/).
- **Dictionary toggle** — `data-dict-toggle` checkboxes live on Multiplayer / Online host / Online joiner / Laserlink compose screens. State is shared via `useLasermanDict` (localStorage key `laserman_use_dict`). When on, word submissions are validated against the master dictionary; invalid words show `"WORD" NOT IN LASERMAN DICTIONARY` and block submission.
- **Dictionary loading** — lazy: `loadLasermanDictionary()` fetches on first toggle-enable or first submission. De-duped via `dictionaryLoadPromise`. Cached as `Set<string>` in memory. Fetch uses `cache: 'force-cache'` for browser caching. On fetch failure the validator returns `true` (don't block users).

## Laserlink Details
- **Async mode** — sender creates a word; receiver opens a URL to play it. No WebRTC, no active connection.
- **URL format**: `<origin><path>?ll=<base64url(payload)>`. Payload is pipe-separated: `WORD|NAME|CHAR_INDEX|COLOR_INDEX` (e.g., `TEST|GMK|0|0`). Character indices (in `LL_CHARS`): 0=laserman, 1=laserwoman, 2=lasercat, 3=laserkid, 4=laserbug. Color indices (in `LL_COLORS`): 0=green, 1=blue, 2=yellow. (Only the original three colors are encoded; orange/red/purple will appear on the receiver as green — they were added after the URL format was fixed and are intentionally not worth bumping the URL schema for.) Falls back to JSON parse for backward compat.
- **URL shortener**: `shortenUrl(url)` fetches `https://is.gd/create.php?format=json&url=…` with a 4-second timeout; on success the sender's textarea gets the short URL (e.g. `https://is.gd/hVAZPL`). On failure (CORS, offline, timeout) it silently falls back to the full URL. The share textarea auto-sizes height to content, and the share container's width is measured against a hidden mirror span so the box ends exactly where the text ends (capped at 92vw).
- **URL encoding**: `b64urlEncode`/`b64urlDecode` — URI-safe base64 (`+` → `-`, `/` → `_`, no padding).
- **Link base**: Built from current `window.location.origin + pathname` — automatically routes to wbcgamez, ladybug, or GitHub Pages depending on where sender is playing.
- **Message template**: `"<SENDER> has sent you a Laserlink! <URL>"` — composed in `btn-laserlink-create` handler.
- **Sharing**: Uses `navigator.clipboard.writeText` with fallback to `document.execCommand('copy')`; shows native share sheet via `navigator.share` on supported mobile browsers.
- **Receiver flow**: On page load, `parseLaserlinkUrl()` checks for `?ll=` param. If valid, `startLaserlinkGame(info)` sets `gameState.mode = 'laserlink'`, applies sender's character/color, strips URL param (via `history.replaceState`), and jumps straight to game screen.
- **Receiver UI**: `#laserlink-sender-row` above the word display shows "LASERLINK FROM <SENDER>"; NEW GAME button renamed to CREATE LINK (amber, routes to laserlink compose screen); MAIN MENU button navigates to Laserman's main menu.
- **Receiver performance tier**: Auto-detected (or uses receiver's saved setting) — NOT taken from the sender.
- **Name persistence**: Sender's name saved in `localStorage` key `laserman_online_name` (shared with Online mode).

## Laserbug win/lose spot animation
On win/lose, the still-green turrets (on win) or all red turrets (on lose) aim at each spot one at a time — like Lasercat's face-feature sweep. Each spot fills from center outward over ~260ms, then the animation moves to the next spot. Driven by `laserbugSpotAnim = {startTime, color, perSpot, spotCount, currentSpot, currentProgress}` and rendered via `drawLaserbugSpots` (fill) + `drawLaserbugSpotBeams` (beams from active turrets to the currently-filling spot).

Spots are **hollow by default** (no fill) — only the border is visible during normal play. The border uses the palette color before the body is etched, and switches to `COL.bodyRed` (red normally, gray for red theme) as a damage cue once `bodyPartStates[1] === 'red'`.

## Online Multiplayer Details
- **Signaling**: PeerJS cloud server (`0.peerjs.com`) with `peerjs@1.5.4` CDN
- **ICE/TURN**: Metered.ca TURN servers (free 20GB/month) — credentials are static in `ICE_SERVERS` constant. Metered account: `mkgamez.metered.live`. Manage credentials at metered.ca dashboard under TURN Server > TURN Credentials
- **Room codes**: 3-character alphanumeric (no I/O/0/1 to avoid confusion), prefixed with `laserman-` for PeerJS peer IDs
- **Heartbeat**: 5-second ping/pong interval, 15-second timeout to detect dead connections (e.g., phone screen lock)
- **`visibilitychange`** listener re-pings on wake to quickly detect stale connections
- **Rematch flow**: Player clicks NEW GAME → other player sees "PLAY AGAIN?" popup → both must agree before transitioning. Mutual NEW GAME clicks treated as auto-accept
- **Confirm leave**: MAIN MENU button in online mode shows confirmation popup before disconnecting
- **Disconnect handling**: Toast notification with opponent's name, kicks both players to main menu
- **Animation sync**: `onlineWaitingForPeerAnim` flag + `anim-done` messages keep both players in sync regardless of performance tier
- **Color/character sync**: Word-setter's character and color choices are sent with `start-game` message and applied on both sides
- **Online OPTIONS panels**: Show character and color pickers only (no performance picker — that's main menu only)
- **`onlineRoundCount`**: Tracks rounds; "YOUR TURN TO SET THE WORD!" message hidden on first round

## Mode titles (MULTIPLAYER / ONLINE / LASERLINK / OPTIONS)
Each submenu shows a single bold mode title at the same size as the home-page LASERMAN (3.2rem desktop / 2.2rem / 1.8rem responsive). White text with a pulsing colored glow matching the mode — implemented via `.game-title.mode-label` + color-class modifiers `.ml-green`, `.ml-purple`, `.ml-amber`, `.ml-white`, each with their own `titlePulse{Green,Purple,Amber,White}` keyframes. LASERMAN subtitle was removed from these screens.

## Dictionary indicator on game screen
`#dict-indicator-row` sits in the same slot as the solo category row and shows `LASERMAN DICTIONARY WORD? [check/x]`. `updateDictIndicator(word)` is called from `startGame` — loads the dictionary (lazy, cached), then shows a cyan ✓ if the word is in the dictionary, red ✕ if not, and `...` while loading. Hides itself entirely if the dictionary fetch fails. The toggle and indicator share a `.dict-box` class — cyan ✓ or red ✕ square, with a `data-dict-toggle` checkbox wired via `syncDictToggles()` / `setUseLasermanDict()`. `useLasermanDict` defaults to `true` for first-time users.

## Deployment
- **GitHub Pages**: `https://brightinfinity3.github.io/laserman/laserman.html` — auto-deploys on push to `main`
- **Railway (wbcgamez)**: `wbcgamez-production.up.railway.app/laserman/`. Deploy: copy `Laserman.html` → `wbcgamez/public/laserman/index.html`, then `railway up` from `wbcgamez` dir. Already linked.
- **Railway (ladybug)**: `ladybug.up.railway.app/laserman/`. Deploy: copy `Laserman.html` → `ladybug-gamez/public/laserman/index.html`, then `railway up` from `ladybug-gamez` dir. Already linked.
- **GitHub repo**: `https://github.com/BrightInfinity3/laserman.git` on `main` branch
- **Full deploy workflow**: After changes, push to GitHub, then copy `Laserman.html` to both `wbcgamez/public/laserman/index.html` and `ladybug-gamez/public/laserman/index.html`, then `railway up` from each directory
- **Selective deploys (current default)**: The user has asked to ship experimental character work (new characters, yo-yo, color themes, etc.) to **ladybug only** and keep wbcgamez stable. Each v4.18+ ladybug-only update skips the wbcgamez copy/upload step. Ask before reverting to full-platform deploys.

## Responsive Design
- Game buttons (`.game-btn`) use `clamp()` for font-size, padding, letter-spacing with `max-width: 45vw` and `overflow: hidden; text-overflow: ellipsis`
- Entry-action buttons (`.entry-actions .menu-btn`) also use `clamp()` to prevent text overflow on small screens
- Media queries at 600px and 400px breakpoints for `.menu-btn` and `.char-option` sizing
- Touch devices get larger tap targets via `@media (pointer: coarse)`
- No-cache meta tags added to prevent stale code on GitHub Pages

## Conventions
- All words in the dictionary are UPPERCASE
- Screen navigation via `showScreen(id)` function
- Game state tracked in `gameState` object
- Toast notifications via `showToast(text, onDone)` for transient messages
- Confirmation modals via `showConfirmLeave(onConfirm)` / `hideConfirmLeave()`
- `cleanupPeer()` handles full connection teardown (stops heartbeat, closes connection, destroys peer)
