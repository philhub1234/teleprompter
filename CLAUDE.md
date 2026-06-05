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
- **Sidebar (left)**: Settings only — font size, line height, speed, margins, mirror/flip, text color, formatting toolbar (B/I/U). Collapsible.
- **Main stage (right)**: This IS the script editor AND the teleprompter display. In edit mode, it's a contenteditable area where you paste/type your script. When you hit Start, the same content scrolls as the teleprompter.

### Cross-device script sharing (Firebase Firestore)
The old approach encoded the entire script as base64 in the URL query string. This breaks on long scripts (URL too long for browsers). The new approach:

1. User writes/pastes script on laptop
2. Clicks "Share" → script + settings are saved to Firebase Firestore
3. Gets a short URL like `https://philhub1234.github.io/teleprompter/#s/abc123`
4. Opens that URL on phone → script loads automatically

**Firebase setup** (one-time, already done or needs to be done):
- Project: Create at https://console.firebase.google.com
- Enable Firestore in test mode (or with read-anyone / write-anyone rules for simplicity)
- Copy the Firebase config object into the `firebaseConfig` variable in `index.html`
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
1. **Edit mode**: Sidebar open, stage shows editable script area with placeholder text
2. **Playing**: Sidebar collapses, stage scrolls the script, floating controls appear (speed ±, play/pause, restart, stop)
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
4. Push to `main` branch → GitHub Pages auto-deploys

## Firebase Config Location
The Firebase config is at the top of the `<script>` section in `index.html`. Look for:
```js
const firebaseConfig = { ... };
```
This needs to be filled in with real values from the Firebase console.

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

## Known Issues / TODO
- Firebase project needs to be created and config added
- Consider adding local script saving (IndexedDB) for offline use
- Consider a "recent scripts" list
