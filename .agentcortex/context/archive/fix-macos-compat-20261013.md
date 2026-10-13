# Work Log: fix/macos-compat

## Header

- Branch: `fix/macos-compat`
- Classification: `quick-win`
- Classified by: `Claude`
- Frozen: `true`
- Created Date: `2026-10-13`
- Owner: `nalbam`
- Guardrails Mode: `Quick`
- Current Phase: `ship`
- Diff Base SHA: `460d71db3b41746748e53e881d7ecd08e1207f60`
- Checkpoint SHA: `460d71db3b41746748e53e881d7ecd08e1207f60`
- Recommended Skills: `none`
- Primary Domain Snapshot: `office-runtime`
- SSoT Sequence: `148`

---

## Session Info

- Agent: `Claude Opus 5.5`
- Session: `2026-10-13T06:50Z`
- Platform: `claude-code`
- Files Read: `12`
- Override: none
- Downstream-Capabilities: none
- Context Read Receipt: `current_state.md` → Update Sequence 148, Last Verified 2026-10-11; Work Log → created; Spec Scope → `docs/specs/engineering-audit-remediation.md` (2026-09-26 Dev-Server Parity Wave).

---

## Task Description

Owner (new fork, macOS 15 / Node 24) asked to analyze the project, adapt it to their environment and start it. The baseline test run on this machine failed `tests/viteDevServerParity.test.js` R-3 on every run: the dev server's file-watcher fallback fabricated a `source: 'file-watcher'` agent from a write inside `OFFICE_STATUS_DIR` when that directory is reached through a symlink (macOS temp dir). Fix the comparison in `vite.config.mjs` so the exclusion holds for symlinked paths. The same pass found `.agentcortex/bin/validate.sh` aborting on macOS (`head: illegal line count -- -1`, GNU-only `head -n -1`), so the repo's required validator could not finish; make that line portable. Chain: `/plan → /implement → /ship`.

---

## Phase Sequence

| Phase | Status | Entered | Notes |
|---|---|---|---|
| bootstrap | completed | 2026-10-13T06:50:18Z | quick-win: 2 modules (dev-server config, validator), one line of logic each |
| plan | completed | 2026-10-13T06:52:00Z | realpath compare in `isUnderStatusDir`; `sed '$d'` in the validator's M8 check |
| implement | completed | 2026-10-13T06:54:00Z | 2 code files + 1 spec section |
| review | skipped | — | quick-win, evidence inline |
| test | skipped | — | quick-win, evidence inline |
| handoff | skipped | — | quick-win exempt |
| ship | completed | 2026-10-13T07:00:00Z | closure in same PR |

---

## Phase Summary

- plan: Target `vite.config.mjs` (`fileWatcherFallbackPlugin` → `isUnderStatusDir`) + `docs/specs/engineering-audit-remediation.md` (new follow-up section). Step: compare `fs.realpathSync.native` of the file and of the status directory, resolved per event. Risk: a realpath syscall per in-project change event (negligible; only after `isUnderProjectRoot` passes). Rollback: revert the commit. Confidence: 95% — root cause reproduced with a path probe before the change.
- implement: `isUnderStatusDir` now compares real paths; unresolvable paths keep the previous `path.resolve` comparison. Spec follow-up section added. `validate.sh` M8 check drops the count line with `sed '$d'` instead of `head -n -1`.
- ship: PASS — closure (archive + INDEX chain + Ship History + rotation) committed in this PR.

⚡ ACX

---

## Gate Evidence

- Gate: bootstrap | Verdict: PASS | Classification: quick-win | Timestamp: 2026-10-13T06:50:18Z
- Gate: plan | Verdict: PASS | Classification: quick-win | Timestamp: 2026-10-13T06:52:00Z
- Gate: implement | Verdict: PASS | Classification: quick-win | Timestamp: 2026-10-13T06:54:00Z
- Gate: ship | Verdict: PASS | Classification: quick-win | Timestamp: 2026-10-13T07:00:00Z

---

## External References

| Type | Path / URL | Notes |
|---|---|---|
| Spec | docs/specs/engineering-audit-remediation.md | `## 2026-10-13 Symlinked Status-Dir Follow-Up` (extends the 2026-09-26 Dev-Server Parity Wave, R-3) |
| ADR | — | — |
| Issue | — | — |
| PR | — | — |

---

## Known Risk

- Root Cause: `isUnderStatusDir` compared the as-given `OFFICE_STATUS_DIR` string against watcher paths, but Vite realpaths its root and the watcher reports the realpath spelling for files under it, so a symlinked status directory never matched.
- Rollback plan: revert the single commit; the change is confined to one helper in dev-only build tooling (`vite.config.mjs` is never bundled). Detection: `tests/viteDevServerParity.test.js` R-3.
- `validate.sh` is framework-managed (`core` in `.agentcortex-manifest`, hash not updated): the file now differs from the shipped hash and a framework update will restore the GNU-only line unless fixed upstream. `validate.ps1` is unaffected (no `head`).
- Residual: `isUnderProjectRoot` still compares strings. Not changed — Vite hands it an already-realpathed root and the watcher reports in-root files in that spelling, so no failing case was found.

---

## Decisions

none

---

## Conflict Resolution

none

---

## Skill Notes

none

---

## Drift Log

- The code edit preceded the Work Log: the defect surfaced during the baseline analysis run and the fix was applied and verified before bootstrap paperwork was written. Receipts are recorded in phase order; no gate was skipped.
- Scope widened once before ship (validator fix added, branch renamed from `fix/macos-symlink-status-dir`); still 2 modules, classification unchanged. No spec covers `validate.sh` in this repo, so that change is recorded in Ship History only.
- `Last Verified` refreshed at ship together with the Ship History write (one guarded write instead of two).

---

## Review Feedback

none

---

## Red Team Findings

none

---

## Design Reference

none

---

## Observability

none

---

## Resume

none

---

## Test Gate Results

none

---

## Evidence

- Before (macOS 15, Node v24.21.0, main @ 460d71d): `npx vitest run tests/viteDevServerParity.test.js` → `R-3: OFFICE_STATUS_DIR nested INSIDE the project root is still excluded` FAIL, `expected '{"_seq":…,"source":"file-watcher","_hint":"no-hooks"}' to be 'null'` (3/3 runs).
- Probe (Vite `createServer`, temp root): `given root /var/folders/…`, `config.root /private/var/folders/…`, change events for one write: `/private/var/…/.status-nested/debug-log.txt` and `/var/…/.status-nested/debug-log.txt`.
- After: same command → `Tests 11 passed (11)`, 5/5 runs.
- Full suite after `npm run build`: `Test Files 152 passed (152)` / `Tests 2719 passed (2719)`, 2/2 runs.
- Behavior statement: before, a write to a non-source file inside a symlinked in-project `OFFICE_STATUS_DIR` produced a fake working agent; after, `/api/status` stays `null`.
- Validator before (main @ 460d71d, macOS): `bash .agentcortex/bin/validate.sh` → `head: illegal line count -- -1`, exit 1, no `Summary:` line. Trigger: `archive/codex-main-codex-routes.md` has 5 broken relative links, so the `head -n -1` branch runs every time.
- Validator after: `Summary: pass=96 warn=5 fail=0 skip=6` / `Agentic OS integrity check passed`, exit 0.
