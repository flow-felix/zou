# CLAUDE.md — Flow fork of **Zou** (API backend)

> Persistent workflow + deployment rules for this repo. Read before editing,
> building, or deploying. Sister doc: the Flow fork of **Kitsu** (`flow-dev/kitsu/CLAUDE.md`).

## 1. Repo purpose
Flow Animation's fork of CGWire **Zou** (Flask API + RQ jobs + event stream; the
backend behind Kitsu). Flow's backend customizations are delivered primarily as
**Zou plugins**, not by patching core Zou — see §4.

## 2. Remotes (already configured — verify, don't recreate)
```
origin    https://github.com/flow-felix/zou.git       # Flow fork
upstream  https://github.com/cgwire/zou.git           # CGWire (read-only)
```
Push only to `origin`. Never push to `upstream`.

## 3. Version state (reconciled 2026-07-01)
- **Source repo `main`/`flow/main` and production pip both at `1.0.52`.** `main`
  mirrors upstream; `flow/main` = upstream + Flow (currently no core patches, only
  this CLAUDE.md). Runtime is a clean pip `zou==1.0.52` in `/opt/zou/zouenv`.
- Flow's only live backend customizations remain the **plugins** (§4) + service/nginx
  config; core Zou is unmodified, so upstream bumps stay trivial.
- **History:** the repo previously lagged production (repo `1.0.24` vs prod pip
  `1.0.28`). On 2026-07-01 both `main`/`flow/main` were fast-forwarded to upstream
  `1.0.52` and production was upgraded to match (`pip install zou==1.0.52` +
  `zou upgrade-db`). See `CHANGELOG.md`. Keep this in lockstep on future bumps —
  sync git first (§12), then pin prod to the same version.

## 4. Flow customizations — isolation rules (plugins first)
**Strongly prefer plugins over editing core Zou** (keeps upstream upgrades trivial — prod is a clean pip install).
1. **Plugin** under `/flow-srv/zou/plugins/<name>` — the default. All current Flow backend features are plugins.
2. **Environment override** via `/etc/zou/zou.env` (config, no code change).
3. **Monkeypatch in a plugin's init** — only for behavior you cannot reach via the plugin API; isolate it in one clearly-commented module so it's auditable on upgrade.
4. **Core edit in this fork** — last resort. If unavoidable, make it a discrete `flow/<feature>` branch, mark with `# FLOW:` comments, and re-apply on each upstream bump.

### Plugins inventory (deployed at `/flow-srv/zou/plugins/`, currently NOT git repos)
| Plugin | Purpose | Upstream-conflict risk | Notes |
|---|---|---|---|
| `flowly` | Flow render/SVN/worker bridge | low (self-contained) | dev source: `flow-felix/flowly-plugin` |
| `production_assistant` | Claude agent (shells `claude -p` in worker, `exec()`s LLM code) | low-conflict but **operational risk** | runs blocking subprocess in a gevent worker; can stall API. No tracked origin. |
| `flow-client-portal` | client review portal | low | dev source has no git origin set |
| `tickets` | kitsu-tickets | low | dev origin: `cgwire/kitsu-tickets` |
| `whiteboard` | kitsu-whiteboard | low | dev origin: `INGIPSA/...` |
| `flow_schedule_hotfix` | Monkeypatch guard for empty-source schedule-version task-link copy (works around a Zou 1.0.52 core bug) | low (routes-less; delegates populated copies to upstream unchanged) | dev source: `Kitsu-Plugins/flow_schedule_hotfix`. **Temporary** — remove when upstream Zou fixes the bug. Added 2026-07-01. |

> **Known upstream issue (Zou 1.0.52):** in `schedule_service.py`, seeding a new
> production-schedule-version's task-links *from a source that has no tasks/links*
> builds `rows == []` and `insert(Model).values([])` renders a degenerate INSERT
> that omits the NOT NULL FK columns → `NotNullViolation` → HTTP 500. Symptom in
> Kitsu Forecast: a new calendar comes up empty / "changes don't save". Guarded by
> `flow_schedule_hotfix`. Copying from a *populated* source was never affected.
>
> **Known plugin issue (`flowly`):** `render_artifacts._write` does `os.replace(tmp, mp)`
> where the tmp file is sometimes already gone → recurring `FileNotFoundError` in
> `gunicorn_error.log` during render-artifact ingestion (seen ~39× post-restart on
> 2026-07-01). Pre-existing, unrelated to the version bump; fix belongs in the flowly
> plugin (make the write atomic / tolerate a missing tmp).

## 5. Production paths (exact)
| Thing | Path |
|---|---|
| Source repo | `/home/felix-eyal/flow-dev/Kitsu-Mods/zou` |
| Deployed runtime (venv) | `/opt/zou/zouenv` (Python 3.13; `pip show zou` → 1.0.52) |
| Installed package | `/opt/zou/zouenv/lib/python3.13/site-packages/zou` |
| Plugins (live) | `/flow-srv/zou/plugins/` |
| Plugin dev sources | `/flow-srv/dev/kitsu-plugins/` |
| Env / config | `/etc/zou/zou.env`, `/etc/zou/gunicorn.py`, `/etc/zou/gunicorn-events.py` |
| Logs | `/opt/zou/logs/gunicorn_{access,error}.log` |
| Backups (pip-freeze + DB dumps) | `/flow-srv/zou/backups/` (root-owned; `/flow-srv` has ~700G free — DB is only ~140MB) |
| Data / previews / tmp | `/flow-srv/kitsu/{data,previews,tmp}` (symlinked under `/opt/zou`) |
| DB | PostgreSQL `zoudb` on `localhost:5432` (user `postgres`) |

## 6. systemd services
| Unit | Role | Bind |
|---|---|---|
| `zou.service` | gunicorn API (gevent, 4 workers) | `127.0.0.1:5000` |
| `zou-events.service` | gunicorn event stream | `127.0.0.1:5001` |
| `zou-jobs.service` | RQ worker (async jobs) | — |
All use `EnvironmentFile=/etc/zou/zou.env` and run `/opt/zou/zouenv/bin/gunicorn`.
nginx (`/etc/nginx/sites-enabled/zou`) proxies `/api` → `:5000`, events → `:5001`.

## 7. Build / install steps (runtime = pip in venv, not a source build)
```bash
# Upgrade or pin Zou itself (production target version):
sudo -u zou /opt/zou/zouenv/bin/pip install "zou==<version>"
# Install a Flow plugin from its git source (do NOT hand-copy):
sudo -u zou /opt/zou/zouenv/bin/zou install-plugin /flow-srv/dev/kitsu-plugins/<plugin>
# DB migrations after a version bump:
sudo -u zou bash -c 'set -a; . /etc/zou/zou.env; /opt/zou/zouenv/bin/zou upgrade-db'
```

## 8. Deployment workflow
1. Commit + tag the plugin/source change in its git repo.
2. Back up: `pip freeze > /opt/zou/backups/pip-freeze-$(date +%F-%H%M).txt`; snapshot the plugin dir.
3. Install from git (pip pin or `install-plugin`), run migrations if needed.
4. `sudo systemctl restart zou zou-events zou-jobs`
5. Verify (§10).

## 9. Rollback workflow
```bash
# Code: reinstall the previous pinned version / previous plugin tag
sudo -u zou /opt/zou/zouenv/bin/pip install "zou==<previous>"
sudo -u zou /opt/zou/zouenv/bin/zou install-plugin <plugin-at-previous-tag>
sudo systemctl restart zou zou-events zou-jobs
# DB: restore from the pre-deploy dump if a migration must be undone:
#   pg_restore/psql from /opt/zou/backups/<dump>   (take a dump BEFORE every migration)
```
Always `pg_dump zoudb` before any `upgrade-db`.

## 10. Testing checklist (before deploy)
- [ ] Plugin/source change committed + tagged in git
- [ ] `pip freeze` snapshot + DB dump taken
- [ ] Staging or `zou test`/plugin tests pass
- [ ] After restart: `systemctl is-active zou zou-events zou-jobs` all active
- [ ] `curl -s localhost:5000/api` (or health route) OK; tail `gunicorn_error.log` clean
- [ ] Smoke a time-spent write and a plugin endpoint

## 11. Commit standards
- Upstream style `[area] summary` for core; for plugins use the plugin repo's convention.
- Core Flow patches isolated on `flow/<feature>` branches, `# FLOW:` markers in code.

## 12. Sync upstream safely
```bash
git fetch upstream --tags
git checkout main && git merge --ff-only upstream/main && git push origin main
git checkout flow/main && git merge main      # re-apply/verify any core patches here
# bump the pinned prod version only after staging validation
```

## 13. Safe-editing rules (hard rules)
1. **Never hand-edit `site-packages/zou`** — it is wiped on every `pip install`. All backend changes go through plugins or a versioned fork build.
2. **Plugins are git repos** — never edit the live copy under `/flow-srv/zou/plugins` directly; change the dev source, commit, reinstall.
3. `pg_dump` before every migration; keep a `pip freeze` per deploy.
4. Production must be reproducible from: a pinned `zou==` version + tagged plugin commits + `/etc/zou` config in git.
