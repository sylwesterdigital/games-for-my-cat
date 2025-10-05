# Games for my cat

## catch the tail
Live Demo:
https://helena.social/static/games/catchtail

https://github.com/user-attachments/assets/d734b2a8-8543-4c7a-89f7-c82850125a28

# Orbit & Hit — WebGL2 Rainbow

Fast, tiny, and flashy. **Orbit & Hit** is a one-file WebGL2 skill game with an HMAC-secured global leaderboard. No engines, no bundlers, no node_modules — just modern browser APIs and a lean Flask backend.

<img src="docs/catchtail.gif" alt="Gameplay preview" width="600"/>

## Why this is cool

* **Zero-dependency WebAudio SFX** — microscopic synth, no assets.
* **WebGL2 shader** renders the rainbow ring + orb at 60+ FPS.
* **Tight game loop** with precise hit windows and difficulty curve.
* **Global leaderboard** with HMAC tokens, nonce replay protection, and per-email claim binding.
* **Local persistence** via IndexedDB for profile, avatar, and results.
* **Responsive UI** with lightweight, clean CSS. Works on desktop & mobile.

---

## Gameplay

* Hit when the moving dot overlaps the target ring.
* Goal: **50** hits.
* Difficulty ramps automatically; rebounds flip direction.
* Controls: **Space / Click / Tap** to hit, **P** pause, **R** restart.

---

## Tech Overview

* **Frontend:** Plain HTML/CSS/JS + WebGL2 shader + WebAudio.
* **Persistence:** IndexedDB (`profile`, `results`).
* **Backend:** Flask + SQLite (`games.db`).
* **Security:** HMAC-signed submit tokens, nonce replay blocklist, user claim per email (first clientId wins).

---

## Project layout

```
.
├─ index.html              # Game + UI + client logic
├─ docs/catchtail.gif      # README/demo
├─ server/                 # Flask backend (leaderboard)
│  ├─ app.py               # Flask app + endpoints
│  └─ requirements.txt     # Flask, gunicorn, limiter, etc.
├─ deploy.sh               # rsync + systemd restart + SQLite perms
└─ README.md               # This file
```

> If your paths differ a bit, the commands below still apply: you need `index.html` served alongside the Flask API.

---

## Run locally (single path that works)

1. **Python env**

```bash
cd server
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

2. **Environment**
   Create `server/.env`:

```bash
SUBMIT_HMAC_SECRET=replace_with_long_random_string
FLASK_ENV=development
PYTHONUNBUFFERED=1
```

3. **Start Flask**

```bash
# from server/
python app.py
# or: gunicorn -b 0.0.0.0:1111 'app:app' --workers 2
```

4. **Serve the game**
   Open a static server at repo root so the page and the API share origin/port (CORS-free). Easiest way:

```bash
# from repo root (one level above server/)
python3 -m http.server 1111
```

Now visit: `http://localhost:1111/`
This serves `index.html` and the Flask API at `/api/...` on the same port. (If you prefer, reverse-proxy Flask behind the same port; the point is: same origin.)

---

## Environment variables (backend)

| Name                 | Purpose                                          |
| -------------------- | ------------------------------------------------ |
| `SUBMIT_HMAC_SECRET` | HMAC key for token signing/verify (must be set). |
| `APP_PIN`            | Admin/maintenance pin if your app.py expects it. |
| `FLASK_ENV`          | `production` or `development`.                   |
| `PYTHONUNBUFFERED`   | Recommended `1` for line-buffered logs.          |

---

## Leaderboard API

All routes are under `/api`.

* `POST /api/submit_token`
  **Input:** `{ clientId, email }`
  **Output:** `{ status:"ok", token, exp }`
  Binds the email to the first clientId that claims it. Refreshes `last_seen`.

* `POST /api/submit_result_public`
  **Headers:** `X-Client-Id`
  **Body:** `{ token, nickname, email, clientId, hitsMade, target, avgPrecision, outcome, date, durationMs }`
  Validates HMAC token, checks nonce replay, checks email ownership, then inserts a row in `games`.

* `GET /api/leaderboard?limit=10`
  Returns top unique user scores (best row per user_key) with ranks.

* `GET /api/leaderboard/me`
  Returns current player’s best ranked item. (Implemented server-side; used by the footer “Player: …” line.)

**SQLite tables:**
`games(hits_made,target,avg_precision,outcome,duration_ms,created_at,...)`
`users(email UNIQUE,user_key,client_id,nickname,created_at,last_seen)`
`used_nonces(nonce PRIMARY KEY,seen_at)`

---

## Deployment (rsync + systemd)

A ready script is included. It pushes code, writes `.env`, ensures DB perms, and restarts the unit.

```bash
./deploy.sh -pin YOURPIN -hmac YOUR_LONG_RANDOM_SECRET
```

Flags you’ll actually use:

* `--fix-perms` — ensure directory/`games.db` perms for the app user.
* `--clean-game` — **hard reset** DB: backup then recreate file.
* `--clean-game-soft` — **soft reset**: delete rows, keep schema/file.
* `--no-restart` — push files without bouncing the service.

**What it syncs:** code only. It **does not** ship `node_modules` (there are none), venv, `games.db`, or Git metadata.

---

## Troubleshooting

* **Footer shows “Player: unranked” forever**
  Make sure you saved a **nickname + email** in Profile (top-right avatar), then play at least one round that submits successfully. The footer reads `/api/leaderboard/me`.

* **500 on `/api/leaderboard/me`**
  Backend must be running with a valid `SUBMIT_HMAC_SECRET`, and the DB schema must be initialized by the app at start. Check logs; fix the env; restart.

* **SQLite WAL/SHM permission issues**
  Run: `./deploy.sh --fix-perms` or manually `chown www-data:www-data games.db && chmod 660 games.db` and remove `games.db-wal`/`games.db-shm`.

---

## License

MIT for the game and backend. If you use the leaderboard model elsewhere, keep the HMAC/nonce credit and please don’t ship weak secrets.

---

## Credits

* Core idea, shader, synth, and gameplay loop in this repo.
* You, for shipping it without bloat. 🚀

---

### Quick checklist

* [ ] Set `SUBMIT_HMAC_SECRET` in `.env`
* [ ] Run Flask (`python server/app.py` or gunicorn)
* [ ] Serve `index.html` on the same origin/port
* [ ] Save profile (nickname + email) in the UI
* [ ] Play a round → see yourself on the global board

Happy orbiting.
