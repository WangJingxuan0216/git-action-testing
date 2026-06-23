# Concurrency-Group Testing Plan

**Project:** Allianz CI/CD (Databricks / DABs) — GitHub Actions concurrency validation
**Source design:** [Allianz – CI/CD Design](https://docs.google.com/document/d/1igDl9GoXoGI5nI8W7ZUFwes9fF0YpS_WmTzC1FzKBf4/edit) (Deployment Matrix)
**Date:** 2026-06-23
**Repo:** `git-action-testing`

---

## 1. Objective

Validate **how concurrency groups should be configured per environment** so that:

1. **Dev & Test** — concurrency is scoped finely enough that **squads never block each other**, and within a squad **each developer's feature pipeline runs independently** (no queuing, no interruption from teammates).
2. **UAT, Pre-Prod, Prod** — deployments are **strictly one workflow at a time** (serialized; an in-flight deploy is never cancelled).
3. **UAT (secondary, client request)** — evaluate whether UAT *can* be made **concurrent**, and what breaks if it is.

This is a **test plan only** — it specifies the concurrency configuration to put under test, the scenarios to run, and the pass/fail criteria. It does not itself change production behaviour.

---

## 2. Background: how concurrency works in GitHub Actions

Only two levers exist, both under the `concurrency:` key:

| Lever | Effect |
|-------|--------|
| `group: <string>` | Runs sharing the **same evaluated string** form one serialization lane. Different strings ⇒ run in parallel. **The string is matched across workflow files** — two different `.yml` files with the same `group` share a lane. |
| `cancel-in-progress: true` | A new run in the lane **cancels the in-progress run** (newest-wins). |
| `cancel-in-progress: false` | A new run **waits** for the in-progress run to finish (queue). |

**Granularity is entirely a function of the group key:**

- Per-developer isolation → key contains `github.head_ref` (the source/feature branch).
- Per-squad isolation → key contains the squad identifier.
- "One at a time" → a **static** key (no variable), so every run collides into a single lane.

### ⚠️ Critical caveat to test explicitly
With `cancel-in-progress: false`, GitHub keeps **at most ONE pending run per group**. If a third run arrives while one is running and one is already pending, the **older pending run is cancelled** (the running one is untouched). So "queue" really means *running + 1 waiting*. For UAT/Prod this means a burst of 3 releases can silently drop the middle one — this must be verified, not assumed.

### Notes on the trigger model
- `github.head_ref` is populated **only for `pull_request` events** (empty for `push`/`release`). All Dev/Test/UAT/Pre-Prod workflows here are `pull_request`-triggered, so `head_ref` is available. `5-deploy-prod.yml` is `release`-triggered, so it must use a static key.
- These workflows include a `Hold (observability — remove after test): sleep 45` step and a `Show concurrency key` echo step specifically so overlapping runs are observable in the Actions UI. **Keep these during testing; remove before merge.**

---

## 3. Environment → workflow → concurrency mapping (under test)

| Stage / trigger | Env | Workflow file | **Current** `group` / `cancel` | **Target** `group` / `cancel` | Intent |
|---|---|---|---|---|---|
| PR `feature/*` → `integrate/*` | **Dev** | `1-deploy-dev.yml` | `dev-${{ github.head_ref }}` / `true` | `dev-${{ github.head_ref }}` / `true` | Per-developer (per feature branch); newest push wins |
| PR `integrate/<squad>` → `develop` | **Test** | `2-deploy-test.yml` | `test-${{ github.head_ref }}` / `true` | `test-${{ github.head_ref }}` / `true` | Per-squad (per integration branch); squads isolated |
| `workflow_dispatch` (cut release) | Test | `3-cut-release.yml` | `cut-release` / `false` | `cut-release` / `false` | One cut at a time |
| PR `release/*` → `main` (UAT + BU sign-off) | **UAT** | `4-deploy-uat.yml` | _none_ | `uat` / `false` | **One at a time (static lane)** |
| PR `hotfix/*` → `release/*` (Pre-Prod test) | **Pre-Prod** | `6-hotfix-preprod.yml` | _none_ | `preprod` / `false` | **One at a time (static lane)** |
| `release: published` → Pre-Prod test, then Prod | **Pre-Prod**, then **Prod** | `5-deploy-prod.yml` | `deploy-prod` / `false` (whole workflow) | `preprod-validate` job → `preprod` / `false`; `deploy-prod` job → `prod` / `false` (see §6) | **One at a time per env** |

**Pre-Prod footprint (validated 2026-06-23):** the Pre-Prod env is exercised in exactly **two** places — the `preprod-validate` job of `5-deploy-prod.yml` (test before prod, followed by the `production` approval gate) and `6-hotfix-preprod.yml` (test for `hotfix/*` PRs into `release/*`). The `preprod` lane must serialize **both**. Note: `4-deploy-uat.yml`'s stage-2 job is currently mis-named `preprod-validate` / `environment: preprod`, but per the latest design it is the **UAT / BU sign-off gate (Branch Flow step 6.a)** — it should be `environment: uat` and belongs to the `uat` lane, not `preprod`. See §6 Q1.

**Configuration changes introduced for testing** (target column): (a) add a static `uat` group to `4-deploy-uat.yml`; (b) add a static `preprod` group to `6-hotfix-preprod.yml`; (c) split `5-deploy-prod.yml`'s single workflow-level `deploy-prod` group into **job-level** lanes — `preprod` on `preprod-validate`, `prod` on `deploy-prod` — so Pre-Prod and Prod serialize independently and the Pre-Prod test shares the `preprod` lane with `6-hotfix-preprod.yml`.

> **Design note (squad granularity):** for feature branches, `github.head_ref` (e.g. `feature/home-quote-fix`) is *finer* than squad — so squads are isolated automatically and developers within a squad never collide, **provided each developer uses a distinct branch**. An explicit squad key is therefore only needed if you want squad-level *serialization*, which the requirement explicitly does **not** want. This assumption (one branch per developer) is itself a test condition — see A1/A2.

---

## 4. Prerequisites & setup

- Branch protection / required checks **disabled or relaxed** on the test branches so PRs can be opened freely and runs aren't blocked.
- The following branches exist (already present in repo): `develop`, `integrate/squad-1`, `integrate/team-a`, `integrate/team-b`, `feature/team-a-1`, `feature/team-b-1`, `release/v0.1`, `release/v1`, `hotfix/test1`, `hotfix/ct`, `main`.
- Each workflow retains its `sleep 45` hold and `Show concurrency key` echo for the duration of testing.
- Observation surface: **Actions tab** → each run shows status `Queued` / `In progress` / `Cancelled` / `Success`, and the `Show concurrency key` log line confirms which lane it landed in.
- To exercise per-developer scenarios without real teammates, simulate "two developers in the same squad" with **two distinct feature branches** (e.g. `feature/team-a-1` and a new `feature/team-a-2`), both PR'd into the same `integrate/team-a`.

---

## 5. Test scenarios

### Group A — Dev & Test: squad + per-developer isolation

| ID | Scenario | Setup | Expected result | Pass criterion |
|----|----------|-------|-----------------|----------------|
| **A1** | Two developers, **same squad**, different feature branches run at the same time | Open PRs `feature/team-a-1 → integrate/team-a` **and** `feature/team-a-2 → integrate/team-a` within the 45s hold window | Two runs of `1-deploy-dev`, groups `dev-feature/team-a-1` and `dev-feature/team-a-2` | **Both run concurrently**; neither is `Queued` or `Cancelled` |
| **A2** | Same developer pushes twice rapidly to one feature branch | Push to `feature/team-a-1`, then push again within 45s | Second run enters lane `dev-feature/team-a-1` | First run is **`Cancelled`**, second runs (newest-wins, `cancel-in-progress: true`) |
| **A3** | Two **different squads** active simultaneously | PRs `integrate/team-a → develop` and `integrate/team-b → develop` within hold window | Runs of `2-deploy-test`, groups `test-integrate/team-a` and `test-integrate/team-b` | **Both run concurrently**; squads do not block each other |
| **A4** | Stage isolation: feature, integration, and develop activity for the **same squad** don't collide | Trigger A1 + A3 for `team-a` together | Groups are `dev-feature/...`, `test-integrate/team-a` (distinct prefixes) | **No cross-stage queuing**; all run in parallel |
| **A5** | Same integration branch, rapid re-push | Push twice to `integrate/team-a` within 45s | Lane `test-integrate/team-a` | First **`Cancelled`**, second runs (newest-wins) |

**Group A pass = squads isolated (A3), developers isolated (A1), stages isolated (A4), and stale runs auto-cancelled for fast feedback (A2, A5).**

---

### Group B — UAT / Pre-Prod / Prod: one workflow at a time

| ID | Scenario | Setup | Expected result | Pass criterion |
|----|----------|-------|-----------------|----------------|
| **B1** | Two release PRs into `main` at once (UAT) | After applying target config, open PRs `release/v0.1 → main` and `release/v1 → main` within hold window | Both land in static lane `uat` | First **`In progress`**, second **`Queued`** until first finishes, then runs |
| **B2** | Two hotfix PRs into release branches at once (Pre-Prod) | Open `hotfix/test1 → release/v0.1` and `hotfix/ct → release/v1` within hold window | Both land in static lane `preprod` | First runs, second **`Queued`** (serialized) |
| **B3** | Two prod deploys (release published) | Publish two GitHub Releases in quick succession | Both land in lane `deploy-prod` | First runs to completion, second **`Queued`**, **never cancelled** (`cancel-in-progress: false`) |
| **B4** | In-flight deploy is **never** killed | During B1/B3, while run #1 is mid-`sleep`, trigger run #2 | — | Run #1 reaches `Success` (not `Cancelled`); run #2 only starts afterward |
| **B5** | ⚠️ **Burst-of-3 queue-drop** | Trigger **three** release PRs into `main` within the hold window | Lane `uat`: 1 running + 1 pending max | **Middle** run is `Cancelled` (pending-slot displaced); confirm this is acceptable, or document mitigation |
| **B6** | Pre-Prod gate is **not** double-counted | Drive one release through `4-deploy-uat` (UAT sign-off) then a `release: published` through `5-deploy-prod` (Pre-Prod test + prod approval) | `4-deploy-uat` stage-2 = **UAT** approval; `5-deploy-prod` = Pre-Prod test → **production** approval | Exactly **one** Pre-Prod test (in `5-deploy-prod`) and no duplicate `preprod` approval at the release→main PR |

**Group B pass = exactly one run executes per environment lane at any instant (B1–B3), in-flight deploys are protected (B4), and the queue-depth-1 behaviour is observed and accepted/mitigated (B5).**

---

### Group C — UAT concurrent (secondary, client request)

Goal: determine whether multiple releases can sit in UAT **at the same time** and what it costs.

| ID | Scenario | Setup | Expected result | Pass criterion |
|----|----------|-------|-----------------|----------------|
| **C1** | Switch UAT lane from static → per-branch | Change `4-deploy-uat.yml` group to `uat-${{ github.head_ref }}`, `cancel-in-progress: false` | Lanes `uat-release/v0.1` and `uat-release/v1` are distinct | Two release PRs run UAT **concurrently** |
| **C2** | Resource-contention probe | With C1 active, inspect what each concurrent UAT run would deploy (DAB target `uat`, workspace, resource names) | Both target the **same** UAT workspace + canonical (un-prefixed) resource names per design | **Identify collisions** — same job/pipeline names overwrite each other. Document the blocker. |
| **C3** | Isolation options write-up | Based on C2 | — | Produce a recommendation: (a) keep UAT serial, or (b) make concurrent **only if** per-release isolation exists (separate UAT target/workspace or release-suffixed resource names per the design's `mode: production` canonical-naming rule) |

**Group C pass = a clear, evidenced recommendation** on whether concurrent UAT is feasible, and the precise conditions (workspace/target/naming isolation) required to enable it without resource clashes.

> **Design tension to call out:** the Deployment Matrix specifies UAT runs in `mode: production` with **canonical (no-prefix) resource names**. Concurrent UAT runs deploying to one workspace with identical names will overwrite each other. So C is fundamentally a question of *isolation*, not just *concurrency keys* — the group key alone cannot make UAT safely concurrent.

---

## 6. Open design questions to resolve via testing

1. **Pre-Prod lane scope (validated).** Pre-Prod genuinely runs in **two** workflows: the `preprod-validate` job of `5-deploy-prod.yml` (test before prod) and `6-hotfix-preprod.yml` (hotfix PR into `release/*`). Both must **share the `preprod` group string** to serialize across workflow files. Recommended target:
   - **Per-environment static lanes** — `uat`, `preprod`, `prod` as three independent groups (matches "one at a time per env").
   - **Split `5-deploy-prod.yml` at the job level** — its current single workflow-level `deploy-prod` group couples Pre-Prod and Prod. Move concurrency to job level: `preprod` on `preprod-validate`, `prod` on `deploy-prod`. This (i) lets Pre-Prod share a lane with `6-hotfix-preprod.yml` and (ii) keeps the Prod lane independent so a queued prod deploy never blocks an unrelated hotfix's pre-prod test.
   - **Fix the mis-labeled gate** — `4-deploy-uat.yml`'s stage-2 `preprod-validate` (`environment: preprod`) is actually the UAT/BU sign-off (step 6.a). Rename to `environment: uat` so there are not two `preprod` approval gates and the release→main PR isn't double-approved. Add test **B6** below.
2. **One-branch-per-developer assumption** (§3 design note) — confirm in A1/A2 that developer isolation depends on distinct branch names; document the convention.
3. **Squad extraction** — if any feature branch naming does *not* encode the squad and squad-level grouping is later desired, decide how to derive the squad (branch prefix convention vs. mapping).

---

## 7. Pass/fail summary

| Requirement | Validated by | Pass condition |
|-------------|--------------|----------------|
| Dev/Test isolated per squad | A3, A4 | Different squads run in parallel |
| Dev isolated per developer | A1 | Same-squad developers run in parallel |
| Fast feedback (no stale runs) | A2, A5 | Superseded runs auto-cancel |
| UAT one at a time | B1, B4 | Second release queues; first never cancelled |
| Pre-Prod one at a time | B2, B4 | Second hotfix queues |
| Prod one at a time | B3, B4 | Second deploy queues; never cancelled |
| Queue-depth-1 understood | B5 | Burst-of-3 behaviour observed & accepted/mitigated |
| UAT concurrency feasibility | C1, C2, C3 | Evidenced go/no-go recommendation |

---

## 8. Cleanup (post-test)

- Remove the `Hold (observability — remove after test): sleep 45` step from all workflows.
- Revert any temporary Group C config (`uat-${{ github.head_ref }}`) unless the concurrent-UAT decision is to keep it.
- Re-enable branch protection / required status checks.
- Fold the validated `group` / `cancel-in-progress` settings into `alz-dbx-domain-template` so all project repos inherit them.
