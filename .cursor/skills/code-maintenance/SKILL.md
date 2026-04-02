---
name: code-maintenance
description: >
  Periodic codebase maintenance aligned with AI_GUIDELINES.md (when present at repo root).
  Use when the user asks to "maintain", "clean up", "refactor", "tidy up", or "improve code quality".
  Also trigger on: cleaning docs, adding tests, extracting shared code, splitting large files,
  replacing hand-rolled utilities with libraries, consolidating duplicates, or phrases like
  "too many docs", "file is too big", "use a library", or "DRY up".
---

# Code Maintenance Skill

This skill aligns maintenance work with **`AI_GUIDELINES.md`** at the repo root when that file exists. Run tasks in order **1→6** when doing a full pass (later tasks may depend on earlier cleanup), or let the user pick specific tasks. Each task is independent — skip what is not needed.

## Prerequisites — Read Before You Touch Anything

1. **Read `AI_GUIDELINES.md` in full** — It is the source of truth for documentation layout, dev workflow, code conventions, tooling defaults, and security expectations.
2. **Scan project structure** — `package.json` (or equivalent), configs, `src/` layout — so changes match actual stack (e.g. pnpm, Vitest, Vite when present).

**Hard constraints from `AI_GUIDELINES.md` (do not contradict):**

| Topic | Where to look | Why it matters |
|--------|----------------|----------------|
| Daily task flow | `WIP.md` → `TODO.md` → `PLAN.md` | Maintenance work should not fight the repo’s execution ladder. |
| Decisions & pitfalls | `DEV_NOTE.md` | Prevents re-introducing resolved issues; capture new decisions here (decision basis + conclusion). |
| How to run tests | `TESTING.md` | **Authoritative** for install prerequisites, standard test commands, layout/naming; details live in `docs/06-quality/`. |
| Architecture & contracts | `docs/03-architecture/` | Update when maintenance touches interfaces or module boundaries. |
| Formal / long-form docs | `docs/` (numbered folders) | **Do not arbitrarily rename** `docs/*` numbering; root = high-frequency executable notes, `docs/` = deeper reference. |
| Ops / deploy | `docs/07-operations/` | Routine dev tasks usually need not read it; maintenance that affects deploy must stay consistent with it. |
| Secrets | `.env`, keys, production configs | **Do not read** sensitive files; use scripts the user runs locally if credentials are needed. |

**Quick orientation (`AI_GUIDELINES.md` “我想快速找信息”):**

- Run the project: `README.md` → `docs/00-overview/GETTING-STARTED.md` (if present)
- What to do next: `WIP.md` → `TODO.md` → `PLAN.md`
- Tests: `TESTING.md` → `docs/06-quality/`
- Decisions & gotchas: `DEV_NOTE.md`

Skipping this step leads to doc moves that break team alignment, tests run the wrong way, or changes that violate documented boundaries.

---

## Task 1: Clean Up Documentation

**Goal:** Keep only **maintainable, necessary** documentation — executable conclusions and constraints, not churn. Match root vs `docs/` split from `AI_GUIDELINES.md`.

**Steps:**

1. Inventory markdown (exclude `node_modules`, `dist`, build artifacts). Map each file to either **root** (high-frequency, `@`-friendly) or **`docs/`** (formal / long-form).
2. For each doc, assess:
   - Duplicate or weaker twin of another doc? → merge or delete the weaker one; ensure a single **authoritative** copy (if a root copy exists only as convenience, state which path is canonical).
   - Temporary / superseded? → delete or archive per guidelines (“阶段性完成后，及时清理或归档临时文档”).
   - Stale (wrong paths, removed APIs)? → update or remove.
   - Too thin to own a file? → fold into parent (`PLAN`/`TODO`/`DEV_NOTE`/`docs/` section as appropriate).
3. Consolidate scattered notes into the **right home**: e.g. deployment fragments → `docs/07-operations/` (summaries in root only if truly daily-use).
4. After cleanup, each remaining doc’s **first paragraph** should state purpose; prefer “结论、约束、决策依据” over narrative filler (`AI_GUIDELINES.md`).

**What NOT to do:**

- Do **not** rename or renumber `docs/` folders on a whim — team alignment depends on stable paths.
- Do **not** delete migration artifacts (SQL, Drizzle, Prisma, etc.).
- Do **not** remove multi-tool AI instruction files if present — they serve different products and must stay separate when the repo includes them (e.g. `AGENTS.md`, `GEMINI.md`, `.github/copilot-instructions.md`, `.claude/` config).

---

## Task 2: Crystallize Knowledge into Long-term Docs

**Goal:** Persist recent learnings so they outlive chat — same spirit as “需要时沉淀决策” and onboarding table in `AI_GUIDELINES.md`.

**Steps:**

1. Review recent history (`git log --oneline -30`) for patterns, infra changes, or contracts not reflected in docs.
2. **`DEV_NOTE.md`**: Ensure it still reflects **key decisions** (basis + conclusion), not a diary. Add pitfalls, framework quirks, irreversible choices discovered lately.
3. **`docs/03-architecture/`**: If maintenance touches module boundaries, data model, or APIs, update the relevant section or OpenAPI-linked doc.
4. **`PLAN.md`**: If new milestones or scope shifts emerged, align so `PLAN`/`TODO`/`WIP` stay consistent (“若 … 在 `PLAN.md` 做一致性同步” where applicable).
5. **`docs/06-quality/`**: If test strategy or reporting changed, update there; keep **`TESTING.md`** as the one-page command + pointer — not a duplicate strategy bible.
6. Prefer **extending existing docs** over new files to honor “只保留必要文档”. **Principle:** a teammate joining in three months understands **why** things are the way they are.

---

## Task 3: Expand Test Coverage

**Goal:** Improve coverage **using the project’s real toolchain**, with verification commands from **`TESTING.md`**.

**First:** Detect test runner (e.g. Vitest, Jest, pytest) and layout from `TESTING.md` and `docs/06-quality/`; **follow those conventions exactly**.

**Priority (when choosing what to add):**

1. **Pure logic / utilities** — High priority; algorithms, helpers, data transforms.
2. **API routes / endpoints** — Validation, error shapes, edge cases; mock externals at boundaries.
3. **UI components** — Rendering, interaction, state; use the project’s DOM test setup.
4. **E2E / smoke** — Light coverage for critical paths and config loading.

**How to write tests (`AI_GUIDELINES.md`):**

- **Sync tests with code changes** — same minimal-delivery loop: implement → add/adjust tests for behavior and regressions → run until green.
- Names describe **scenario**, not implementation detail.
- One focused behavior per test where practical.

**Bug-driven maintenance:** stabilize reproduction → capture in test or scripted steps → fix → keep regression test.

**What NOT to do:**

- Do not test framework internals.
- Do not duplicate checks the type system already enforces (when TypeScript is in use).
- Do not over-mock so tests only prove mock wiring.

**Verification:** After changes, run whatever **`TESTING.md`** lists; align with CI if documented there.

---

## Task 4: Refactor Large Files

**Goal:** Keep files navigable — **`AI_GUIDELINES.md` thresholds**: aim near **~300 lines** per file; **above ~400 lines**, prioritize splitting when clarity/ownership suffers.

**Steps:**

1. Find large sources (adapt to OS; examples):
   - Unix-like: `find` + `wc` on `*.{ts,tsx,js,jsx,py}` under `src` (or project convention).
   - Or use editor / ripgrep tooling to list longest files.
2. For each candidate, find **natural seams**: types file, domain service, route group, single component per file, config/constants separate from logic.
3. After splitting: fix imports, run **typecheck** and **tests** per project (`AI_GUIDELINES.md` pre-submit: format if present, typecheck, build).

**Splitting hints:** route groups, domain services, one component per file, kebab-case paths (`AI_GUIDELINES.md` naming).

**What NOT to do:**

- Do not split cohesive modules solely for line count.
- Do not create many tiny files that only import each other — worse than one clear ~300-line module.

---

## Task 5: Extract Shared Code (DRY)

**Goal:** If the **same logic appears three or more times**, centralize in a shared module with a clear home.

**Steps:**

1. Search for duplicated fetch wrappers, validation, error handling, types, guards, formatters.
2. Place shared code by scope:
   - Cross-cutting → `common`/shared directory (match existing layout).
   - Module-specific → inside that module.
3. Replace call sites; **add tests** for extracted units.
4. Run full test suite and typecheck.

**What NOT to do:**

- Do not extract at two occurrences only — threshold remains **3+**.
- Do not build a single “god utils” file; group by domain.
- Do not merge code that looks alike but **must evolve independently**.

---

## Task 6: Replace Hand-rolled Code with Libraries

**Goal:** Prefer maintained libraries over custom implementations **when dependency cost is justified** — consistent with “少靠大而全的预防工程” (no dependency bloat).

**Project defaults from `AI_GUIDELINES.md`:**

- **Icons:** Prefer a **single** icon library (example given: **`lucide`**); avoid inline SVG; use unambiguous names (e.g. `SaveIcon` vs `Save`).
- **Dates, charts, deep utils:** If the project already standardized (e.g. dayjs, chart lib, lodash-es), **follow existing choice** before introducing a second stack.

**Steps:**

1. Find hand-rolled patterns: inline SVG icons, ad-hoc debounce/clone/merge, fragile date math, duplicate state plumbing where a project-approved library already exists.
2. For each replacement: add dependency only if justified → swap implementation → remove dead code → test and typecheck.
3. For embeddable or size-sensitive bundles: prefer tree-shakeable imports or lighter alternatives; **weigh bundle size** (`AI_GUIDELINES.md` spirit: avoid unnecessary weight).

**What NOT to do:**

- Do not add a heavy library for one trivial helper.
- Do not replace stable, well-tested bespoke code unless the library meaningfully reduces risk or complexity.
- Do not ignore bundle constraints on thin clients.

**Code style reminder when touching refactors:** prefer `function foo() {}` over `const foo = () => {}` when there is no project override (`AI_GUIDELINES.md`).

---

## Running the Maintenance

When this skill triggers:

1. Ask whether to run **all tasks (1→6)** or a **subset**.
2. For each selected task, give a **short summary of findings** and **proposed changes** before applying edits (per `AI_GUIDELINES.md` “直说问题” / trade-offs when relevant).
3. After substantive changes, run checks **as the project defines** — typically match the “提交前常规检查” pattern: format (if any), **typecheck**, **build**, and **tests** from **`TESTING.md`**.
4. Close the loop like the minimal unit workflow: update **`WIP.md` / `TODO.md`** if maintenance was task-driven; record **new key decisions** in **`DEV_NOTE.md`** (and `docs/03-architecture/` if contracts changed).
5. End with a concise summary of **what changed** and **what was verified**.

---

## Reference

- **Canonical norms for repos using this layout:** `AI_GUIDELINES.md` at the repository root.
