# Controlled Deployment Workflow — Checkout Service

This document defines the complete, controlled release movement process that replaces
the direct-push-to-production model described in `docs/current-release-process.md`. It
is designed to directly close the gaps identified in `DEPLOYMENT-AUDIT.md`.

Release movement, at a glance:

```text
Code Commit
    ↓
Build Validation Stage
    ↓
Release Approval Stage
    ↓
Deployment Execution Stage
    ↓
Health Verification Stage
    ↓
Production Success
```

A release may only move forward through this sequence. It may never skip a stage, and
any stage may send the release to the **Recovery Stage** instead of forward.

---

## 1. Build Validation Stage

**Purpose:** Guarantee that only code which is built, tested, and scanned reaches a
human approver. This directly replaces the current pipeline, which has "No validation
gate or approval step configured" (`deploy.yml`).

**What gets validated:**
- Build succeeds and produces a versioned, immutable artifact/container image.
- Unit and integration test suites pass with no failures.
- Static analysis / lint checks pass.
- Dependency and container security scan finds no critical/high vulnerabilities.
- The artifact is deployed to a staging environment and passes automated smoke tests
  (checkout flow, payment validation, inventory checks — the exact area that failed in
  Incidents 1 and 3).

**Who owns validation:** The CI pipeline is the enforcement mechanism; the **committing
engineer** is responsible for ensuring their change passes it, and the **Service Tech
Lead** owns the validation suite's coverage and correctness.

**What blocks release progression:**
- Any failed test, failed build, or critical/high security finding blocks the pipeline
  automatically — no manual override is available at this stage.
- A release cannot be submitted for approval without a green Build Validation run
  attached (referenced by commit SHA / run ID).

---

## 2. Release Approval Stage

**Purpose:** Introduce the human sign-off that is entirely absent today
(`docs/current-release-process.md`: "No human approval step is required").

**Who approves:**
- Standard releases: one **Release Approver** — the on-call lead or designated
  reviewer for the checkout service, who must not be the same person who authored the
  change (no self-approval).
- High-risk releases (hotfixes, releases touching payment/inventory logic, releases
  during a freeze window): **two approvers**, including the Service Tech Lead.

**Approval criteria:**
- Build Validation Stage passed and is linked in the release request.
- A named **Release Owner** is assigned (see Deployment Execution Stage) — closes the
  ownership gap seen in `docs/deployment-history.md`.
- A rollback target (last known good version) is documented.
- The release request follows the standard record template (see Section 5 below),
  replacing the incomplete template in `examples/release-request-example.md`.
- The release is not scheduled inside an active release freeze window, unless
  emergency-approved.

**Escalation process:**
- If the primary Release Approver is unavailable within a defined SLA (e.g., 1 hour for
  standard releases, 15 minutes for emergency fixes), the request escalates to the
  Service Tech Lead, then to the Engineering Manager on-call.
- Emergency/hotfix releases require verbal or written sign-off from the Engineering
  Manager on-call in addition to the standard approver, and must be retroactively
  documented within 24 hours.

---

## 3. Deployment Execution Stage

**Purpose:** Make deployment a controlled, sequenced, owned action rather than an
automatic side effect of a `git push` (as seen in
`examples/release-request-example.md`).

**Deployment sequence:**
1. Deploy the approved, validated artifact to production using the exact version that
   passed Build Validation (no manual branch selection — this directly prevents the
   "Wrong version deployed" failure in Incident 4).
2. Deploy using a progressive strategy where infrastructure allows (canary or rolling
   deployment) rather than an all-at-once cutover.
3. Tag the deployed release in the deployment system with version, commit SHA, and
   timestamp.

**Release ownership:**
- The named **Release Owner** (assigned during approval) executes or directly
  supervises the deployment and is accountable for the outcome end-to-end, closing the
  "no release ownership" gap where releases were only attributed to `@devops-team`.

**Production release procedure:**
- Deployment is triggered only through the pipeline (never manual `kubectl`/ad hoc
  commands) so the executed steps are identical every time and fully logged.
- The Release Owner confirms deployment completion against the deployment system's
  status, not merely against the CI job exiting with code 0.
- The release record is updated with deployment start/end time and the artifact version
  actually deployed.

---

## 4. Health Verification Stage

**Purpose:** Replace the current practice of declaring success once "deployment
commands finish" (`docs/current-release-process.md`) with actual evidence the service
is healthy.

**Verification checks:**
- Automated smoke tests against production endpoints (checkout, payment validation,
  inventory) immediately after deployment.
- Error rate, latency (p95/p99), and checkout success rate monitored against baseline
  for a defined bake-in window (minimum 15 minutes for standard releases).
- Log-based check for new/elevated error signatures introduced by the release.

**Success criteria:**
- Zero smoke test failures.
- Error rate and latency remain within pre-defined thresholds of pre-deployment
  baseline throughout the bake-in window.
- No new critical alerts fire during the bake-in window.

**Monitoring requirements:**
- The Release Owner actively monitors dashboards during the bake-in window (this is not
  a passive/background task).
- Health Verification results (pass/fail, metrics snapshot) are attached to the release
  record — this is the missing evidence flagged in `docs/deployment-history.md`
  ("Verification records are missing for most successful deployments").
- If any success criterion is not met, the release automatically moves to the Recovery
  Stage — it is not left to individual judgment whether to consider the release
  successful.

---

## 5. Recovery Stage

**Purpose:** Give engineers a pre-defined path to follow instead of reacting from
memory, which caused the 45-minute rollback delay in Incident 2.

**Failure handling process:**
- Any failure detected in Build Validation, Approval, Deployment, or Health
  Verification immediately halts forward progress and invokes the corresponding
  procedure in `ROLLBACK-PLAYBOOK.md`.
- The Release Owner declares the incident and begins execution of the rollback
  procedure without waiting for additional approval — rollback authorization is
  pre-granted by the original release approval.

**Rollback trigger conditions:**
- Health Verification fails any success criterion during the bake-in window.
- A critical customer-facing defect is reported post-deployment and attributed to the
  release.
- Deployment Execution fails to complete (partial rollout, failed health probes at the
  infrastructure level).

**Incident ownership:**
- The **Release Owner** is the incident owner by default and directs the rollback.
- For full-scale incidents, ownership transfers to the on-call Incident Commander per
  `ROLLBACK-PLAYBOOK.md`, with the Release Owner remaining the technical point of
  contact for the change itself.

---

## Standard Release Record Template

Every release must be tracked using a record containing, at minimum:

- Service, target environment, requested release date
- Release Owner (named individual) and Release Approver(s)
- Change summary and linked commit/PR
- Build Validation run reference (pass/fail, scan results)
- Rollback target version
- Deployment start/end time and deployed version
- Health Verification results and bake-in window outcome
- Stakeholder communication log (who was notified, when)

This replaces the incomplete `examples/release-request-example.md` template, which is
missing approval, validation, rollback, and communication fields.
