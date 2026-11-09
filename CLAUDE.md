# Gideon — project handoff for Claude Code

## Who you're working with
- **Liam Perreault**, a beginner coder who works **only on an iPhone** (no Mac/PC, no Terminal).
- Explain things simply and step by step, with copy-pasteable blocks. He likes learning *how* things work, so briefly explain what each change does.
- He deploys by uploading files to GitHub through **Safari on iPhone**. Keep the project as a few static files, with no build step and no npm. If he's using Claude Code with this repo connected, you can commit directly instead.

## What Gideon is
A personal AI voice assistant, inspired by the ship's-computer AI "Gideon" from *The Flash*, but with an **original** look and voice. We deliberately don't copy the show actress's face or voice, and we don't use DC assets.

- A **3D holographic glass head and shoulders** (Three.js) in front of a studded metal wall with wispy light streaks.
- You talk to it by voice or text. It answers out loud with **lip-sync** and blinks.
- A **wake word**: say "Gideon" while the app is open.
- The brain is **Google Gemini (free tier)**, with optional Google Search grounding.

## Hard constraints
- **100% free.** Gemini API free tier only (key from aistudio.google.com, no credit card). No paid hosting.
- **Hosting:** GitHub Pages from repo `liamxper42/Gideon`. The live URL is case-sensitive:
  `https://liamxper42.github.io/Gideon/`
- **The API key must never be in the code.** The repo is public. The user pastes the key into the in-app ⚙︎ Setup, and it's stored in `localStorage` on the phone only.
- **Runs in iPhone Safari** and as an "Add to Home Screen" web app.

## Files
| File | Purpose |
|---|---|
| `index.html` | The entire app: HTML + CSS + JS in one file. Loads Three.js r128 from cdnjs. |
| `icon.png` | 180×180 Home Screen icon (`apple-touch-icon`). |
| `README.md` | Auto-created by GitHub; harmless. |
| `CLAUDE.md` | This file. |

## How index.html is organised (top to bottom)
1. **Meta tags:** Home Screen web-app tags (`apple-mobile-web-app-capable`, status bar, title, icon), `viewport-fit=cover`.
2. **CSS:** dark sci-fi theme (Orbitron + Inter fonts). `#stage` has the studded wall drawn in pure CSS (two repeating radial gradients) plus a fade. There's also the chat log, a control bar, the `.modal` settings card and the `#boot` "TAP TO ACTIVATE" overlay.
3. **HTML:** header (title, status line, ⚙︎ button), `#stage` holding `<canvas id="wisps">` (2D) and `<canvas id="orb">` (WebGL head), `#log` chat, `#heard` (live transcript), `#note` (yellow warnings), the form (🔊 voice toggle, 🎙️ talk once, 👂 wake-word toggle, text input, SEND), `#boot` overlay and `#settings` modal (key, model, name, city, voice select, speed/pitch sliders, test voice).
4. **`SYSTEM` prompt:** Gideon's personality (calm, formal, dry wit, short spoken answers, no markdown, teen-appropriate). It has `{NAME}` and `{CITY}` placeholders, and the current date/time is appended at call time.
5. **3D head (`buildHead`, `sculpt`)**
   - A `SphereGeometry(1, 240, 240)` is sculpted vertex by vertex in `sculpt(x,y,z,closed)` using Gaussian `bump()`s for every facial feature: eye sockets, eyeballs, eyelids/creases, brows, nose bridge/tip/nostrils, philtrum, cupid's bow, lips, chin, cheekbones, nasolabial folds and temples. Ears are built as base + helix rim + concha + lobe. A smooth proportion pass shapes the head (round skull, softer jaw). Avoid hard `if` switches in the shaping, because they create visible seams.
   - `fsp()` = "face space" scaling (`FK=1.32`, `FY0=-0.08`), so the features sit at human spacing. Feature constants: `EYE_X`, `EYE_Y`, `MOUTH_Y`.
   - Per-vertex data: `jawW` (jaw hinge weight), `mouthW` (lip shaping), `gap` (darkens the mouth opening in the shader), `irisA` (soft iris shading), `blinkD` (open→closed eyelid offsets, precomputed by calling `sculpt(...,1)`). Only the `moving` vertex list is updated each frame, for performance.
   - **Shader** (`ShaderMaterial`): a frosted ice-glass look. It mixes dark blue to white by key/fill lighting, adds a Fresnel edge glow, a specular highlight, faint animated "veins" (thresholded sines), soft scanlines and a sweep band. Uniforms: `uTime`, `uLevel`, `uOpen`, `uBlink`, `uFade`.
   - **Bust:** a `LatheGeometry` neck and shoulders, flattened front-to-back, that fades out at the bottom. The head sits in `headGroup` (scale 0.9). Projector rings and floating particle motes surround it.
   - Drag to rotate (pointer events). Idle sway and bob.
6. **Lip sync:** `VISEME` maps each letter to [jaw open, lip round]. `speechSynthesis` `onboundary` word events drive `lipStartWord()`. If a voice gives no boundaries, `lipWholeText()` estimates the timing. The animation loop eases `mouthOpen` and `mouthRound`.
7. **Voice (TTS):** browser `speechSynthesis`. The user picks a voice, speed and pitch in Setup (stored as `gideonVoice`, `gideonRate`, `gideonPitch`). iOS requires a user tap before any audio, so the `#boot` overlay handles the first tap. `speak(text, after)` runs the `after` callback when finished; it's used to resume listening.
8. **Listening (STT):** `webkitSpeechRecognition`.
   - `startRec('query')` listens for one question and sends it after a 1.4 s pause.
   - `startRec('wake')` listens continuously. `findWake()` does fuzzy matching (edit distance ≤2 for words starting with g/j/k, ≤1 otherwise; also handles two-word splits like "giddy on"). Saying "Gideon …" sends the rest directly; saying just "Gideon" makes it speak "Yes?" and then listen for the question.
   - Recognition stops while Gideon speaks or thinks. A watchdog restarts it every 3 s, it's force-restarted every ~50 s, and it resumes when the page becomes visible again.
9. **Brain:** `callGemini(model, useSearch)` POSTs to
   `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` with header `x-goog-api-key`. The body has `system_instruction`, plus `contents` (the last 20 turns, with roles user/model), optional `tools:[{google_search:{}}]`, and `thinkingConfig.thinkingLevel:'low'` for `gemini-3*` models. Each request is aborted after 25 s.
   - `askGemini()` tries the chosen model, then `FALLBACK_MODELS`, each with and without search. It remembers the working model and turns off search (`searchOK=false`) if search fails. It stops immediately on a bad key or no internet. Errors show in `#note` as "Google says: …".
10. **Settings:** the `store` helper wraps `localStorage` in try/catch. Keys: `gideonKey`, `gideonModel` (default `gemini-3.8-flash`), `gideonName` (default Liam), `gideonCity`, `gideonVoice`, `gideonRate`, `gideonPitch`, `gideonMemory` (JSON array of facts), `gideonTimers` (JSON array of `{id,end,label}`).
11. **Action tags:** the `SYSTEM` prompt teaches Gemini to add hidden tags to replies. `applyActions()` strips them and acts on them before the reply is shown or spoken:
    - `[TIMER: seconds | label]` → `addTimer()`. Timers store an absolute `end` time, so they survive reloads. A 1 s interval rings due timers (`chime()` via WebAudio + spoken message + alert mood), but only after the boot tap and never while speaking/thinking. Overdue timers say "While I was offline…". Timer chips in `#timers` show the countdown; tap one to cancel.
    - `[CANCEL]` clears all timers. `[REMEMBER: fact]` → `remember()` (max 40, de-duplicated). `[FORGET]` clears memory. `[MOOD: alert|happy]` → `setMood()`.
    - Tag names must not contain `_ * # \``, because `callGemini()` strips those characters.
    - Memory facts and running timers are appended to the system prompt on every call. Setup shows the memory list with a FORGET MEMORIES button.
12. **Mood colours:** `MOODS` holds `tint` (multiplies the shader colour via the `uTint` uniform, shared by head and bust) and `glow` (sets the CSS `--glow` var, so the title and buttons recolour too). Thinking = purple automatically, alert = red (timers, errors, `[MOOD: alert]`), happy = gold, calm = ice blue. `updateMood()` eases toward the target every frame.

## History (what we tried and learned)
1. **v1 was built as a claude.ai artifact.** It used Claude through the artifact `sample` capability, which needs no key. Inside the Claude app's sandbox, though, the **mic and sound are blocked**, so we moved to a standalone page in Safari.
2. The head evolved from a 2D canvas orb, to a 2D head, to a Three.js sphere head, to a refined, frosted-glass, bust version styled after a reference image. The style was copied (icy glass, studded wall, wisps); the face is original.
3. Gemini setup issues:
   - `gemini-2.5-flash` returned 404, because it's retired.
   - Usage-limit (429) errors, which were likely caused by search grounding or the model not being free on the key. That's why the fallback loop and search auto-disable exist.
4. GitHub Pages issues: README showing instead of the app (index.html was missing at first), 404s from wrong capitalisation (`/Gideon/` vs `/gideon/`), and Safari caching. Add `?v=N` to the URL, and re-add the Home Screen icon after updates.
5. The Google AI Studio account must be 18+. Liam's key was made by an adult. Don't help bypass age checks.

## Known limitations / open issues
- **Timers only ring while Gideon is open on screen** (no background notifications for web apps without a server). If the app was closed, an overdue timer is announced the next time it opens.
- **Wake word only works while the app is open and on screen.** iOS doesn't allow background listening for web apps. Safari speech recognition is flaky: it stops on silence and sometimes needs a tap (the status shows "Mic paused, tap 👂"). The mic may not work at all in Home Screen (standalone) mode on some iOS versions; Safari is the fallback.
- **Background "Hey Siri" usage** needs an iOS **Shortcuts** shortcut named "Gideon": Speak "Yes?" → Dictate Text → Get Contents of URL (Gemini POST, JSON body as above) → unwrap `candidates[0].content.parts[0].text` with Get Dictionary Value / Get Item from List → Speak Text. The model in the URL should be current (e.g. `gemini-3.8-flash`); leave out search to avoid quota issues. We offered to write this but haven't yet.
- Siri's own voices aren't available to web pages. The best results come from downloading Enhanced/Premium voices (Settings → Accessibility → Spoken Content → Voices).
- Gemini model names change over time. If you get a 404, check the current free Flash models.

## Ideas Liam may want next
- Write the Siri Shortcut steps in full.
- A startup "boot sequence" animation and sound.
- Optionally replace the procedural head with a CC0 3D head model (glTF) for more realism; the lip-sync would need morph targets or vertex weights.

## Working rules for this repo
- Keep everything in `index.html` unless there's a strong reason not to.
- Test in mobile Safari terms: there's no hover, iOS needs a gesture before audio, there are safe-area insets, and `speechSynthesis` voices load asynchronously.
- Never commit an API key. Never log the key.
- After a change, tell Liam to reload with a new `?v=` number and to re-add the Home Screen icon if needed.
