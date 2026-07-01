# Flow Zou — Deployment Changelog

Production deploy history for the Flow fork of Zou. Newest first.
Production runtime is a pinned pip install in `/opt/zou/zouenv` (not a source
build); Flow backend logic lives in plugins under `/flow-srv/zou/plugins`. Each
entry records the pinned version, the pre-deploy DB dump + pip freeze, and the
exact rollback command.

## 2026-07-01 — Upstream sync to CGWire 1.0.52

- **git `flow/main`:** `c73f51724d0339e39f634278ea056e3acda6dfa5` (mirrors upstream `1.0.52`)
- **Pinned version:** `1.0.28` → **`1.0.52`** (`pip install zou==1.0.52`)
- **DB migration:** `zou upgrade-db` — ~13 Alembic migrations applied (EXIT 0),
  e.g. project template tables, playlist sharing, drop legacy `asset_types`,
  `for_client` flag on comments, `country` on person, salary-scale cascade delete.
- **Pre-deploy backups:**
  - DB dump (custom fmt, 27 MB): `/flow-srv/zou/backups/zoudb-20260701-174006-pre-1.0.52.dump`
  - pip freeze (records `zou==1.0.28`): `/flow-srv/zou/backups/pip-freeze-20260701-174006-pre-1.0.52.txt`
- **Services restarted:** `zou zou-events zou-jobs` — all active; `/status` reports
  `version 1.0.52` with database/key-value/event-stream/job-queue/indexer all up.
- **Notes:** One transient `flowly` plugin `FileNotFoundError` (render-artifact tmp
  file race) fired once on RQ-worker restart and did not recur — unrelated to the
  version bump. Leftover `~ou`/`~crypt` pip dirs from a prior interrupted install
  were cleaned. No core Zou patches in this fork; all Flow logic remains in plugins.
- **Scope:** Backend. Deployed alongside the Kitsu 1.0.21 → 1.0.48 build the same
  day (kept as a matched pair).
- **Rollback:**
  ```bash
  # Code:
  sudo -u zou /opt/zou/zouenv/bin/python -m pip install "zou==1.0.28"
  # DB (only if the schema migration must be undone — restores pre-deploy state):
  sudo -u postgres pg_restore --clean --if-exists -d zoudb /flow-srv/zou/backups/zoudb-20260701-174006-pre-1.0.52.dump
  sudo systemctl restart zou zou-events zou-jobs
  ```
