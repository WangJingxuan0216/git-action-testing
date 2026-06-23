# Concurrency-Group Testing Plan

**Project:** Allianz CI/CD (Databricks / DABs) — GitHub Actions concurrency validation
**Source design:** [Allianz – CI/CD Design](https://docs.google.com/document/d/1igDl9GoXoGI5nI8W7ZUFwes9fF0YpS_WmTzC1FzKBf4/edit) (Deployment Matrix)
**Date:** 2026-06-23
**Repo:** `git-action-testing`
**Status:** Executed 2026-06-23 — Groups A, B, C all run. **All scenarios PASS.**

---

## 0. Results Summary & Recommendations

All three requirement groups were validated empirically against live GitHub Actions runs (status transitions captured by polling `gh run list` during a `sleep 45` hold). Detailed per-scenario evidence is in §5.

| Req | Outcome | Final config |
|-----|---------|--------------|
| **Dev & Test** — squad + per-developer isolation | ✅ PASS (A1–A5) | **No change** — `1-deploy-dev.yml` / `2-deploy-test.yml` already key on `…-${{ github.head_ref }}`, `cancel-in-progress: true` |
| **UAT / Pre-Prod / Prod** — one at a time | ✅ PASS (B1–B6) | `uat` (workflow-level), `preprod` (shared by `5-deploy-prod` preprod job + `6-hotfix-preprod`), `prod` (`deploy-prod` job); all `cancel-in-progress: false` |
| **UAT concurrent** (secondary) | ✅ Achievable, ⚠️ not recommended (C1–C3) | Per-branch key `uat-${{ github.head_ref }}` works, but unsafe without per-release workspace/name isolation → **keep UAT serial** |

**Three takeaways:**

1. **Granularity is just the group key.** Per-developer/per-squad isolation comes free from `github.head_ref` (squads never collide as long as each dev uses a distinct branch); "one at a time" comes from a *static* key. The Dev/Test workflows already had this right — no change needed.
2. **`cancel-in-progress: false` queues only one run deep.** Proven in B5: a burst of 3 deploys keeps 1 running + 1 pending and **silently cancels the older pending one** — the middle release is dropped. *Action:* accept (deploys are infrequent + serialized by the gated cut) or add alerting on cancelled deploy runs.
3. **Concurrent UAT is a one-line key change but not safe as-is.** C1 ran two releases through UAT in parallel; C2 showed they'd overwrite each other in the single UAT workspace (canonical resource names, shared bundle lock). A concurrency key controls *timing*, not *isolation*. Recommend serial UAT unless per-release UAT environments/namespaces are provisioned.

**One fix applied to the workflows** (committed on `feature/team-a-1`, `6b0d750`): added the `uat`/`preprod`/`prod` lanes and corrected `4-deploy-uat.yml`'s mis-labelled stage-2 gate from `environment: preprod` → `uat` (it is the UAT/BU sign-off, not a second Pre-Prod approval). See §3.

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

#### Group A — Execution Results (2026-06-23, repo `WangJingxuan0216/git-action-testing`)

**Outcome: ALL PASS ✅.** Runs triggered on workflows `1-deploy-dev.yml` and `2-deploy-test.yml` (unchanged from baseline — both already key on `…-${{ github.head_ref }}` with `cancel-in-progress: true`). Status observed via `gh run list` polling during the `sleep 45` hold.

| ID | Result | Evidence (run state over time) |
|----|--------|--------------------------------|
| **A1** | ✅ PASS | `feature/team-a-2` + `feature/team-a-3` both `in_progress` 14:53:27 → 14:54:12 (~45s overlap), then both `completed/success`. Distinct lanes `dev-feature/team-a-2`, `dev-feature/team-a-3`. No queuing, no cancellation. |
| **A2** | ✅ PASS | `feature/team-a-2`: run from push @04:55:44 went `in_progress`; second push @04:56:10 entered `pending`, then GitHub **cancelled** the first and promoted the second to `success`. Lane `dev-feature/team-a-2`. |
| **A3** | ✅ PASS | `integrate/team-a` + `integrate/team-b` both `in_progress` 14:58:41 → 14:59:24, both `completed/success`. Lanes `test-integrate/team-a`, `test-integrate/team-b`. |
| **A4** | ✅ PASS | Same squad, both stages: `feature/team-a-2` (dev lane) + `integrate/team-a` (test lane) both `in_progress` 15:01:03 → 15:01:32; both `success`. `dev-` and `test-` prefixes keep stages independent. |
| **A5** | ✅ PASS | `integrate/team-a`: run from push @05:03:09 `in_progress`; second push @05:03:25 went `pending` → `in_progress` while the first flipped to `cancelled`; second finished `success`. Lane `test-integrate/team-a`. |

**Conclusion:** the baseline Dev/Test keys (`dev-${{ github.head_ref }}`, `test-${{ github.head_ref }}`, `cancel-in-progress: true`) fully satisfy requirement #1 — squads never block each other, developers within a squad run independently, dev/test stages do not collide, and superseded runs auto-cancel. **No change needed to `1-deploy-dev.yml` / `2-deploy-test.yml`.** The one-branch-per-developer assumption (§3 design note) held.

**Observed nuance (matches GitHub docs):** with `cancel-in-progress: true`, a superseding run first appears as `pending` for a few seconds before GitHub cancels the in-flight run and promotes it — the cancellation is not instantaneous.

**Test artifacts to clean up:** branches `feature/team-a-2`, `feature/team-a-3`; PRs #13, #14; empty trigger-commits on `integrate/team-a` and `integrate/team-b` (no file changes).

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

#### Group B — Execution Results (2026-06-23, repo `WangJingxuan0216/git-action-testing`)

**Outcome: ALL PASS ✅.** Tested the **target** config: `uat` (workflow-level on `4-deploy-uat.yml`), `preprod` (shared by `6-hotfix-preprod.yml` and the `preprod-validate` job of `5-deploy-prod.yml`), and `prod` (the `deploy-prod` job). All `cancel-in-progress: false`. Run on throwaway `release/*`, `hotfix/*` branches + two GitHub Releases, with a temporary `sleep 45` hold added to each lane-holding job for observability. `main` temporarily carried the instrumented workflows (so `release: published` could fire `5-deploy-prod`).

| ID | Result | Evidence (run/job state over time) |
|----|--------|-------------------------------------|
| **B1** | ✅ PASS | `uat` lane: `release/b1-1` `in_progress` 15:29:27→15:30:09 while `release/b1-2` held `pending`; b1-1 `success`, *then* b1-2 ran → `success`. Never overlapped, b1-1 never cancelled. |
| **B2** | ✅ PASS | `preprod` lane: `hotfix/b2-1` `in_progress` 15:33:23→15:33:56 while `hotfix/b2-2` `pending`; b2-1 `success`, then b2-2 ran → `success`. (Ran concurrently with B5 on the `uat` lane — lanes proven independent.) |
| **B3** | ✅ PASS | `prod` lane: release-1 `Deploy to Prod` = `waiting` at the `production` gate (from 15:37:08) holding the lane; release-2 `Deploy to Prod` = `pending` behind it for ~4 min. Serialized; one deploy at a time. (`preprod-validate` jobs serialized first on the `preprod` lane.) |
| **B4** | ✅ PASS | Across the entire B3 observation neither prod run was `cancelled` — the `waiting` deploy was protected; the second `pending`-queued, not killed. (`cancel-in-progress: false`.) |
| **B5** | ✅ PASS | `uat` lane burst of 3: `release/b5-1` ran; `release/b5-2` `queued` then **`cancelled`** the instant `release/b5-3` took the single pending slot; final = b5-1 `success`, **b5-2 `cancelled`**, b5-3 `success`. Empirically confirms the queue-depth-1 caveat (§2). |
| **B6** | ✅ PASS (structural + observed) | Release→main runs hit `environment: uat` (`uat-signoff` job); only the prod runs hit `environment: production`. Two distinct gates — no duplicate Pre-Prod approval. |

**Conclusion:** the target config delivers requirement #2 — UAT, Pre-Prod, and Prod each run strictly one workflow at a time, in-flight deploys are never cancelled, and the `preprod`/`prod` split keeps the two lanes independent (a queued prod deploy does not block an unrelated pre-prod test). The `preprod` group is correctly shared across `6-hotfix-preprod.yml` and `5-deploy-prod.yml`.

**Caveat surfaced (B5) — action required:** with `cancel-in-progress: false`, a lane holds only **1 running + 1 pending**; a 3rd concurrent trigger silently **drops the older pending run**. For UAT/Prod this means a burst of releases can lose the middle one. Mitigations to decide: (a) accept (releases are infrequent and serialized upstream by the gated cut); (b) rely on branch protection / the gated `cut-release` to prevent bursts; or (c) add alerting on cancelled deploy runs so a dropped one is noticed.

**Test-instrumentation note:** an initial B1 run failed because an unquoted YAML scalar `echo "... #${{ … }}"` let YAML treat ` #` as a comment (truncating the line → shell EOF error). Fixed by removing the `#`; this was a defect in the throwaway *test* copy only — the committed workflows use block scalars and are unaffected.

**Test artifacts to clean up:** branches `release/b1-1`, `release/b1-2`, `release/b5-1..3`, `release/b2-base`, `hotfix/b2-1`, `hotfix/b2-2`; PRs #17–#23; Releases/tags `vB3-1`, `vB3-2`; the instrumented workflows on `main` (revert `main` to its pre-test state, commit `5e5e865`).

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

#### Group C — Execution Results (2026-06-23, repo `WangJingxuan0216/git-action-testing`)

**Outcome: concurrency is achievable at the CI layer (C1 ✅), but NOT safe to enable under the current UAT deployment model (C2). Recommendation: keep UAT serial by default (C3).**

| ID | Result | Evidence / finding |
|----|--------|--------------------|
| **C1** | ✅ Concurrent confirmed | Changed the lane key to `uat-${{ github.head_ref }}`. Two `release/* → main` PRs (`release/c1-1`, `release/c1-2`) ran **`in_progress` simultaneously** 16:27:44 → 16:28:18 (~35s overlap), both `success`. Inverse of B1 — the per-branch key gives each release its own lane. |
| **C2** | ⚠️ Contention identified | The CI key removes the *serialization*, but both runs still `databricks bundle deploy --target uat` into the **same UAT workspace** with **canonical (un-prefixed) names** (per the Deployment Matrix). Concrete collisions: (1) identical job/pipeline/schema names → the later deploy **overwrites** the earlier; (2) shared DABs bundle root `/Workspace/Deployed/.bundle/<bundle>/uat` → deployment-state/lock contention, one deploy can fail or corrupt the other; (3) shared `run_as` runner SP and UC objects; (4) business ambiguity — a BU reviewer can no longer tell *which* version they are signing off. |
| **C3** | ✅ Recommendation | **Keep UAT serial (static `uat` lane, as validated in B1)** — it matches the design's explicit "Sequential UAT constraint." Concurrent UAT is only safe if the client adds **per-release isolation**, e.g. (a) a per-release UAT target/workspace (`--target uat-<release>` → separate workspace, or a release-namespaced catalog/schema), or (b) release-suffixed resource names + separate bundle root per release — but (b) breaks the "production-mode canonical names" property UAT is meant to validate. The long-UAT-cycle pain is better addressed by the existing `release/v*` branching (parallel feature work continues on `develop` while one release sits in UAT) than by parallelising UAT itself. |

**Bottom line for the client:** *"Can UAT be made concurrent?"* — Yes at the pipeline level (one-line key change, proven in C1), but **not recommended** without provisioning isolated UAT environments/namespaces per release. The concurrency key is necessary but not sufficient; the real blocker is workspace + resource-name isolation. Default recommendation stands: **serial UAT**.

**Test artifacts to clean up (add to the Group B list):** branches `release/c1-1`, `release/c1-2`; PRs #24, #25; and `main` now carries the per-branch `uat-${{ github.head_ref }}` variant on top of the Group B instrumentation.

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


## 9. Testing screenshots
