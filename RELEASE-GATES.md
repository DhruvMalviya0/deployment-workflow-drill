# Release Gate Standards — Checkout Service

A release gate is a mandatory checkpoint that a release cannot pass without meeting a
defined, verifiable criterion. Today, `.github/workflows/deploy.yml` contains **zero**
gates between `git push` and production deployment. This document defines the gates
required to close that gap, referencing the specific incidents in
`docs/incident-log.md` that each gate would have prevented.

---

## Gate 1: Build Success Gate

- **Purpose:** Ensure only a successfully compiled, packaged artifact can proceed. No
  release should be based on an artifact that failed to build cleanly.
- **Validation criteria:** CI build job completes with exit code 0; a versioned,
  immutable artifact/container image is produced and stored in the registry.
- **Failure consequence:** Pipeline halts immediately; the release request cannot be
  created without a reference to a successful build run.
- **Responsible owner:** CI pipeline (automated enforcement); committing engineer is
  responsible for fixing build failures.
- **Why bypassing this creates risk:** Without this gate, an incomplete or broken
  artifact could reach staging or production, wasting downstream validation effort or,
  worse, deploying a non-functional service.

---

## Gate 2: Test Coverage / Test Success Gate

- **Purpose:** Ensure functional correctness is verified before a human ever reviews
  the release. Currently no test gate exists in the pipeline at all.
- **Validation criteria:** Unit and integration test suites pass at 100% (no skipped or
  failing tests); coverage does not regress below the team's defined minimum threshold
  for changed code.
- **Failure consequence:** Pipeline halts; release request cannot proceed to approval.
- **Responsible owner:** Service Tech Lead owns test suite adequacy; committing
  engineer owns making their change pass.
- **Why bypassing this creates risk:** Incident 3 (`docs/incident-log.md`) shows exactly
  this failure mode — "the deployment process did not require validation gates or build
  verification," resulting in a broken checkout flow and failed orders in production.

---

## Gate 3: Security Scan Gate

- **Purpose:** Prevent known-vulnerable dependencies or container images from reaching
  production.
- **Validation criteria:** Dependency and container image scan reports zero critical or
  high-severity vulnerabilities (or an approved, time-boxed exception is on file).
- **Failure consequence:** Pipeline halts until vulnerabilities are remediated or an
  explicit, documented exception is granted by the Service Tech Lead.
- **Responsible owner:** Security/Platform team defines scan policy; Service Tech Lead
  approves exceptions.
- **Why bypassing this creates risk:** With no scan gate today, a vulnerable dependency
  introduced in any change would ship straight to production with no detection point,
  turning a routine release into a security incident.

---

## Gate 4: Release Approval Gate

- **Purpose:** Introduce mandatory human authorization before production deployment —
  the single largest gap identified in the audit
  (`docs/current-release-process.md`: "No human approval step is required").
- **Validation criteria:** A designated Release Approver (not the change author) has
  explicitly approved the release request, which must include a named Release Owner,
  passing validation results, and a documented rollback target.
- **Failure consequence:** The deployment job is blocked at the environment level
  (e.g., GitHub Environment protection rule) and cannot execute without recorded
  approval.
- **Responsible owner:** On-call lead / designated Release Approver for the service.
- **Why bypassing this creates risk:** Incident 1 (`docs/incident-log.md`) happened
  because "a feature branch was deployed directly to production without proper
  validation" — a direct consequence of no approval checkpoint existing.

---

## Gate 5: Production Readiness Gate

- **Purpose:** Confirm operational preparedness (correct version, rollback plan,
  communication plan) immediately before deployment executes — distinct from code
  correctness, which earlier gates already covered.
- **Validation criteria:** Deployment targets the exact artifact version that passed
  Build Validation (no manual branch/version selection); rollback target is confirmed
  available; stakeholders/on-call are notified of the deployment window.
- **Failure consequence:** Deployment is blocked and returned to the Release Owner to
  correct the readiness gap before re-attempting.
- **Responsible owner:** Release Owner.
- **Why bypassing this creates risk:** Incident 4 ("Wrong version released") occurred
  because "production received the wrong version after manual deployment commands were
  executed" — this gate exists specifically to make manual version selection
  impossible.

---

## Gate 6: Health Verification Gate

- **Purpose:** Confirm the deployed release is actually healthy in production before it
  is declared successful — today, "deployment complete" is the only signal used
  (`.github/workflows/deploy.yml`).
- **Validation criteria:** Automated smoke tests pass against production; error rate,
  latency, and checkout success rate remain within threshold of baseline throughout the
  defined bake-in window.
- **Failure consequence:** The release is automatically flagged as failed and routed to
  the Recovery Stage / `ROLLBACK-PLAYBOOK.md`, regardless of whether the deploy command
  itself exited successfully.
- **Responsible owner:** Release Owner (executes/monitors); Platform/SRE team owns the
  monitoring thresholds.
- **Why bypassing this creates risk:** Incident 1 shows a deployment that technically
  "completed" but silently caused incorrect checkout totals for 30 minutes — a health
  gate is the only mechanism that would have caught this before it became a
  customer-facing incident lasting half an hour.

---

## Summary: Why Gates Cannot Be Bypassed

Every incident in `docs/incident-log.md` maps to exactly one missing gate above. A
process where gates can be manually skipped is operationally identical to having no
gates at all, because the skip will always be exercised under the exact conditions
(urgency, hotfixes, confidence) where the gate is needed most — as already demonstrated
by the hotfix that "bypassed validation gates and caused customer-facing errors"
(`docs/deployment-history.md`, v1.3.6). Gates must be enforced by the pipeline/platform,
not by convention.
