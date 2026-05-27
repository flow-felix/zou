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

## 3. ⚠️ Current divergence (must reconcile — see migration plan)
- **Source repo is at `1.0.24`** (`zou/__init__.py`), `main` mirrors upstream.
- **Production runs stock pip `zou==1.0.28`** in the venv (`/opt/zou/zouenv`) — *not* built from this repo, and **no in-place edits** to site-packages were found (all files share the install timestamp).
- Therefore: today, Flow's only live backend customization is the **plugins** + service/nginx config, none of which is committed here.
- Reconciliation: bump this fork's `main`/`flow/main` to the `v1.0.28` upstream tag so the repo matches production, then manage plugins + any core patches from git.

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

## 5. Production paths (exact)
| Thing | Path |
|---|---|
| Source repo | `/home/felix-eyal/flow-dev/Kitsu-Mods/zou` |
| Deployed runtime (venv) | `/opt/zou/zouenv` (Python 3.13; `pip show zou` → 1.0.28) |
| Installed package | `/opt/zou/zouenv/lib/python3.13/site-packages/zou` |
| Plugins (live) | `/flow-srv/zou/plugins/` |
| Plugin dev sources | `/flow-srv/dev/kitsu-plugins/` |
| Env / config | `/etc/zou/zou.env`, `/etc/zou/gunicorn.py`, `/etc/zou/gunicorn-events.py` |
| Logs | `/opt/zou/logs/gunicorn_{access,error}.log` |
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
