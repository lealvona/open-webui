# Open-WebUI Deployment Notes

**Host:** Ms-GPT  
**URL:** https://chat.lealvona.com  
**Install type:** In-place editable install (bare metal, Python venv — not Docker)

---

## How It Runs

Managed by **systemd** as `openwebui.service`:

```
/etc/systemd/system/openwebui.service
```

The service calls `/usr/local/bin/run_openwebui.sh`, which activates the Python venv and starts the app:

```bash
cd /home/lvona/src/open-webui
source env/bin/activate
open-webui serve --port 8383
```

Runs as user `lvona`. Restarts automatically on failure.

### Managing the service

```bash
sudo systemctl status openwebui
sudo systemctl restart openwebui
sudo systemctl stop openwebui
journalctl -u openwebui -f          # live logs
```

---

## Key Paths

| Purpose | Path |
|---|---|
| App source | `/home/lvona/src/open-webui/` |
| Python venv | `/home/lvona/src/open-webui/env/` |
| Startup script | `/usr/local/bin/run_openwebui.sh` |
| Systemd unit | `/etc/systemd/system/openwebui.service` |
| Data directory | `/home/lvona/src/open-webui/backend/open_webui/data/` |
| SQLite database | `.../data/webui.db` |
| File uploads | `.../data/uploads/` |
| Vector DB | `.../data/vector_db/` |
| Frontend build output | `/home/lvona/src/open-webui/build/` |
| Nginx config | `/etc/nginx/sites-available/openwebui.conf` |
| Nginx access log | `/var/log/nginx/openwebui_access.log` |
| Nginx error log | `/var/log/nginx/openwebui_error.log` |

---

## Environment

Set in the systemd unit (`/etc/systemd/system/openwebui.service`):

| Variable | Value |
|---|---|
| `DATA_DIR` | `/home/lvona/src/open-webui/backend/open_webui/data` |
| `FRONTEND_BUILD_DIR` | `/home/lvona/src/open-webui/build` |
| `GLOBAL_LOG_LEVEL` | `DEBUG` |
| `DOCLING_OCR_ENGINE` | `easyocr` |
| `ENABLE_FORWARD_USER_INFO_HEADERS` | `True` |

Additional settings in `.env` (loaded by the app at runtime):

```
SCARF_NO_ANALYTICS=true
DO_NOT_TRACK=true
ANONYMIZED_TELEMETRY=false
CUDA_VISIBLE_DEVICES=0,1
OLLAMA_SCHED_SPREAD=1
DOCLING_OCR_ENGINE=easyocr
DOCLING_OCR_LANG=en,fr,de,es
```

---

## What Gets Served

| Component | Source |
|---|---|
| Backend (Python/FastAPI) | `backend/open_webui/` (local source, editable install) |
| Frontend (SvelteKit) | `build/` (local npm build output) |
| Data / uploads / DB | `DATA_DIR` (persisted, never replaced by installs) |

The `FRONTEND_BUILD_DIR` env var overrides the package default so the service always reads from the local `build/` directory, even though `__init__.py` would otherwise point it at the bundled package frontend.

---

## Nginx Reverse Proxy

Nginx listens on 443 (HTTPS) and proxies to `localhost:8383`.

Key settings in `/etc/nginx/sites-available/openwebui.conf`:

- `client_max_body_size 300M` — upload size limit
- `proxy_read_timeout 600s` / `proxy_send_timeout 600s` — long-running requests
- SSL via Let's Encrypt / Certbot
- Rate limit: 600 req/burst per IP, 200 concurrent connections per IP
- CrowdSec bouncer enabled (LAPI at `http://127.0.0.1:8080`, API key in `/etc/crowdsec/bouncers/crowdsec-nginx-bouncer.conf`)
- CSP `connect-src` includes `data:` — required for file uploads (frontend converts files to data URIs before fetching)

### Cache headers (post-deploy stale cache prevention)

| Path | Cache-Control |
|---|---|
| `/index.html` | `no-cache, no-store, must-revalidate` |
| `/_app/version.json` | `no-cache, no-store, must-revalidate` |
| `/_app/immutable/**` | `public, max-age=31536000, immutable` |

Browsers always re-fetch `index.html` and `version.json`, so they pick up new chunk hashes immediately after a deploy. Immutable chunks are cached for 1 year (safe because the hash in the filename changes with each build).

---

## File Uploads

- **Storage provider:** local (default)
- **Upload directory:** `/home/lvona/src/open-webui/backend/open_webui/data/uploads/`
- **Permissions:** `0775`, owned by `lvona`
- Files are renamed to `{uuid}_{original_filename}` on save
- No `RAG_FILE_MAX_SIZE` set → no application-level size cap (nginx limit of 300M applies)
- OCR via Docling with EasyOCR engine (`en`, `fr`, `de`, `es`)

---

## How to Update

> ⚠️ **Do NOT `git pull <tag>` and rebuild.** This install carries **5 uncommitted
> working-tree patches** (see below) that a pull/checkout would clobber, and Open
> WebUI **rewrites its branch history between releases** (consecutive release tags
> are not ancestors — `--ff-only`/`merge` fail). The authoritative, patch-aware
> runbook is the **`owui-update` skill** (`~/.claude/skills/owui-update/` for Claude
> Code, `~/.hermes/skills/owui-update/` for hermes agents). Summary below.
>
> ☠️ **DB-WIPE FOOTGUN — never run `open-webui …` or `import open_webui[.env]`
> without `DATA_DIR` set.** `env.py` (~line 218-238) runs an unguarded
> `copy2`+`make_archive`+`rmtree(DATA_DIR)` under `FROM_INIT_PY=true` with `DATA_DIR`
> unset — it copies a stale empty `webui.db` over the real one and deletes the data
> dir, *before* the `WEBUI_SECRET_KEY` check, so even a "failed" command destroys
> data. This wiped production twice (2026-05-12, 2026-06-19). For version checks use
> `importlib.metadata.version('open-webui')` ONLY. **Record live user/chat counts
> before AND after every update** (`sqlite3 …/webui.db "select count(*) from chat;"`).

**The 5 local patches** (uncommitted, in the working tree — saved as files under
`backups/upgrade-*/patches/`):
1. `retrieval/vector/dbs/pgvector.py` — `rollback()` in `finally` (pg idle-in-txn leak)
2. `utils/middleware.py` — all-`reasoning` stream → message (vLLM `--reasoning-parser qwen3`)
3. `utils/middleware.py` — always inject forced web-search even under native function-calling
4. `chat/Navbar.svelte` — `($banners?.length ?? 0)` null-safety
5. `apis/configs/index.ts` — `getBanners` returns `res ?? []`

**Procedure (one point release; ~5 min downtime):**

```bash
cd /home/lvona/src/open-webui
CUR=$(env/bin/pip show open-webui | awk '/^Version/{print $2}')
LATEST=$(git ls-remote --tags upstream 'v*' | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1)
git fetch upstream --tags

# 0. survey: which patched files did upstream change? new migrations? dep changes?
git diff --name-only "v$CUR" "$LATEST" | grep -Ff <(git diff --name-only)
git diff --name-status "v$CUR" "$LATEST" -- backend/open_webui/migrations/versions/ | grep '^A'
git diff "v$CUR" "$LATEST" -- pyproject.toml backend/requirements.txt

sudo systemctl stop openwebui                                    # KEEP stopped through the build (else SIGBUS)

# 1. back up: webui.db(+wal/shm), pgvector dump, unit+drop-ins, run wrapper, and fresh patch diffs
BK="backups/upgrade-${LATEST#v}"; mkdir -p "$BK/patches"
cp -a backend/open_webui/data/webui.db* "$BK/"
PGPASSWORD="$PGVECTOR_PASSWORD" pg_dump -h 127.0.0.1 -U openwebui openwebui_vec | gzip > "$BK/openwebui_vec.sql.gz"  # pw from the pgvector.conf systemd drop-in
cp -a /etc/systemd/system/openwebui.service{,.d} /usr/local/bin/run_openwebui.sh "$BK/"
for f in $(git diff --name-only); do git diff -- "$f" > "$BK/patches/$(echo "$f"|tr / _).patch"; done

# 2. RESET to the new tag (not pull/merge). Prove no local commits first.
[ "$(git rev-list --count local_copy --not --remotes=upstream)" = 0 ] || echo "LOCAL COMMITS — STOP"
git stash push -m "pre-${LATEST}-patches" -- \
  backend/open_webui/retrieval/vector/dbs/pgvector.py backend/open_webui/utils/middleware.py \
  src/lib/apis/configs/index.ts src/lib/components/chat/Navbar.svelte
git checkout -B local_copy "$LATEST"

# 3. re-apply patches (3-way), check supersede/landing, unstage, py_compile
for p in "$BK"/patches/*.patch; do git apply --3way "$p"; done
git restore --staged backend/open_webui/{retrieval/vector/dbs/pgvector.py,utils/middleware.py} \
  src/lib/{apis/configs/index.ts,components/chat/Navbar.svelte}
env/bin/python -m py_compile backend/open_webui/{utils/middleware.py,retrieval/vector/dbs/pgvector.py}

# 4. backend: re-pin + FIX version coherence (CLI reads importlib.metadata, not package.json)
env/bin/pip install -e . --no-deps                              # full `-e .` only if deps changed in step 0
#   if two open_webui-*.dist-info exist in env/.../site-packages, rm the OLD one, else app mis-reports version
FROM_INIT_PY=true env/bin/python -c "import importlib.metadata as m;print(m.version('open-webui'))"  # == $LATEST

# 5. frontend (engine-strict=true → --force)
npm install --force && npm run build
cat build/_app/version.json                                     # SHA == new HEAD

# 6. migrations may be a NO-OP (DB shared with worktrees may already be at head) — check, then start clean
export WEBUI_SECRET_KEY="$(cat .webui_secret_key)" DATA_DIR="$PWD/backend/open_webui/data" VECTOR_DB=pgvector
( cd backend/open_webui && ../../env/bin/alembic -c alembic.ini heads; ../../env/bin/alembic -c alembic.ini current )
sudo systemctl reset-failed openwebui && sudo systemctl start openwebui
```

**Verify:** `/health` 200 · `/api/version` == new · public `chat.lealvona.com/health`
200 · served `/_app/version.json` == new SHA · `pg_stat_activity` 0 idle-in-transaction
· then in the browser: a local-model stream renders, web-search globe injects, RAG
query returns, banners render. Rollback = restore `webui.db` + `git reset --hard
v$CUR` + re-apply patches (full steps in the `owui-update` skill).

Last run: **0.9.5 → 0.9.6 on 2026-06-19** — all 5 patches re-applied cleanly,
migrations were a no-op (DB already at head), backup in `backups/upgrade-0.9.6/`.

---

## Known Issues

### CrowdSec bouncer misconfiguration

The nginx error log shows this on every request:

```
failed to query LAPI ${CROWDSEC_LAPI_URL}/v1/decisions?ip=...: bad uri
```

The `CROWDSEC_LAPI_URL` environment variable is not being substituted into the nginx CrowdSec config. The bouncer is currently failing open (requests pass through). Fix: set the LAPI URL directly in the CrowdSec bouncer config file instead of relying on env var substitution.
