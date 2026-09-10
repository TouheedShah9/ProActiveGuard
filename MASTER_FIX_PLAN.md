# ProactiveGuard v2.0 — Master Integration & Hardening Plan

Purpose: this is the working checklist for turning ProactiveGuard from
"8 impressive standalone scripts" into one real, integrated system with
a single source of truth for data and logic. Each phase has a done
condition. We do not mark a phase done until its done condition is met
and verified (compiled + traced, and re-verified once you can run it
locally with all packages installed).

Status legend: [ ] not started · [~] in progress · [x] done & verified

---

## PHASE 0 — Foundation fixes (blockers)
Done condition: a clean `pip install -r requirements.txt` followed by
running each module in order does not crash on missing deps, and no
secrets are exposed.

- [x] Add missing `matplotlib` to requirements.txt (used by 5 of 8
      modules; without it, model.py/digital_twin.py/rl_scheduler.py/
      safety_infrastructure.py/zone_engine.py crash on import).
- [x] Confirm `.gitignore` correctly excludes `.env`, `*.db`, `models/`,
      `.streamlit/secrets.toml` (already correct — no change needed).
- [ ] YOU: rotate the Groq API key that was in the uploaded `.env`
      (it was exposed in a chat upload — treat it as burned).

## PHASE 1 — Single source of truth for domain data
Done condition: worker roster and site definitions exist in exactly
ONE place. No module hand-declares its own copy of WORKERS/SITES.

Problem found: `v2_data_engine.py`, `v2_llm_engine.py`, and
`v2_dashboard.py` each independently hard-code the same 10-worker
roster (name, resting_hr, max_hr, baseline_temp, accl_days...). Same
for the 3-site definitions in `v2_data_engine.py` and
`v2_zone_engine.py`. Three copies that can silently drift out of sync
is not a "real" system — it's three demos that happen to agree today.

- [x] Create `v2_config.py` — single canonical definition of `WORKERS`
      and `SITES`.
- [x] Refactor `v2_data_engine.py` to import from `v2_config`.
- [x] Refactor `v2_llm_engine.py` to import from `v2_config`.
- [x] Refactor `v2_dashboard.py` to import from `v2_config`.
- [x] Refactor `v2_zone_engine.py` to import `SITES` from `v2_config`.

## PHASE 2 — Wire the orphaned modules into the dashboard for real
Done condition: the dashboard imports and calls the real functions
from `v2_llm_engine.py` and `v2_zone_engine.py` — no more duplicate,
simplified, inline reimplementations living inside the dashboard file.

- [x] Dashboard imports `generate_groq_explanation` and
      `generate_rule_based_explanation` from `v2_llm_engine.py`
      instead of its own hand-rolled `get_llm_explanation()`.
- [x] Dashboard imports zone WBGT + coordinated-risk logic from
      `v2_zone_engine.py` instead of its own inline
      `ZONE_WBGT_OFFSET` heuristic.
- [x] Dashboard imports `init_audit_database` from
      `v2_safety_infrastructure.py` and calls it once at startup —
      audit panel no longer silently no-ops when `data/audit.db`
      doesn't exist yet.
- [x] Dashboard imports `generate_groq_explanation` and
      `generate_rule_based_explanation` from `v2_llm_engine.py`
      instead of its own hand-rolled `get_llm_explanation()`.
- [x] Dashboard imports zone WBGT + coordinated-risk logic from
      `v2_zone_engine.py` instead of its own inline
      `ZONE_WBGT_OFFSET` heuristic.
- [x] Dashboard imports `init_audit_database` from
      `v2_safety_infrastructure.py` and calls it once at startup —
      audit panel no longer silently no-ops when `data/audit.db`
      doesn't exist yet.
- [x] Dashboard imports RL scheduler's `generate_proactive_schedule()`
      and surfaces a real 2-hour proactive schedule in an expander
      next to the live single-tick action recommendation (previously
      this function was computed by `v2_rl_scheduler.py` but never
      called anywhere in the live app).
- [x] `v2_llm_engine.py`'s own `detect_zone_event()` was a THIRD
      independent reimplementation of "3+ workers spiking = zone
      event" (dashboard had one, `v2_zone_engine.py` had one). It now
      delegates to `v2_zone_engine.detect_coordinated_risk()` and
      reshapes the output to keep `generate_zone_report()` working
      unchanged.
- [x] Found and fixed a 6th independent copy of the worker roster
      hiding in `v2_rl_scheduler.py` (`WORKER_PROFILES`) — also
      switched to the shared `v2_config.py` import.

## PHASE 3 — Fail loudly, not silently
Done condition: no bare `except Exception: pass` blocks hiding real
errors from you during development. Errors get logged, not swallowed.

- [x] Audit-trail read/write functions (`get_audit`, `log_alert`) now
      show a `st.warning` with the real exception instead of silently
      returning `[]` / doing nothing.
- [x] `load_models()` now returns a `load_warnings` list explaining
      exactly why it fell back to rule-based mode (missing model
      files vs. a real exception vs. missing RL agent), and the
      dashboard displays these warnings on load instead of hiding the
      degraded state.
- [x] Found and fixed a real bug while doing this: `load_models()`
      was being called **three separate times** per Streamlit rerun
      (re-deserializing the XGBoost model, SHAP explainer, and RL
      Q-table from disk on every single UI interaction and every
      simulation tick). Added `@st.cache_resource` so it loads once
      per session, and the two redundant calls now reuse the already-
      loaded `M` dict.
- [x] Found and fixed a real regression risk before it shipped:
      `v2_zone_engine.py`, `v2_safety_infrastructure.py`, and
      `v2_rl_scheduler.py` (via implicit import when unpickling the
      RL agent) all call `np.random.seed(42)` at import time for
      their own offline reproducibility. Left unhandled, importing
      them into the dashboard would have made the "live" simulation
      replay identically every restart. Added explicit re-randomization
      after all imports resolve, including the implicit one triggered
      by unpickling the RL agent.

## PHASE 4 — Verify against real generated data (not the demo stub)
Done condition: every module runs against the actual CSV/pickle files
produced by the previous module in the chain — not hardcoded demo
dictionaries.

- [x] `v2_llm_engine.py`'s `run_demo()` replaced with a real demo that
      loads the last row of `data/v2_multisite_data.csv` +
      `data/v2_twin_predictions.csv` + the trained model's SHAP output,
      instead of a hand-typed dictionary of fake vitals.
- [ ] YOU (needs your machine — see below): run the full chain
      data_engine → digital_twin → model → zone_engine → rl_scheduler →
      safety_infrastructure → llm_engine → dashboard, in that order,
      and confirm no runtime errors. This sandbox has no internet
      access, so I cannot install streamlit/xgboost/shap/optuna/groq/
      ephem/imbalanced-learn here to execute it myself.

## PHASE 5 — Deployment readiness
Done condition: repo is push-ready and deployable to a server.

- [ ] Write final `README.md` (architecture diagram, setup, results
      table, limitations section).
- [ ] Add a `run_pipeline.sh` / `Makefile` that runs all modules in the
      correct dependency order with one command.
- [ ] Add basic `.streamlit/config.toml` (theme + server settings) —
      currently the `.streamlit` folder uploaded was empty.
- [ ] Confirm the app runs behind `streamlit run v2_dashboard.py
      --server.port 8504` cleanly on a clean venv.

---

## What "done" will look like
One command builds the data, one command trains the model, one
command launches a dashboard that pulls from the *same* worker
config, the *same* zone engine, and the *same* LLM engine as the
standalone scripts — because it's literally importing them, not
re-describing them.
