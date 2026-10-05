# Phases: ReelMotion (working title)

> Build plan for v1. Companion to `PRD.md` and `Architecture.md`.
> Written for **vibe coding**: each phase is small, has a clear "done when", and can be given to an AI coding tool one at a time.
> Time estimates are rough guesses for one person building with AI help. Treat them as a guide, not a promise.

---

## How to use this document with an AI coding tool

1. Give the AI these files at the start of each session: `PRD.md`, `Architecture.md`, and (once it exists) the AI rules file.
2. Work on **one phase at a time**, and inside it, **one step at a time**. Do not ask the AI to "build the whole app".
3. After each step: run it, test it, then commit to git. If something breaks, go back to the last commit.
4. Test on a **real phone** from Phase 1 onward, not only on the laptop.
5. If the AI wants to change the stack or the scene contract (Architecture Section 5), stop and decide yourself first.

## Order and why

The riskiest part is **turning code into an MP4 that also works on phones**. So that is built **first**, with a hard-coded animation and no AI. If export does not work well, everything else is wasted effort, and it is better to know in the first week.

```
Phase 0  Setup
Phase 1  Scene player + MP4 export (no AI yet)      <- biggest risk, go/no-go
Phase 2  AI generation (Gemini key, prompt -> scene)
Phase 3  Login + projects (Firebase)
Phase 4  Edit by chat (patch-based edits)
Phase 5  Images + icons (Worker, R2, SVG Repo)
Phase 6  Polish + first creators
```

---

## Phase 0: Setup (about 1 day)

**Goal:** an empty app that is live on the internet.

Steps:
- [ ] Create a git repository.
- [ ] Create the app with Vite + React + TypeScript + Tailwind.
- [ ] Deploy it to Vercel (own subdomain) and confirm the page opens.
- [ ] Create a Firebase project (Auth + Firestore) but do not use it yet.
- [ ] Create a Cloudflare account (Workers + R2) but do not use it yet.
- [ ] Write the AI rules file (stack, folder layout, coding rules, "do not change the scene contract").

**Done when:** the empty app opens at its public URL and the repo has the first commit.

---

## Phase 1: Scene player and MP4 export (about 3-5 days)

**Goal:** a hard-coded animation plays in a preview box and downloads as an MP4, on laptop **and** phone.

Steps:
- [ ] Write the helper library: easing, `popIn`, `slide`, `drawLine`, fade, scale.
- [ ] Write one hard-coded scene as `render(t)` (the WhatsApp -> line -> export icon -> zip icon example, 6 s, background `#ebebeb`).
- [ ] Build the sandboxed iframe that runs a scene (`sandbox="allow-scripts"`) and the `postMessage` protocol (Architecture Section 6).
- [ ] Build the preview panel: aspect ratio box (default 9:16), Start, Pause, Resume, Restart, and a seek bar.
- [ ] Build the exporter: frame-by-frame render -> canvas -> WebCodecs -> MP4 -> download, with a progress bar.
- [ ] Test the exported MP4 in a video player and by uploading it to Instagram or YouTube Shorts.
- [ ] Test on at least one Android phone and one iPhone (if available), plus desktop Chrome.

**Done when:** the MP4 looks the same as the preview, the timing is exact, and export works on the phones you tested.

**Go / no-go:** if phone export fails, decide before continuing: lower resolution, shorter cap, a desktop-only fallback, or a different export approach. Do not start Phase 2 until this is settled.

---

## Phase 2: AI generation (about 3-5 days)

**Goal:** the user types a prompt and gets a working animation.

Steps:
- [ ] Settings screen: enter the Gemini key, saved only in the browser. Add a short "how to get a key" guide.
- [ ] Gemini client in the browser, with friendly errors (invalid key, rate limit, network).
- [ ] Write the system prompt: scene contract rules, helper library docs, one or two example scenes, the 15 second cap.
- [ ] Parse the AI response, validate it (has `meta` and `render`, duration <= 15, no forbidden things like `Date.now` or `Math.random`).
- [ ] Read aspect ratio and duration from the prompt; default to 9:16.
- [ ] Load the generated scene into the preview and export it with the Phase 1 exporter.
- [ ] Error state: show the error with a **Retry** button (no silent auto-fix).
- [ ] Build a **test set of 20-30 prompts** (simple, medium, tricky) and save it in the repo. Re-run it whenever the system prompt or model changes.

**Done when:** most test prompts give a playable animation, failures show a clear error and Retry, and generated scenes export correctly.

---

## Phase 3: Login and projects (about 3-4 days)

**Goal:** users sign in and their work is saved.

Steps:
- [ ] Google sign-in with Firebase Auth; sign-out.
- [ ] Firestore data model from Architecture Section 8.
- [ ] Firestore security rules: each user can only access their own data. **Test the rules.**
- [ ] Project list: create, rename, open, delete.
- [ ] Workspace layout: chat left, preview right, responsive for phone (stacked layout).
- [ ] Autosave scene code and chat messages.
- [ ] Reopen a project and see the same animation and chat.

**Done when:** a user can sign in on one device, create a project, close the browser, come back, and continue.

---

## Phase 4: Edit by chat (about 2-3 days)

**Goal:** "make the line faster" changes only that part.

Steps:
- [ ] Change the AI response format for follow-ups to patches (search/replace blocks) instead of a full rewrite.
- [ ] Apply patches to the current scene code; if a patch does not apply, show an error with Retry.
- [ ] Re-validate the scene after each edit.
- [ ] Keep a simple version history (at least the previous version) so the user can undo.
- [ ] Add follow-up prompts to the test set ("make it faster", "change the background", "add a second icon").

**Done when:** common edits work without breaking the rest of the animation, and undo works.

---

## Phase 5: Images and icons (about 4-6 days)

**Goal:** users can upload images, and icons come from SVG Repo with an AI fallback.

Steps:
- [ ] Cloudflare Worker: check the Firebase login token, accept image uploads (type and size limits), store in R2.
- [ ] Image upload button in chat; show thumbnails; save references in the project.
- [ ] Convert images to inline data (base64) before passing them to the scene (Architecture Section 5).
- [ ] Worker endpoint for icon lookup on SVG Repo.
- [ ] Sanitize SVG (remove scripts and event handlers).
- [ ] Icon resolver: try SVG Repo first, otherwise let the AI draw the SVG.
- [ ] Keep a note of each used icon's source and license.
- [ ] Add prompts with logos, uploaded images, and custom shapes to the test set.

**Done when:** an uploaded image appears correctly in the animation and in the exported MP4, and an icon request works even when SVG Repo returns nothing.

---

## Phase 6: Polish and first creators (about 4-7 days)

**Goal:** something real creators can use.

Steps:
- [ ] Onboarding: guided Gemini key setup with screenshots (this is the biggest drop-off risk).
- [ ] Clear messages for every error type.
- [ ] Phone UI pass: buttons, panels, keyboard behavior, export on slow phones.
- [ ] Limits: image size, project count, text length.
- [ ] Simple landing section explaining what the tool does, with an example output.
- [ ] Add basic analytics for the PRD metrics (users who generate, users who export).
- [ ] Short terms/credits note (icon sources, trademarks, user responsibility for uploaded content).
- [ ] Write `Deployment.md` with the final setup steps.
- [ ] Give the tool to 5-10 real creators and watch where they get stuck.

**Done when:** a new creator can go from sign-in to an exported MP4 without your help.

---

## Rough total

About **4-6 weeks** for one person with AI help, if things go smoothly. The two places that most often take longer are phone export (Phase 1) and getting Gemini to follow the scene contract reliably (Phase 2).

## After v1 (not planned yet)

- Audio and music.
- Longer videos.
- Templates and sharing.
- Paid options (decide only after seeing real usage).
- Bundled icon fallback if SVG Repo proves unreliable.

## Documents still to write

- `Deployment.md`: hosting and environment setup (Vercel, Firebase, Cloudflare), written later as requested.
- AI rules file: instructions for the AI coding tool (stack, folder layout, what not to change).
