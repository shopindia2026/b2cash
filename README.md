# B2 CASH – setup guide

Free-to-play games with virtual points. Coins and points have no cash value. No deposits, betting, prizes or withdrawals exist anywhere in this code.

## What is in this build

| Area | Status |
|---|---|
| Responsive dark navy UI (mobile, tablet, desktop), PWA manifest, service worker | Built |
| Firebase Auth: register, login, logout, blocked-user check | Built |
| Unique UID, username, dashboard, daily reward (server-enforced, 24h) | Built |
| Game history and leaderboard (top 20, live from Firestore) | Built |
| Max 20 active games (client cap + server-enforced on enable) | Built |
| Admin panel: separate page, admin claim check, game enable/disable, block/unblock, support inbox, stats | Built |
| Server-side score submission with cooldown and bounds | Built |
| Firestore rules blocking client writes to coins, points, history | Built |
| Tic Tac Toe (vs computer) | Built |
| The other 19 games | **Not built.** Listed in the seed below but hidden until implemented (see `BUILT_GAMES` in `public/js/app.js`). |
| Multiplayer quiz rooms, Realtime Database lobby, real player counts | **Not built** |
| Notifications, announcements, homepage banners, maintenance toggle UI, profile photos, user profile page | **Not built** (the Firestore collection `config/site` already drives the maintenance banner) |

Do not ship this to real users until the not-built items are finished or removed from the nav.

## 1. Create the Firebase project

1. Go to the Firebase console and create a project.
2. Build → Authentication → Sign-in method → enable **Email/Password**.
3. Build → Firestore Database → create in production mode.
4. Project settings → General → add a **Web app**. Paste the config values into the `<script type="application/json" id="firebase-config">` block in **both** `public/index.html` and `public/admin.html` (replace every `PASTE_…`). These values are public by design; the security comes from the rules and functions.
5. Upgrade to the Blaze plan (required for Cloud Functions).

## 2. Seed the 20 game slots

In Firestore, create a collection `games`. Create one document per ID below, each with `name` (string), `icon` (emoji), `description` (string), `maxScore` (number), and `enabled` (boolean). Set `enabled: true` only for `tictactoe`; leave the rest `false`.

`quiz`, `ludo`, `numberpuzzle`, `memory`, `carracing`, `snake`, `tictactoe`, `pingpong`, `brickbreaker`, `targetchallenge`, `cardmatching`, `wordpuzzle`, `mathchallenge`, `reactiontest`, `connectfour`, `spaceshooter`, `blockpuzzle`, `colormatching`, `archery`, `sudoku`

Example for Tic Tac Toe: `tictactoe` → `{ name: "Tic Tac Toe", icon: "❌", description: "Beat the computer", maxScore: 10, enabled: true }`

Also create `config/rewards` with `{ dailyReward: 50, coinsPerWin: 5, maxScorePerSubmit: 10 }` and `config/site` with `{ maintenance: false }`.

## 3. Create the admin account and claim

1. Authentication → Users → Add user with the admin email and a strong password. Verify the email.
2. Set the admin custom claim once, from a machine with a service-account key (never commit this key):

```bash
cd functions && npm install
node -e '
const { initializeApp, cert } = require("firebase-admin/app");
const { getAuth } = require("firebase-admin/auth");
initializeApp({ credential: cert(require("./service-account.json")) });
getAuth().getUserByEmail("ADMIN@EMAIL.COM")
  .then(u => getAuth().setCustomUserClaims(u.uid, { admin: true }))
  .then(() => console.log("admin claim set")).catch(console.error);
'
```

3. Put the same email in `functions/.env` so the functions know who the admin is (Firebase deploys this file with the functions):

```
ADMIN_EMAIL=ADMIN@EMAIL.COM
```

Do not commit `.env` or `service-account.json`. Add both to `.gitignore`.

## 4. Deploy rules and functions

```bash
npm install -g firebase-tools
firebase login
firebase use YOUR_PROJECT
firebase deploy --only firestore:rules,functions
```

The first query on `gameHistory` (filtered by `uid`, sorted by `playedAt`) needs a composite index. The app's error message in the browser console includes a link that creates it in one click.

## 5. Deploy the site to Netlify

1. Push this folder to a Git repository.
2. Netlify → Add new site → Import from Git. `netlify.toml` already sets `publish = "public"`, so no build command is needed.
3. Deploy. Visit `/` for the app and `/admin` for the admin panel.
4. Add your Netlify domain to Firebase → Authentication → Settings → Authorized domains.

## 6. Install as an app (PWA)

Open the site in Chrome on Android and choose "Install app". The service worker caches the shell only; live Firebase data is never cached.

## Security model

- Coins and points change only inside Cloud Functions (`claimDailyReward`, `submitScore`). Firestore rules reject any client write to those fields.
- Admin actions require the `admin` custom claim and the admin email, checked in the functions and again in the rules. Hiding the admin page is not the protection.
- Score submissions are bounded by `maxScore`, rate-limited per user, and only accepted for enabled games.
- The Content-Security-Policy in `netlify.toml` blocks inline scripts and unknown hosts.
- No admin credentials or service-account keys are in the frontend.

## Before launch

- Replace every `YOUR_…` placeholder.
- Add your real privacy and terms text (the current pages are short placeholders).
- Finish or remove the not-built items above.
- Set Firebase budget alerts.

## Publish on GitHub (public repo)

Before the first push, check that no secret file is staged:

```bash
git status            # functions/.env and service-account*.json must NOT appear
git check-ignore -v functions/.env functions/service-account.json
```

Then:

```bash
git init && git add . && git commit -m "Initial commit: B2 CASH"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/b2cash.git
git push -u origin main
```

Safe to make public: the Firebase web config, all source code, and `firestore.rules`. Never make public: `functions/.env`, any `service-account*.json`, or the admin password. If a secret is ever pushed by mistake, rotate it in Firebase immediately; deleting it from later commits is not enough.

Licensed under MIT (see `LICENSE`).
