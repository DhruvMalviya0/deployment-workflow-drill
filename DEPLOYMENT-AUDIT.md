# Deployment Audit — Checkout Service

This audit reviews the current release process for the checkout service based on
`docs/current-release-process.md`, `docs/deployment-history.md`, `docs/incident-log.md`,
`examples/release-request-example.md`, and `.github/workflows/deploy.yml`. It identifies
concrete process weaknesses, the operational risk each one creates, the release impact
already observed, and the control needed to close the gap.

---

## Weakness 1: No approval stage before production deployment

**Evidence:** `.github/workflows/deploy.yml` triggers on every `push` to `main` and
proceeds straight to `Deploy checkout service` with no manual approval job, required
reviewer, or environment protection rule. `docs/current-release-process.md` confirms
this directly: "No human approval step is required." The release request example
(`examples/release-request-example.md`) lists the deployment commands as `git checkout
main` → `git push origin main` → "Release executes automatically," with the note "No
approval workflow is defined."

**Operational risk:** Any engineer with write access to `main` can unilaterally trigger
a production release. There is no independent check that the change is intended,
reviewed, or scheduled before it ships.

**Release impact:** Incident 1 in `docs/incident-log.md` ("Bad deployment reached
production," 2026-05-18) occurred because "a feature branch was deployed directly to
production without proper validation," causing 30 minutes of incorrect checkout totals.

**Recommended control:** Require a named Release Approver (or on-call lead) to approve
a GitHub Environment protection rule (or equivalent manual gate) before the deploy job
is allowed to run against the `production` environment. No workflow run reaches the
deploy step without a recorded approval.

---

## Weakness 2: No pre-deployment validation or build gate

**Evidence:** The entire `deploy.yml` workflow consists of a checkout step and a single
`Deploy checkout service` step. Its own echo output admits this: `"No validation gate or
approval step is configured."` There is no test job, no build-artifact verification, no
lint/security scan step, and no staging deployment prior to production.

**Operational risk:** Broken, untested, or unreviewed code can reach production the
moment it lands on `main`, with no automated backstop to catch regressions before
customers are affected.

**Release impact:** Incident 3 in `docs/incident-log.md` ("Validation skipped,"
2026-04-29) shipped a broken checkout flow because "the deployment process did not
require validation gates or build verification," causing failed orders and manual
remediation. Deployment history also shows v1.4.2 failed because "inventory validation
was skipped."

**Recommended control:** Add mandatory CI stages (unit tests, build verification,
security/dependency scan) as required status checks on `main`, and insert a staging
deployment + smoke-test step before the workflow is permitted to promote to production.

---

## Weakness 3: No rollback plan or rollback readiness check

**Evidence:** `docs/current-release-process.md` states plainly: "No rollback plan is
prepared before deployment." The release request template
(`examples/release-request-example.md`) has no rollback section at all. `deploy.yml`
has no previous-version reference, tagging, or rollback job.

**Operational risk:** When a release fails, there is no pre-defined rollback artifact
(previous image tag, manifest version, or documented procedure) for engineers to
execute against. Recovery becomes ad hoc investigation under incident pressure.

**Release impact:** Incident 2 in `docs/incident-log.md` ("Rollback delayed,"
2026-05-12) reports the team "spent 45 minutes determining the rollback procedure,"
directly extending customer downtime. `docs/deployment-history.md` corroborates this
for v1.4.2: "Rollback took 45 minutes."

**Recommended control:** Require every release to record a rollback target (last known
good version/tag) as part of the release request, and maintain a documented,
pre-tested rollback procedure (see `ROLLBACK-PLAYBOOK.md`) that can be executed without
discovery work during an incident.

---

## Weakness 4: No post-deployment health verification

**Evidence:** `docs/current-release-process.md` states: "No health checks are recorded
after deployment." The `deploy.yml` workflow ends at the `echo "Deployment complete"`
line — there is no smoke test, endpoint check, error-rate check, or monitoring
assertion after the deploy step runs. Deployment history shows this is systemic: v1.4.1
and v1.3.7 were marked "Success" with "no post-release verification captured" / "no
health verification was documented."

**Operational risk:** A deployment can be labeled "successful" purely because the
deploy command exited without error, even if the service is unhealthy, erroring, or
serving incorrect data in production.

**Release impact:** Incident 1 shows checkout returning "incorrect totals for 30
minutes" — an outcome that a post-deploy health/functional check would have caught
immediately instead of relying on customer-facing symptoms to surface the failure.

**Recommended control:** Add a mandatory Health Verification Stage that runs automated
smoke tests and checks key SLIs (error rate, latency, checkout success rate) for a
defined bake-in window immediately after deployment, with explicit pass/fail criteria
before the release is declared complete.

---

## Weakness 5: No release ownership or accountability

**Evidence:** `examples/release-request-example.md` lists the release owner only as the
generic team alias `@devops-team`, not a named individual. `docs/deployment-history.md`
lists five releases with no "approved by" or "owned by" field in the log at all —
the "Missing information" section explicitly asks "Who approved each release?" as an
unanswered question.

**Operational risk:** When no individual is accountable for a release, decisions during
deployment (go/no-go, escalation, rollback) have no clear decision-maker, and
post-incident review cannot establish who should have caught the problem.

**Release impact:** Incident 4 ("Wrong version released," 2026-04-01) resulted in
"confusion, rollback, and missing release notes" — a direct symptom of no single owner
being responsible for confirming the correct version and documenting the release.

**Recommended control:** Require every release request to name a specific Release
Owner (individual, not a team alias) who is accountable end-to-end: confirming the
correct artifact/version, coordinating approval, executing deployment, and owning
rollback decisions if the release fails.

---

## Weakness 6: No release freeze / change-window discipline

**Evidence:** `deploy.yml` deploys on every push to `main` with no time-based or
condition-based restriction. `docs/current-release-process.md` and the release request
example show no concept of blackout periods, staggered rollout windows, or
high-risk-period restrictions.

**Operational risk:** High-risk releases (e.g., hotfixes, releases during peak traffic,
or releases stacked immediately after a prior failed release) can ship with the same
lack of scrutiny as routine changes, compounding risk during already-fragile periods.

**Release impact:** `docs/incident-log.md` notes a hotfix "bypassed validation gates and
caused customer-facing errors" (v1.3.6, 2026-04-01 per history) — evidence that
urgency-driven releases are treated identically to routine ones, with no additional
control applied during higher-risk conditions.

**Recommended control:** Define release freeze windows (e.g., peak traffic hours, after
a recent failed deployment, major incidents in progress) during which only
emergency-approved releases (with elevated approval requirements) may proceed.

---

## Weakness 7: No deployment visibility or standardized release record

**Evidence:** `docs/deployment-history.md` states: "Release notes are inconsistent
between entries" and "There is no consistent deployment checklist." The release request
example has no field for communication or stakeholder notification: "No stakeholder
notification process is described."

**Operational risk:** Teams outside the deploying engineer (support, on-call, adjacent
services) have no reliable way to know a release happened, what changed, or who to
contact if something looks wrong.

**Release impact:** Inconsistent release notes directly contributed to the confusion
recorded in Incident 4, and the lack of a standard record is called out as a systemic
gap across all five logged releases in `docs/deployment-history.md`.

**Recommended control:** Require a standardized release record (owner, version,
validation results, approval, rollback target, health check results) to be published to
a shared release channel/log for every production deployment, generated automatically
from the pipeline where possible.
