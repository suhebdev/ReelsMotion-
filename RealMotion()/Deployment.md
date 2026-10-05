# Deployment and Stack: ReelMotion (working title)

> Companion to `PRD.md`, `Architecture.md`, and `Phases.md`.
> This file says **what runs where**, **which accounts are needed**, **how to set each part up**, and **how to release safely**.
> Free-tier numbers change over time. Always re-check the provider's pricing page before relying on a limit.
> Placeholders look like `<this>`. Fill them in as you go.

---

## 1. Final stack

| Layer | Tool | Where it runs |
|---|---|---|
| Frontend | React + TypeScript + Vite + Tailwind CSS | Vercel (own subdomain) |
| Login | Firebase Authentication (Google sign-in) | Firebase |
| Database | Firestore (projects, chat messages) | Firebase |
| Backend | Cloudflare Worker (image upload, icon lookup proxy) | Cloudflare |
| Image storage | Cloudflare R2 (private bucket) | Cloudflare |
| AI | Gemini API, called from the browser with the user's own key | Google (user's account) |
| Animation preview and MP4 export | Browser (sandboxed iframe, WebCodecs) | User's device |
| Source code | Git repository | GitHub (or similar) |

Not used: Remotion, Next.js, Render, a self-hosted database, a paid video API.

## 2. Accounts needed

- [ ] GitHub account (code + Vercel auto-deploys)
- [ ] Vercel account
- [ ] Google / Firebase account
- [ ] Cloudflare account
- [ ] Domain name (optional; a free Vercel subdomain is enough for v1)

Note: some providers ask for a payment method at signup even for free tiers. Check at signup time.

## 3. Repository layout (suggested)

```
/                      repo root
  /app                 React + Vite frontend
  /worker              Cloudflare Worker (upload + icon proxy)
  /docs                PRD.md, Architecture.md, Phases.md, Deployment.md
  /tests/prompts       saved test prompts for AI generation
```

Keeping the frontend and Worker in one repo makes it easier for an AI coding tool to see both sides.

## 4. Environment variables

### 4.1 Frontend (set in Vercel, names must start with `VITE_`)

| Name | Meaning |
|---|---|
| `VITE_FIREBASE_API_KEY` | Firebase web app key (this is **not** a secret; real protection comes from Firebase rules) |
| `VITE_FIREBASE_AUTH_DOMAIN` | From Firebase web app config |
| `VITE_FIREBASE_PROJECT_ID` | From Firebase web app config |
| `VITE_FIREBASE_APP_ID` | From Firebase web app config |
| `VITE_WORKER_URL` | Public URL of the Cloudflare Worker |

Never put real secrets in `VITE_` variables. Everything with that prefix is visible to anyone who opens the site.

The user's Gemini key is **not** an environment variable. Users enter it in the app and it stays in their browser.

### 4.2 Worker (set in Cloudflare)

| Name | Type | Meaning |
|---|---|---|
| `BUCKET` | R2 binding | The R2 bucket for uploaded images |
| `FIREBASE_PROJECT_ID` | Variable | Used to check Firebase login tokens |
| `ALLOWED_ORIGIN` | Variable | The frontend URL allowed to call the Worker (CORS) |

Anything secret goes in Worker **secrets**, not in the code or the repo.

## 5. Setup steps

### 5.1 Firebase

1. Create a Firebase project.
2. Add a **Web app** and copy its config values into the Vercel variables (Section 4.1).
3. Enable **Authentication** -> **Google** sign-in provider.
4. Under Authentication settings, add your Vercel domain (and `localhost` for development) to **authorized domains**.
5. Create a **Firestore** database in production mode.
6. Write the security rules so each user can only read and write `users/{their own uid}/...` (see `Architecture.md` Section 8).
7. Install the Firebase CLI and deploy the rules: `firebase deploy --only firestore:rules`.
8. **Test the rules** with a second Google account to confirm that user B cannot read user A's projects.

### 5.2 Cloudflare R2 and Worker

1. Create an R2 bucket (private; do not enable public access): `npx wrangler r2 bucket create <bucket-name>`.
2. In `/worker`, create the Worker project with Wrangler.
3. In `wrangler.toml`, bind the bucket as `BUCKET` and set the plain variables (`FIREBASE_PROJECT_ID`, `ALLOWED_ORIGIN`).
4. The Worker must:
   - Verify the Firebase ID token on every request (use a JWT library such as `jose`, and check Firebase's documentation for the current public-key endpoint).
   - Accept only image types, with a maximum file size.
   - Save files under the user's id, for example `<uid>/<projectId>/<fileId>`.
   - Return files only to their owner.
   - Add CORS headers for `ALLOWED_ORIGIN` only.
   - Provide the icon lookup endpoint for SVG Repo (and fail gracefully if the site changes).
5. Run locally with `npx wrangler dev` and test uploads.
6. Deploy with `npx wrangler deploy`. Copy the Worker URL into `VITE_WORKER_URL` on Vercel.
7. For any secret: `npx wrangler secret put <NAME>`.

### 5.3 Vercel (frontend)

1. Push the repo to GitHub.
2. In Vercel, import the repo. Set the **root directory** to `/app`, framework preset **Vite**, build command `npm run build`, output directory `dist`.
3. Add the environment variables from Section 4.1.
4. Because this is a single-page app, add a rewrite so all routes serve `index.html` (a `vercel.json` with a rewrite rule).
5. Deploy. Every push to the main branch now deploys automatically; other branches get **preview** URLs for testing.
6. Choose the project subdomain (`<project-name>.vercel.app`) or connect a custom domain.
7. Add the final URL to Firebase **authorized domains** (Step 5.1.4) and to the Worker's `ALLOWED_ORIGIN`.

Before adding any payment or ads, read Vercel's current terms for the free plan about commercial use. If commercial use is not allowed there, move the frontend to another free host or a paid plan.

## 6. Security checklist

- [ ] Firestore rules tested with two accounts.
- [ ] R2 bucket is private; files are served only through the Worker.
- [ ] Worker checks the login token on every request.
- [ ] Worker CORS allows only your frontend origin.
- [ ] Upload type and size limits are enforced **in the Worker**, not only in the frontend.
- [ ] No secrets in the repo or in `VITE_` variables.
- [ ] Preview iframe uses `sandbox="allow-scripts"` without `allow-same-origin`.
- [ ] Icons (SVG) are sanitized before use.
- [ ] Gemini key never leaves the browser and never enters the iframe.

## 7. Free-tier limits to watch

Checked when this document was written (October 2026). Re-check before launch.

| Service | What to watch |
|---|---|
| Vercel (free plan) | Bandwidth and build limits; commercial-use terms |
| Firebase (free plan) | Firestore daily reads/writes and storage; Auth is usually not the bottleneck |
| Cloudflare Workers (free) | About 100,000 requests per day |
| Cloudflare R2 (free) | About 10 GB storage and monthly operation limits; downloads (egress) are free |
| Gemini | Rate limits apply to **each user's own key** |

Tips to stay inside limits:
- Keep chat history in a subcollection and avoid reading all of it on every screen.
- Compress or resize uploaded images in the browser before upload.
- Cap the number of projects and images per user.

## 8. Release process

1. Build and test on a **preview URL** (any non-main branch on Vercel).
2. Run the saved test prompts (`/tests/prompts`) and check results.
3. Test export on a real phone using the preview URL.
4. Merge to main. Vercel deploys the frontend automatically.
5. If the Worker changed, run `npx wrangler deploy`.
6. If Firestore rules changed, run `firebase deploy --only firestore:rules`.
7. Open the live site and do one full run: sign in -> prompt -> preview -> export.

**Order matters when changing both sides:** deploy the Worker first if the frontend needs a new endpoint, and keep old endpoints working until the new frontend is live.

## 9. Rollback

- **Frontend:** in Vercel, redeploy or promote a previous successful deployment.
- **Worker:** redeploy the previous commit with `npx wrangler deploy`.
- **Firestore rules:** keep the last working rules file in git and redeploy it.
- Always tag the working version in git before a risky release.

## 10. Monitoring and backups

- Check the Vercel, Firebase, and Cloudflare dashboards weekly in the first months for usage and errors.
- Log errors in the Worker (basic request and error logs) without storing personal data.
- Backups: Firestore holds user projects. For v1, rely on provider reliability and add an export routine later. Do not store anything irreplaceable only in the browser.
- Keep the repo as the single source of truth for code, rules, and docs.

## 11. Costs

Target for v1: **zero or near zero**.

- AI cost: paid by each user through their own Gemini key.
- Hosting, auth, database, Worker, storage: free tiers.
- Possible later costs: custom domain, more storage, paid hosting if limits or terms require it.

## 12. Troubleshooting (common first-week problems)

| Problem | Likely cause |
|---|---|
| Google sign-in fails on the live site | The site's domain is missing from Firebase authorized domains |
| Worker calls blocked in the browser | `ALLOWED_ORIGIN` does not match the frontend URL exactly |
| Upload returns "unauthorized" | Missing or expired Firebase ID token in the request |
| Page refresh shows 404 on Vercel | Missing single-page-app rewrite to `index.html` |
| Environment variable seems ignored | Missing `VITE_` prefix, or the app was not redeployed after changing it |
| Exported video differs from preview | A font or image is not inlined, or the scene uses time-based code other than `t` |

## 13. Open items

- Final product name and domain (affects the URL, Firebase authorized domains, and `ALLOWED_ORIGIN`).
- Whether to use two Firebase projects (development and production) or one for v1.
- Exact upload size and file count limits.
- Where the code repository is hosted (GitHub assumed).
