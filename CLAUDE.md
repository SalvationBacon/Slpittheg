# Split the G

Mobile web app (PWA) for tracking "splitting the G" — a Guinness game where the first sip should leave the foam line across the G in the GUINNESS logo. Built for a group of mates. Owner is Joshua, who works from his phone with no local Node setup.

## Stack
- Single `index.html` — vanilla JS in one `<script type="module">`, all CSS inline. No build step.
- Firebase Firestore (Spark/free plan) via CDN imports (firebase 10.12.0). Config lives in `index.html`.
- No Firebase Storage (needs a paid plan). Photos are compressed client-side (600px, JPEG q0.55) and stored as base64 inside Firestore docs.
- Hosting: Netlify free (credit-metered — every production deploy costs credits). Plan is to move to GitHub Pages from this repo.

## Files
- `index.html` — the whole app
- `apple-touch-icon.png`, `apple-touch-icon-precomposed.png` (180px), `icon.png` (1024px) — home screen icons
- `img-header.png`, `img-splash.png`, `img-empty.png`, `img-celebrate.png` — UI art
- Image paths in `index.html` are root-relative (`/img-...`). If hosting on a GitHub Pages project URL (`user.github.io/repo/`), change them to relative (`img-...`) or they will 404.

## Firestore structure
```
groups/{groupCode}/players      { name, avatar, createdAt }
groups/{groupCode}/splits       { playerId, playerName, timestamp, photo }        confirmed
groups/{groupCode}/pending      { playerId, playerName, timestamp, photo, submittedBy, submitterName, votes:{deviceId:'yes'|'no'} }
groups/{groupCode}/rejected     { playerId, playerName, timestamp, photo, rejectedBy:'vote'|'leader' }
groups/{groupCode}/leadervotes  { nomineeDeviceId, nomineeName, timestamp, votes }
```
Root-level `/players` is orphaned data from an early version — ignore it.
Do NOT change this structure; live groups (e.g. `theboys156`) depend on it.

## localStorage keys
- `splitg-device-id` — random per-device ID used for votes
- `splitg-groups` — groups this device has joined
- `splitg-active` — current group
- `splitg-leaders` — groups where this device is leader
- `splitg-cache-{code}` — cached players/splits (no photos) for instant load
- `splitg-install-dismissed`

## Features / rules
- Create group → code = slugified name + 3 random digits; creator becomes leader (stored locally only).
- Join group by code. Active group code is also kept in the URL hash so iOS "Add to Home Screen" remembers it.
- Settings drawer: switch/add/remove groups, nominate self as leader.
- Logging a split requires a photo. It goes to `pending`; the submitter cannot vote. Majority of the other players (`floor((players-1)/2)+1`) confirms → moved to `splits`; majority no → `rejected`.
- Log shows confirmed and rejected entries (rejected greyed out, struck through).
- Only the leader sees delete buttons. Every delete has a confirm dialog.
- Shared rankings: ties share a place, next rank skips (1, 1, 3).
- Listener renders are debounced (`scheduleRender`) to stop duplicate cards.

## Known issues / TODO
1. **Create/Join buttons reported unresponsive.** Latest change wires them with `addEventListener` at the end of the script. Unverified — may have been a stale Netlify deploy (credit limit hit) rather than a code bug. Test this first.
2. **Leader vote bug:** in `castLeaderVote`, when the majority is reached it grants leadership to the *voter's* device (`deviceId`, `isLeader = true`) instead of the nominee (`data2.nomineeDeviceId`). The Firestore write also uses a broken `updateDoc` → `addDoc` fallback. Store leadership at a fixed location (e.g. field `leaderDeviceId` on `groups/{code}` via `setDoc` with merge) and derive `isLeader` from it on every device.
3. `nomineeName` is a placeholder `'(you)'` — devices aren't linked to player names. Consider a one-time "which player are you?" prompt per group; that also fixes #4.
4. `submitterName` is set to the split's player name, not the person who actually submitted.
5. Firestore test-mode rules expire 30 days after database creation. Rules should allow read/write on `groups/{group}/{document=**}`.
6. Photos as base64 in Firestore cost reads/egress on every app open. Fine for one group; revisit if usage grows.

## Working conventions
- Keep it a single static `index.html` that can be deployed by uploading files — the owner deploys from his phone.
- Owner prefers short, direct replies and seeing changes shipped rather than explained.
