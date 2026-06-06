# PROMPTER — Teleprompter Web App

## Project Overview
A single-page teleprompter app hosted on GitHub Pages at `philhub1234.github.io/teleprompter`. Designed for pasting scripts on a laptop and reading them on a phone (or vice versa). Rich text (bold, italic, underline, line breaks) is preserved from Notion/Pages paste.

## Repo
- **GitHub**: https://github.com/philhub1234/teleprompter
- **Live URL**: https://philhub1234.github.io/teleprompter/
- **Hosting**: GitHub Pages (static, `main` branch)

## Architecture

### Single-file HTML app (`index.html`)
Everything lives in one file — HTML, CSS, JS. No build step, no bundler. This is intentional: GitHub Pages serves it directly and it stays simple to iterate on.

### Layout
- **Sidebar (left)**: Settings only — font size, line height, speed, margins, mirror/flip, text color, formatting toolbar (B/I/U). Collapsible via a circular chevron button on the sidebar edge. **Desktop only (>768px).**
- **Mobile top bar (≤768px)**: Replaces the sidebar on narrow viewports. A slim 48px bar with: a ▶/■ play-stop button (left), H-flip/V-flip/focus-line icon buttons (centre), and a ∨ chevron (right) that opens a compact dropdown settings panel directly below the bar. During playback the bar fades out after 3 seconds; tapping the screen brings it back.
- **Main stage (right/full)**: This IS the script editor AND the teleprompter display. In edit mode, it's a contenteditable area where you paste/type your script. When you hit Start, the same content scrolls as the teleprompter.

### PIN protection
- A PIN overlay appears on page load. PIN is `554433`, stored as a SHA-256 hash in the JS.
- Once unlocked, auth persists in `localStorage` so the user isn't asked again on the same device/browser.

### Cross-device script sharing (Firebase Firestore)
The old approach encoded the entire script as base64 in the URL query string. This breaks on long scripts (URL too long for browsers). The new approach:

1. User writes/pastes script on laptop
2. Clicks "Share" → script + settings are saved to Firebase Firestore
3. Gets a short URL like `https://philhub1234.github.io/teleprompter/#s/abc123`
4. Opens that URL on phone → script loads automatically

**Firebase setup** (already configured):
- Project: `teleprompter-app-66adb` on Firebase console
- Firestore is live and connected — config is already in `index.html`
- No authentication required — scripts are stored with random IDs (unguessable)

**Firestore rules** (set in Firebase Console → Firestore → Rules):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /scripts/{scriptId} {
      allow read, write: if true;
    }
  }
}
```

**Firestore TTL policy**: Set up in Firebase Console → Firestore → Time-to-live. Create a policy on the `scripts` collection using the `expiresAt` field. Scripts auto-delete after 24 hours.

**Firestore schema:**
```
Collection: scripts
Document ID: auto-generated (8-char random ID)
Fields:
  - html: string (the rich text HTML content)
  - settings: map (fontSize, lineHeight, speed, margin, flipH, flipV, textColor)
  - createdAt: timestamp
  - expiresAt: timestamp (24h from creation, used by TTL policy)
```

### Rich text handling
- The editor uses `contenteditable="true"` with `document.execCommand` for B/I/U
- Paste handler strips unwanted styles/classes but preserves `<b>`, `<strong>`, `<i>`, `<em>`, `<u>`, `<br>`, `<div>`, `<p>` tags
- This is critical — don't switch to a textarea or plain text input

### Key UI states
1. **Edit mode**: Sidebar open (desktop) / top bar visible (mobile), stage shows editable script area with placeholder text
2. **Playing**: Sidebar collapses, stage scrolls the script, floating controls appear (speed ±, play/pause, restart, stop). On mobile the top bar fades out after 3s; tap screen to show it again. Tap ■ to stop.
3. **Shared/loaded**: Script loads from Firebase via URL hash, goes straight to edit mode with content populated

## Tech Stack
- Vanilla HTML/CSS/JS (no framework)
- Google Fonts: DM Mono, Bebas Neue, DM Sans
- Firebase JS SDK (loaded from CDN) for Firestore
- GitHub Pages for hosting

## Development Workflow
1. Edit `index.html` locally
2. Test by opening in browser (file:// works for everything except Firebase calls)
3. For Firebase testing, use a local server: `python3 -m http.server 8000`
4. For mobile testing on the same Wi-Fi network: `python3 -m http.server 8765` then open `http://<your-mac-ip>:8765` on the phone. Get the IP with `ipconfig getifaddr en0`.
5. Push to `main` branch → GitHub Pages auto-deploys

## PIN Auth Note
The PIN check uses `crypto.subtle` (SHA-256), which requires HTTPS or localhost. On plain HTTP (e.g. local IP testing), `crypto.subtle` is unavailable — the code falls back to a direct string comparison. This is intentional and safe for local testing only; production (GitHub Pages) always uses HTTPS.

## Firebase Config Location
The Firebase config is at the top of the `<script>` section in `index.html`. Already populated with live values for project `teleprompter-app-66adb`.

## Important Constraints
- **Must remain a single HTML file** — no build step, no separate JS/CSS files
- **Rich text paste must work** — bold, italic, underline, line breaks from Notion/Pages
- **Mobile-friendly** — the teleprompter view must work well on phones (this is the primary reading device)
- **No authentication** — keep it simple, no user accounts
- **Short share URLs** — the whole point of Firebase is to avoid URL-length limits

## Keyboard Shortcuts (during playback)
- Space: play/pause
- Arrow Up/Down: adjust speed
- Escape: stop and return to editor

## Speed Slider
- Slider range is 1–10 but maps to actual speeds 1.0–5.5 in 0.5 increments via `sliderToSpeed()`
- This gives finer control at the low end which is where normal use sits

## Mobile UI — Element IDs / JS functions to know
- `#mobileTopBar` — the slim top bar (visible on ≤768px only)
- `#btnMobilePlay` — play/stop button; calls `mobilePlayStop()`
- `.mob-icon-btn` — the H-flip, V-flip, focus-line icon buttons in the top bar
- `#btnMobileChevron` — chevron that toggles the dropdown; rotates 180° when open
- `#mobileSheet` / `#mobileSheetPanel` — the dropdown settings panel
- `toggleMobileSheet()` / `closeMobileSheet()` — open/close the dropdown
- `updateMobilePlayBtn()` — syncs the ▶/■ state after play/stop
- `showTopBar()` — reveals the faded top bar and resets the 3s fade timer
- Mobile sliders are prefixed `mob-` (e.g. `mob-fontSize`) and stay in sync with their desktop counterparts via `syncSliders()`

## Known Issues / TODO
- Consider adding local script saving (IndexedDB) for offline use
- Consider a "recent scripts" list
