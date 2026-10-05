# Architecture: ReelMotion (working title)

> Companion to `PRD.md`. Read the PRD first.
> **[Decided]** = chosen by the owner. **[Suggested]** = my proposal, needs confirmation.
> Full hosting and deployment steps will live in a separate `Deployment.md` (written later).

---

## 1. Design principles

1. **Client-heavy.** Almost everything runs in the user's browser: AI calls, preview, and MP4 export. This keeps server cost near zero.
2. **Tiny backend.** Only one job needs a server: safely uploading images to storage.
3. **Preview = export.** The preview and the exported video must come from the exact same animation code path, so they always match.
4. **Untrusted code.** AI-written code is untrusted. It always runs in a sandbox with no access to secrets.
5. **Easy to vibe code.** Few moving parts, popular tools, small files, clear contracts between parts.

## 2. Tech stack

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Frontend | React + TypeScript + Vite + Tailwind CSS | **[Suggested]** | Owner mentioned React/Tailwind or Next.js. Next.js is not needed: no SEO or server rendering is required for a tool. Vite is simpler and AI coding tools make fewer mistakes with it. |
| Frontend hosting | Vercel (own subdomain) | **[Decided]** | Any free static host works. Check the host's terms about commercial use before monetizing. |
| Auth | Firebase Auth, Google sign-in | **[Decided]** | |
| Database | Firestore | **[Suggested]** | Stores projects and chat history. Has a free quota; check current limits. |
| Image storage | Cloudflare R2 | **[Decided]** | 10 GB free tier. |
| Small backend | Cloudflare Worker | **[Suggested]** | Owner suggested Node.js on Render. A Worker is better here: same ecosystem as R2, no cold starts, free tier. Node on Render stays an option if more backend work appears later. |
| AI | Gemini API, called from the browser with the user's own key | **[Decided]** + **[Suggested]** (browser-only key) | |
| Animation runtime | SVG scene driven by time (`render(t)`) | **[Suggested]** | See Section 5. |
| Preview sandbox | `<iframe sandbox>` | **[Suggested]** | See Section 9. |
| MP4 export | WebCodecs `VideoEncoder` + an MP4 muxer library (e.g. `mp4-muxer` or Mediabunny) | **[Suggested]** | Runs fully in the browser, works on phone and laptop. |
| Remotion | Not used | **[Decided]** | |

## 3. System overview

```
                         +------------------------------+
                         |        User's browser        |
                         |                              |
  Google sign-in <------>|  React app                   |
  (Firebase Auth)        |   - Chat panel               |
                         |   - Project list             |
  Firestore <----------->|   - Settings (Gemini key)    |
  (projects, chat)       |   - Preview panel            |
                         |       |                      |
  Gemini API <---------- |  Generation service          |
  (user's own key)       |       |                      |
                         |  +----v-----------------+    |
  svgrepo.com <--proxy-- |  | Sandboxed iframe     |    |
  (icon lookup)    |     |  | runs scene code      |    |
                   |     |  +----+-----------------+    |
                   |     |       |                      |
                   |     |  Exporter (WebCodecs, MP4)   |
                   |     +-------+----------------------+
                   |             | image upload
                   v             v
              +-------------------------+
              |   Cloudflare Worker     |-----> Cloudflare R2
              |  (verify login, proxy)  |       (uploaded images)
              +-------------------------+
```

## 4. Components

### 4.1 Web app (React)
- **Auth gate:** Google sign-in via Firebase Auth.
- **Settings:** Gemini key input and a guide on how to get one.
- **Project list:** create, rename, open, delete.
- **Workspace:** chat on the left, preview on the right.

### 4.2 Generation service (browser module)
- Builds the prompt for Gemini: system prompt + scene rules + helper library docs + project context.
- Calls Gemini directly with the user's key.
- Parses the response into either a **full scene** (first generation) or **patches** (edits).
- Shows errors with a **Retry** button. **[Decided]** No silent auto-fix loop in v1.

### 4.3 Icon resolver
Order **[Decided]**:
1. Ask the Worker proxy to look up the icon on svgrepo.com and return its SVG code.
2. If not found (or the user asks for a custom shape), the AI draws the SVG itself.

Why a proxy: browsers block direct requests to svgrepo.com from another site (CORS). The Worker fetches it server-side. SVG Repo has no official API, so this may break if the site changes; the AI-drawn fallback keeps the product working. Also track each icon's license and source (see PRD Section 10).

### 4.4 Preview sandbox
- The scene code runs inside an `<iframe sandbox="allow-scripts">` (no `allow-same-origin`).
- The parent talks to it with `postMessage` (see Section 6).

### 4.5 Exporter
- Steps through the animation frame by frame and encodes MP4 in the browser (Section 7).

### 4.6 Storage layer
- Firestore for projects and chat. R2 for images through the Worker.

### 4.7 Cloudflare Worker
- `POST /upload`: checks the user's Firebase login token, validates file type and size, stores the image in R2, returns its URL or key.
- `GET /icon?q=...`: icon lookup proxy for SVG Repo.
- `GET /asset/:key`: serves a user's image to that user only (or issues a short-lived URL).

## 5. Animation format (the "scene contract") **[Suggested]**

This is the most important technical decision, so the AI must follow it strictly.

Every generated animation is one module that provides:

```js
export const meta = { width: 1080, height: 1920, duration: 6, fps: 30, background: "#ebebeb" };

// Pure function: given time t (seconds), return the SVG markup for that exact moment.
export function render(t, helpers) {
  // use helpers.popIn(t, start, dur), helpers.slide(...), helpers.drawLine(...), easing, etc.
  return `<svg ...>...</svg>`;
}
```

Rules enforced in the system prompt and checked after generation:
- `render(t)` depends **only** on `t`. No `Date.now`, `Math.random`, `setTimeout`, CSS animations, or `requestAnimationFrame`.
- `meta.duration` must be 15 seconds or less **[Decided]**.
- All images and icons are **inlined** (SVG paths or base64 data URIs). No external URLs.
- Fonts are system-safe fonts, or embedded as base64.
- Output is deterministic: the same `t` always gives the same picture.

Why this contract:
- **Preview** just calls `render(t)` in a loop. Play, pause, resume, restart and scrubbing are trivial (change `t`).
- **Export** calls the same `render(t)` for `t = 0, 1/30, 2/30, ...`, so the video matches the preview exactly.
- It works on phones, where "record the screen" is not available.

**Difference from the original idea:** the original idea was to reload the preview and screen-record it. That only works in desktop Chrome and needs a permission prompt each time. The scene contract above gives the same result (a clip that plays from the start) without those limits. A quick desktop-only recording path can still be used as a stop-gap in an early phase (see `Phases.md`).

## 6. Preview protocol (parent <-> iframe)

| Message | Direction | Meaning |
|---|---|---|
| `init {code, assets}` | parent -> iframe | Load scene code and inlined assets |
| `ready {meta}` | iframe -> parent | Scene compiled; returns width, height, duration, fps |
| `play` / `pause` / `restart` | parent -> iframe | Playback control |
| `seek {t}` | parent -> iframe | Show the frame at time `t` |
| `frame {svg}` | iframe -> parent | (Export) SVG for a requested `t` |
| `error {message, stack}` | iframe -> parent | Compile or runtime error, shown to the user with Retry |

The iframe never receives the Gemini key or any login data.

## 7. Export pipeline **[Suggested]**

1. Read `meta` (size, fps, duration). Round width and height to even numbers (required by H.264).
2. Create a canvas of that size and a `VideoEncoder` (H.264) with an MP4 muxer.
3. For each frame `i` from `0` to `fps * duration`:
   - Ask the scene for `render(i / fps)`.
   - Draw the SVG onto the canvas (serialize SVG -> `Image` -> `drawImage`).
   - Create a `VideoFrame` and give it to the encoder with the right timestamp.
4. Flush the encoder, finalize the MP4, and trigger a download.
5. Show a progress bar based on frames done.

Defaults: 30 fps, about 1080 px on the short side, around 8 Mbps bitrate. Tune after testing.

Notes:
- Faster than real time on most devices, and no permission prompt.
- Phone support depends on the browser's WebCodecs support. Test on real devices early. If unsupported, show a clear message (and consider a fallback later).
- Keep memory in check: encode frame by frame, close each `VideoFrame` right after use.
- The 15 second cap keeps exports light on phones. **[Decided]**

## 8. Data model (Firestore) **[Suggested]**

```
users/{uid}
  - displayName, email, createdAt

users/{uid}/projects/{projectId}
  - name
  - aspectRatio        (e.g. "9:16")
  - durationSeconds
  - sceneCode          (current animation code; keep under 1 MB)
  - assets[]           (R2 keys of uploaded images)
  - createdAt, updatedAt

users/{uid}/projects/{projectId}/messages/{messageId}
  - role               ("user" | "assistant")
  - text
  - createdAt
```

- Messages are in a subcollection because a single Firestore document is limited to about 1 MB.
- **Security rules:** a user can read and write only under `users/{their own uid}`.
- The Gemini key is **not** stored here. **[Suggested]**

## 9. Security

| Area | Rule |
|---|---|
| Generated code | Runs only in a sandboxed iframe without `allow-same-origin`, so it cannot read the app's storage, cookies, or the Gemini key. |
| Gemini key | Stored only in the user's browser. Never sent to the owner's backend or into the iframe. |
| Network from iframe | Restrict with a Content Security Policy where possible (no external requests from scene code). |
| Uploads | Worker checks the Firebase ID token, file type (images only), and size limit. Files are stored under the user's id. |
| Firestore | Rules limit each user to their own data. |
| Icons | Treated as untrusted SVG: strip scripts and event handlers before use. |
| Secrets | R2 credentials and similar secrets live only in Worker settings, never in the frontend. |

## 10. Error handling

- Compile or runtime error in the scene -> show the message with a **Retry** button **[Decided]**.
- Gemini errors (invalid key, rate limit, network) -> specific, friendly messages.
- Export failure -> show reason and let the user try again.
- Icon lookup failure -> silently fall back to AI-drawn SVG.

## 11. Hosting overview

| Part | Where |
|---|---|
| Frontend | Vercel (or similar free static host) |
| Auth + Firestore | Firebase |
| Worker + image storage | Cloudflare (Workers + R2) |
| AI | User's own Gemini key, called from the browser |

Detailed setup, environment variables, and deployment steps go into `Deployment.md` later.

## 12. Technical risks

| Risk | Plan |
|---|---|
| Gemini writes code that breaks the scene contract | Strong system prompt, helper library, validation step, Retry button |
| Phone export limits | Test early on real devices; 15 second cap; clear error message |
| SVG Repo breaks or blocks the proxy | AI-drawn SVG fallback; optional bundled icon set later |
| Large scene code | Keep scenes small; limit output size in the prompt |
| Fonts differ between preview and export | System-safe fonts or embedded fonts only |

## 13. Open items

- Final choice of MP4 muxer library (test `mp4-muxer` vs Mediabunny).
- Exact default Gemini model (use a fast Flash-class model; confirm the current name when building).
- Exact export resolution per aspect ratio.
- Whether the Worker serves images directly or issues short-lived URLs.
