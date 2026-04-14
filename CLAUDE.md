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
- **Laserman** (6 lives), **Laserwoman** (8 lives), **Lasercat** (9 lives)

## Key Technical Details
- Uses `onTap()` helper for cross-platform touch/click handling (iOS compatibility)
- Performance tiers: Low/High (controls scanlines, star particles, animations)
- Color themes: Green, Blue, Yellow
- Scanlines/vignette rendered via `.screen::before/::after` pseudo-elements (not overlay divs) to avoid iOS touch blocking
- CSS variables defined in `:root` for theming

## Layout & Centering
- All `.screen` containers use `display:flex; flex-direction:column; align-items:center; text-align:center` — every screen is centered by default
- Fixed-width elements (`.word-input-row`, `.entry-actions`) use `margin: 0 auto` to center within block parents
- Online sub-containers (`host-word-entry`, `host-role-choice`, etc.) have explicit `text-align:center`
- Always ensure new UI elements are centered — never left-align content within screens

## Laserlink Details
- **Async mode** — sender creates a word; receiver opens a URL to play it. No WebRTC, no active connection.
- **URL format**: `<origin><path>?ll=<base64url(JSON)>`. JSON payload: `{ w: WORD, n: SENDER_NAME, c: character, col: color }`.
- **URL encoding**: `b64urlEncode`/`b64urlDecode` — URI-safe base64 (`+` → `-`, `/` → `_`, no padding).
- **Link base**: Built from current `window.location.origin + pathname` — automatically routes to wbcgamez, ladybug, or GitHub Pages depending on where sender is playing.
- **Message template**: `"<SENDER> has sent you a Laserlink! <URL>"` — composed in `btn-laserlink-create` handler.
- **Sharing**: Uses `navigator.clipboard.writeText` with fallback to `document.execCommand('copy')`; shows native share sheet via `navigator.share` on supported mobile browsers.
- **Receiver flow**: On page load, `parseLaserlinkUrl()` checks for `?ll=` param. If valid, `startLaserlinkGame(info)` sets `gameState.mode = 'laserlink'`, applies sender's character/color, strips URL param (via `history.replaceState`), and jumps straight to game screen.
- **Receiver UI**: `#laserlink-sender-row` above the word display shows "LASERLINK FROM <SENDER>"; NEW GAME button renamed to PLAY AGAIN (replays same word); MAIN MENU button navigates to parent site `/` when on Railway (wbcgamez/ladybug) instead of Laserman's own main menu.
- **Receiver performance tier**: Auto-detected (or uses receiver's saved setting) — NOT taken from the sender.
- **Name persistence**: Sender's name saved in `localStorage` key `laserman_online_name` (shared with Online mode).

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

## Deployment
- **GitHub Pages**: `https://brightinfinity3.github.io/laserman/laserman.html` — auto-deploys on push to `main`
- **Railway (wbcgamez)**: `wbcgamez-production.up.railway.app/laserman/`. Deploy: copy `Laserman.html` → `wbcgamez/public/laserman/index.html`, then `railway up` from `wbcgamez` dir. Already linked.
- **Railway (ladybug)**: `ladybug.up.railway.app/laserman/`. Deploy: copy `Laserman.html` → `ladybug-gamez/public/laserman/index.html`, then `railway up` from `ladybug-gamez` dir. Already linked.
- **GitHub repo**: `https://github.com/BrightInfinity3/laserman.git` on `main` branch
- **Full deploy workflow**: After changes, push to GitHub, then copy `Laserman.html` to both `wbcgamez/public/laserman/index.html` and `ladybug-gamez/public/laserman/index.html`, then `railway up` from each directory

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
