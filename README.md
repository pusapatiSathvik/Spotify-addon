# 🔁 Spotify Loop Tray
**Idea → Design → Implementation → Debugging → Working Feature**

Spotify currently supports two loop modes: looping one song and looping the entire playlist. There is no middle ground. If you want to loop just 2–10 songs for a temporary mood, creating a full playlist is high friction. 

**The Gap:** Loop 2–10 songs together, without creating a playlist, for as long as you want.

---

## ℹ️ Core Insight & UX
* **The Concept**: The "Add to Queue" interaction already exists in Spotify. The Loop Tray extends that same mental model: instead of adding one song to the queue, you build a small set and tell the app to cycle through it indefinitely.
* **Three Loop States**: 
    * **Off**: Normal playback, no looping.
    * **Loop All**: Existing Spotify behavior (loops entire playlist).
    * **Loop Tray**: **New** - cycles only through the 2–10 songs in your tray.
* **Native Feel**: The Loop Tray tab sits alongside the existing Queue tab. Songs are added per-track using a "+ Tray" button, mirroring the existing "Add to Queue" UX.

---

## 🛠️ Technical Implementation

### Authentication (Spotify Web API)
* **Flow**: Uses **Authorization Code + PKCE flow**. This is the current Spotify standard for browser-based apps (replacing the deprecated implicit grant from Nov 2025).
* **No Backend**: No server is required. Tokens are exchanged and stored in `localStorage`, then refreshed automatically using the refresh token with a 60s buffer.

### The Loop Engine: "Full Cycle Push" Strategy
Spotify provides **NO** endpoint to remove items from the queue, replace the queue, or read the full queue state beyond the next ~20 items. This constraint drove our workaround:

1. **Activation Phase**: 
    * Calls `POST /me/player/queue` for **EACH** tray song in order.
    * Uses a **300ms delay** between songs to avoid rate limiting (HTTP 429).
    * Each song retries up to 3 times with **exponential backoff** (500ms, 1000ms, 1500ms).
2. **Watcher Phase (Every 2s)**: 
    * Polls `GET /me/player` to update playback state and progress.
    * Compares the current track ID to detect changes.
    * When the `loopQueue` is empty, it triggers a **full cycle refill** (pushing all songs again).

---

## 📈 Evolution of the Solution
The final working implementation was reached through 5 iterations to overcome API hurdles:
* **Attempt 1-2**: Pre-queuing failed because Spotify’s own auto-fill context pushed tray songs further back.
* **Attempt 3-4**: Solved track change detection but faced UI freezes and a critical crash where Spotify's `POST /queue` returned an empty JSON object `{}`, breaking standard parsers.
* **Final Fixes**: Wrapped JSON parsing in try/catch, implemented a bulletproof API wrapper with 429 handling, and ensured UI rendering runs every 2s unconditionally.

---

## 📋 Function Reference

### API & Engine
| Function | Purpose |
| :--- | :--- |
| `api(path, opts)` | Universal wrapper: handles auth headers, JSON parse safety, 429 backoff, and 401 refresh. |
| `pushFullCycleToQueue()` | Iterates tray, calls queue API with 3-attempt retry + 300ms spacing. |
| `startWatcher()` | 2s interval that updates UI, detects changes, and manages cycle refills. |

### Tray & Auth
| Function | Purpose |
| :--- | :--- |
| `addToTray(track)` | Toggles song in/out of the tray and updates the UI badge. |
| `startAuth()` | Initiates PKCE flow by redirecting to Spotify /authorize. |
| `doRefresh()` | Uses refresh_token to get a new access_token without re-login. |

---

## 🔍 Debug Log System
The app features an in-browser debug log panel (accessible via the document icon in the player controls) to diagnose issues without DevTools.

| Level | Icon | Context |
| :--- | :--- | :--- |
| **LOOP** | 🔁 | Activation, deactivation, and full cycle refills. |
| **SUCCESS** | ✅ | Each individual song successfully added to the queue. |
| **WARN** | ⚠️ | Unexpected track detected (manual skip) or queue boundaries reached. |
| **ERROR** | ❌ | API failures, JSON parse errors, or exhausted push retries. |

---

## ⚠️ Known Limitations
* **Queue is Append-Only**: Songs already in the queue from Spotify's own context cannot be removed; the tray simply "outruns" them.
* **Manual Skip Desync**: If you manually skip, the counter shifts. The watcher logs a **WARN** and adjusts, though the next cycle start point may shift.
* **Premium Required**: All playback control endpoints require a Spotify Premium account.
* **Single Session**: The loop engine only runs while this specific browser tab is open.

---

## 🚀 How to Run
1. Create an app on the [Spotify Developer Dashboard](developer.spotify.com/dashboard).
2. Set your **Redirect URI** (e.g., `http://127.0.0.1:5500/index.html`).
3. Enable scopes: `user-read-currently-playing`, `user-read-playback-state`, `user-modify-playback-state`, `streaming`, `user-read-private`.
4. Open `spotify-loop-tray.html` via a static server (like VS Code Live Server).
5. Enter your **Client ID**, authorize, and start building your tray!

---
*Built with Spotify Web API — Auth: PKCE — No backend required.*