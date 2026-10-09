# Split the G

Mobile web app (PWA) for tracking "splitting the G" — a Guinness game where the first sip should leave the foam line across the G in the GUINNESS logo. Built for a group of mates. Owner is Joshua, who works from his phone with no local Node setup.

## Stack
- Single `index.html` — vanilla JS in one `<script type="module">`, all CSS inline. No build step.
- Firebase Firestore (Spark/free plan) via CDN imports (firebase 10.12.0). Config lives in `index.html`.
- No Firebase Storage (needs a paid plan). Photos are compressed client-side (600px, JPEG q0.55) and stored as base64 inside Firestore docs.
- Hosting: GitHub Pages from `main` — https://salvationbacon.github.io/Slpittheg/ (pushing to `main` deploys). Old Netlify site is no longer updated.

## Files
- `index.html` — the whole app
- `apple-touch-icon.png`, `apple-touch-icon-precomposed.png` (180px), `icon.png` (1024px) — home screen icons
- `img-header.png`, `img-splash.png`, `img-empty.png`, `img-celebrate.png` — UI art
- `firestore.rules` — security rules to paste into the Firebase console (not deployed automatically)
- Image paths in `index.html` are relative (`img-...`) so the app works on both Netlify and a GitHub Pages project URL.

## Firestore structure
```
groups/{groupCode}              { name, createdAt, leaderDeviceId, leaderName }   shared leader (older groups lack this doc until their leader opens the app)
groups/{groupCode}/players      { name, avatar, createdAt }
groups/{groupCode}/splits       { playerId, playerName, timestamp, photo, score, analysis }        confirmed
groups/{groupCode}/pending      { playerId, playerName, timestamp, photo, submittedBy, submitterName, votes:{deviceId:'yes'|'no'}, score, analysis }
groups/{groupCode}/rejected     { playerId, playerName, timestamp, photo, score, analysis, rejectedBy:'vote'|'leader' }
groups/{groupCode}/leadervotes  { nomineeDeviceId, nomineeName, timestamp, votes }
```
`score` is 0–100 or null (older splits / "no G in photo"). `analysis` = `{x0,y0,x1,y1,line,auto,adjusted}` as fractions of the image size (G box + foam line).
Root-level `/players` is orphaned data from an early version — ignore it.
Do NOT change this structure; live groups (e.g. `theboys156`) depend on it. Adding fields is fine.

## localStorage keys
- `splitg-device-id` — random per-device ID used for votes
- `splitg-groups` — groups this device has joined
- `splitg-active` — current group
- `splitg-leaders` — cache of groups where this device is leader (truth is `leaderDeviceId` on the group doc; also used once to migrate old groups)
- `splitg-me` — `{groupCode: playerId}`, which player this device is
- `splitg-cache-{code}` — cached players/splits (no photos) for instant load
- `splitg-install-dismissed`

## Features / rules
- Create group → code = slugified name + 3 random digits (checked for collisions); creator becomes leader via the group doc.
- Join group by code — refused if no group doc and no players exist. Active group code is also kept in the URL hash so iOS "Add to Home Screen" remembers it, and `#code` links work as invite links (Share Invite Link button on Players tab).
- Each device picks "which player are you?" once per group (board card / Players tab). Used for `submitterName`, `nomineeName`, `leaderName`.
- Settings drawer: switch/add/remove groups, nominate self as leader (or claim it directly if the group has no leader).
- All vote resolution (splits and leader) runs in Firestore transactions so simultaneous votes can't double-count.
- Logging a split requires a photo. It goes to `pending`; the submitter cannot vote. Majority of the other players (`votesNeeded()` = `floor((players-1)/2)+1`) confirms → moved to `splits`; majority no → `rejected`.
- **G score:** after picking the photo the user drags a box around the G. `detectLine()` finds the foam line (first row, going down, where dark stout pixels appear and stay — robust to the letter's own strokes) and `splitScore()` = 100% at the middle of the G, 0% at its top/bottom edge or beyond. The line can be dragged by hand; that sets `adjusted` and vote cards show "✋ line moved by hand". Overlay (box, dashed target, foam line, score chip) shown on vote cards and the full-screen viewer; score badge in Log; best score per player on Board; score in the celebration. Fully automatic G detection would need a paid vision API — deliberately not used.
- Log shows confirmed and rejected entries (rejected greyed out, struck through).
- Only the leader sees delete buttons. Every delete has a confirm dialog.
- Shared rankings: ties share a place, next rank skips (1, 1, 3).
- Listener renders are debounced (`scheduleRender`) to stop duplicate cards. `renderPlayers` preserves the typed name across live re-renders.

## Known issues / TODO
1. Firestore test-mode rules expired (Oct 2026) — every read/write returns permission-denied, which breaks everything. Fix: paste `firestore.rules` into Firebase console → Firestore → Rules → Publish. The rules are open (anyone with a code can read/write); acceptable for mates, revisit if needed.
2. Photos as base64 in Firestore cost reads/egress on every app open. Fine for one group; revisit if usage grows.
3. Devices are identified by a random localStorage ID — clearing browser data / switching phones loses leadership and vote identity. Another member can win a leader vote to recover.

## Working conventions
- Keep it a single static `index.html` that can be deployed by uploading files — the owner deploys from his phone.
- Owner prefers short, direct replies and seeing changes shipped rather than explained.
