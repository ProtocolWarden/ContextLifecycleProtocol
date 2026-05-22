# Log

## 2026-05-22 — Manifest concept + CL/Manifest/RepoGraph contract split (design session)

Walked the architecture for how CL should integrate with executors and where cognition state actually lives. Result: a new repo-type vocabulary and a clean three-way contract split.

**Repo type: "manifest"** — locked in as a first-class repo category. A manifest is a repo whose job is to (1) declare what's in an ecosystem, (2) host that ecosystem's cognition state, and (3) anchor sessions that operate on it. Current instances: `PlatformManifest` (public scope), `PrivateManifest` (private scope, superset). The flow rules generalize beyond cognition to any state a manifest hosts.

**Visibility / info-flow rule:** a manifest can host state involving any repo at or below its visibility scope. Private can host cognition about public repos; public cannot host cognition about private ones. RepoGraph enforces this on writes.

**Contract split:**
- **ContextLifecycle (CL)** — owns schema + I/O + policy enforcement (the 330-line `pre_tool_use.sh` decision tree, rewritten in Python and shipped as a CLI from CL's own `.venv/`)
- **RepoGraph** — owns repo↔manifest authorization; validates every cognition write against the active session anchor's scope
- **Manifest** — session anchor + `.context/` host; data lives here, not scattered across consumer repos
- **Consumer repo (executors, etc.)** — ships a ~1-line shim hook (`exec "$CL_HOME/.venv/bin/cl" hook pre_tool_use "$@"`) and nothing else; no `.context/` data, no CL imports

**Session anchoring (two-layer):**
1. *Picking* is UX — operator launches with an explicit anchor via `cl session start <manifest>` (sets `CL_ANCHOR` env var in the session shell). Hard error if missing — no silent unguarded sessions.
2. *Enforcing* is RepoGraph — every write is validated against the anchor's scope, so mis-selects are caught after the fact. Picking and enforcing are decoupled.

**Reverts implied:**
- TE / DE / CE all have shallow CLP scaffolding (config + hooks, no Python integration) — to be removed once the CL CLI + shim pattern lands. They're library-only, so harness hooks are unnecessary there; the shim will be the only CL surface in those repos.
- Anything CL-shaped reaching into executor *code* (none exists today) stays banned. Executors don't import CL.

**`.console/` vs `.context/`** — `.console/` stays per-repo (operational truth: task/guidelines/backlog/log). `.context/` moves to the anchoring manifest (durable cognition: capsules/checkpoints/handoffs). `.console/` compaction is a future need, not addressed here.

**Stop point — implementation specs still open:**
- Final `CL_ANCHOR` env var name + `cl session start` CLI signature (incl. `--json`, RepoGraph-inferred default behavior)
- Shim's resolution path for `cl` (`CL_HOME` env, PATH symlink, or both)
- RepoGraph's authorization schema — how PM/PrivM declare repo membership in a way RepoGraph can read for visibility checks
- Layout of manifest `.context/` when hosting multiple concurrent work loops (capsule namespacing across repos)

Architectural design considered done. Next session: pick implementation specs and either update ADR 0005 or write a new ADR for the manifest concept.

## 2026-05-21 — Gitignore .console/.context

`.console/.context` is regenerated on every session launch (timestamp changes unconditionally). Added it to `.gitignore` and untracked it with `git rm --cached`. All other repos (OC, RxP, etc.) already gitignore it; CLP was the only outlier. Source truth remains `.console/{task,guidelines,backlog,log}.md`.

## 2026-05-21 — Add closing fence to console-context block

Added <!-- /console-context --> end marker so OperatorConsole only replaces its
managed block and leaves repo-owned content below it untouched.

_Chronological continuity log. Decisions, stop points, what changed and why._
_Not a task tracker — that's backlog.md. Keep entries concise and dated._

## Recent Decisions

_Log significant choices here so they survive context resets._

| Decision | Rationale | Date |
|----------|-----------|------|
| [what was decided] | [why] | [date] |

## Stop Points

_Where did you leave off? What should be verified next session?_

- [what to pick up next]

## Notes

_Free-form scratch. Clear periodically — old entries can be deleted once no longer relevant._

---
