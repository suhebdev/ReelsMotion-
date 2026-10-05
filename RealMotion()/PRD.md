# PRD: ReelMotion (working title)

> Product Requirements Document, v1 draft.
> Items marked **[Decided]** were chosen by the owner. Items marked **[Suggested]** are my proposals and need confirmation. Items marked **[Open]** are not decided yet.

---

## 1. Overview

ReelMotion is a website where a user types a description of an animation, and an AI builds it as live code. The user sees the result in a preview box that behaves like a short video, can ask the AI for changes in chat, and can export the preview as an MP4.

Everything is code. No stock video, no video generation model. The AI writes the animation (SVG-based) and the browser plays it.

**Owner / brand:** Built by Vibecoder, as one of several tools published under the same creator brand. **[Open]** Final product name (see Section 12).

## 2. Problem

Short-form creators (Reels, Shorts) often need small motion-graphics clips: logo reveals, flow diagrams, explainer snippets. Today they either learn heavy tools, pay for templates, or ask a general chatbot for code they cannot turn into a video. There is no simple "describe it, see it, export it as MP4" flow that works in a browser.

## 3. Target users

**[Decided]** Reels/Shorts content creators.

- Mostly non-technical.
- Use both laptop/PC and phone about equally. **[Decided]** Both must be supported.
- Need vertical (9:16) output most of the time.

**Implication:** setup must be very simple. The only technical step in v1 is getting a Gemini API key (see Section 6.2). This is the biggest friction point in the product.

## 4. Goals and non-goals (v1)

### Goals
1. A user can go from prompt to an exported MP4 in a few minutes without installing anything.
2. The preview and the exported video look the same.
3. The product costs the owner (almost) nothing to run.
4. Works on laptop and phone.

### Non-goals for v1
- Audio or music. **[Decided]** v1 is silent.
- Animations longer than 15 seconds. **[Decided]**
- Timeline or manual drag-and-drop editor.
- Monetization. **[Decided]** Not planned now; build first and see.
- Templates marketplace, team features, sharing links.
- Video generation by AI models.

## 5. Core user flow

1. User opens the site and signs in with Google.
2. User enters their Gemini API key (first time only, per device).
3. User creates a project.
4. User types a prompt in the chat (optionally uploads images).
5. AI generates the animation. The preview box appears in the aspect ratio from the prompt (default 9:16).
6. User presses Start / Pause / Resume / Restart on the preview.
7. User asks for changes in chat; the AI edits only what was asked.
8. User presses Download. The preview replays from the start and is exported as an MP4.
9. User returns later and finds the project saved.

**Example prompt:** "WhatsApp logo pops in, then slides left. A line draws left to right, an export icon pops in, the line continues, and a zip icon pops in at the end. 6 seconds total. Background #ebebeb. Icons keep their original colors."

## 6. Functional requirements

### 6.1 Authentication **[Decided]**
- Sign in with Google (Firebase Auth).
- Signed-in users see their saved projects.

### 6.2 Gemini API key **[Decided + Suggested]**
- **[Decided]** The tool does not work until the user enters their own Google AI Studio Gemini API key. This keeps the owner's AI cost at zero.
- **[Suggested]** The key is stored only in the user's browser and never sent to the owner's server. The AI is called directly from the browser.
- **[Suggested]** Clear in-app guide (with screenshots) on how to get a key, since users are non-technical.
- Known trade-off: user re-enters the key on a new device.
- Free-tier Gemini limits (rate limits) apply to the user, so errors from rate limiting must be explained clearly.

### 6.3 Projects **[Decided]**
- User can create, rename, open, and delete projects.
- A project stores: chat history, the current animation code, aspect ratio, and references to uploaded images.
- Projects are saved automatically.

### 6.4 Chat interface (left panel)
- Text prompt input.
- Image upload (PNG/JPG at minimum). If the user attaches an image and asks to use it, the animation uses it.
- Prompts may be written in English or Hinglish. **[Suggested]**

### 6.5 Animation generation
- The AI writes the animation as code (SVG-based). **[Decided]**
- Aspect ratio is taken from the prompt if given; otherwise 9:16. **[Decided]**
- Duration is taken from the prompt, with a hard maximum of 15 seconds. **[Decided]**
- Suggested supported ratios: 9:16, 1:1, 4:5, 16:9. **[Suggested]**
- Custom elements the user describes are drawn by the AI as SVG shapes. **[Decided]**

### 6.6 Icons **[Decided]**
- First, try to find a matching SVG on svgrepo.com and use its code.
- If none is found, or the user asks for a specific shape, the AI draws the SVG itself.
- **Risks to track:** SVG Repo has no official API and may change; licenses are set per icon (not all are free of conditions); brand logos (e.g. WhatsApp) are trademarks. See Section 10.
- **[Suggested, optional later]** A small bundled icon set as a fallback if SVG Repo fails.

### 6.7 Preview box (right panel)
- Shows the animation like a web page that runs from the start when started or restarted.
- Controls: Start, Pause, Resume, Restart.
- Box size follows the aspect ratio.
- Download button (see 6.9).

### 6.8 Editing by chat **[Decided]**
- When the user asks for a change ("make the line faster", "background blue"), the AI edits only the relevant part of the existing code instead of regenerating everything.

### 6.9 Export **[Decided]**
- Download exports **only the preview box**, from the start to the end of the animation, as an **MP4**.
- Export must work on **both laptop and phone** (this rules out approaches that only work in desktop Chrome; see Architecture.md).
- Quality: good enough for Reels/Shorts upload (target 1080 px on the short side, 30 fps). **[Suggested]**
- Shows progress while exporting.

### 6.10 Error handling **[Decided]**
- If the generated code fails or the preview is blank, show the error to the user with a **Retry** button. No silent auto-fix loop in v1.

### 6.11 Image storage **[Decided]**
- Uploaded images are stored in Cloudflare R2 (owner's bucket, free tier of 10 GB).
- Basic limits on file size and file type. **[Suggested]**

## 7. Non-functional requirements

- **Cost:** near zero for the owner. AI cost sits with the user's own key; hosting and storage stay in free tiers.
- **Security:** AI-generated code must run in a sandboxed iframe so it cannot read the user's Gemini key or login data. **[Suggested, important]**
- **Privacy:** Gemini key never leaves the user's browser. Uploaded images are only accessible by their owner.
- **Performance:** preview starts within a couple of seconds after generation; export of a 15 s clip should finish in a reasonable time on a mid-range phone.
- **Browsers:** latest Chrome, Edge, Safari (iOS), Firefox as far as export supports. **[Suggested]**
- **Language:** UI in English, with Hinglish prompts supported. **[Suggested]**

## 8. Success metrics (v1)

Simple to start with:
- Number of users who generate at least one animation.
- Number who export at least one MP4.
- Share of generations that succeed without a Retry.
- Share of users who get past the API key step.

## 9. Assumptions

- Users can get a free Gemini API key from Google AI Studio.
- Modern browsers can encode MP4 on-device (needed for the free, serverless export).
- Free tiers of Firebase and Cloudflare are enough for early usage.

## 10. Risks

| Risk | Why it matters | Mitigation |
|---|---|---|
| API key step scares non-technical creators | Could block most users before first use | Guided setup with screenshots; measure drop-off |
| Gemini free-tier rate limits | Generation may fail for users | Clear error message and Retry |
| SVG Repo changes or has no match | Icon step breaks | AI-drawn fallback (already planned); optional bundled set later |
| Icon licensing and trademarks | Some icons need attribution; logos are trademarks | Check each icon's license; keep a note of source |
| Generated code is unreliable | Blank or broken previews | Strict system prompt, helper functions, Retry button |
| Running AI-written code in the page | Could steal the key or data | Sandboxed iframe |
| Export on phones | Heavier and browser-dependent | Frame-by-frame export; 15 s cap; test early |
| Existing tools overlap | Prompt-to-animation exists in open-source form | Focus on creators, simple setup, and MP4 export that just works |

## 11. Out of scope for v1 / future ideas

- Audio and music.
- Longer videos.
- Manual timeline editor.
- Templates and sharing.
- Paid plans or other monetization (**[Open]**).
- Multiple AI providers.

## 12. Open questions

1. **Product name.** **[Open]** Working title is "ReelMotion by Vibecoder". Check domain and social handles before finalizing.
2. Which Gemini model to use by default. **[Suggested]** A fast, low-cost Flash-class model; confirm the current model name when building.
3. Exact image upload limits.
4. Whether to add a bundled icon fallback in v1 or later.
