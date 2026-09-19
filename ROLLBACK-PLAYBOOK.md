# Rollback Playbook — Checkout Service

This playbook exists because, today, "rollback procedures exist only in people's
memories." Incident 2 (`docs/incident-log.md`) shows the direct cost of that gap: a
failed release required rollback, but "the team spent 45 minutes determining the
rollback procedure," extending customer downtime. This document removes the discovery
step from rollback entirely.

---

## Immediate Rollback Procedure

**What happened:** Health Verification fails during the post-deployment bake-in window,
or a critical customer-facing defect is confirmed to be caused by the just-completed
release.

**Trigger conditions:**
- Automated smoke tests fail against production after deployment.
- Error rate, latency, or checkout success rate breaches the defined threshold during
  the bake-in window (see `DEPLOYMENT-WORKFLOW.md`, Health Verification Stage).
- A critical customer-facing incident (e.g., incorrect totals, failed checkouts) is
  reported and correlates with the release timestamp.

**First actions:**
1. Release Owner immediately declares a rollback — no additional approval is required;
   authorization was pre-granted at the Release Approval Gate.
2. Deploy the pre-recorded rollback target (the last known good version documented in
   the release record) using the same pipeline used for deployment — never manual
   commands.
3. Confirm rollback deployment completes and re-run smoke tests against the restored
   version.
4. Post a status update to the release/incident channel noting rollback initiated and
   target version.

**Decision owner:** The **Release Owner** for that deployment. They do not need to wait
for the original approver — the ability to roll back is a standing authorization
attached to every approved release.

**How service is restored:** Production traffic is served by the previously validated,
known-good version once the rollback deployment's health checks pass.

---

## Partial Failure Procedure

**What happened:** A progressive/canary deployment shows failures on a subset of
traffic or instances, but the release has not fully rolled out or the failure is not
yet confirmed as release-caused.

**Detection method:**
- Canary/rolling deployment health probes report unhealthy instances.
- Monitoring shows anomalies isolated to the subset of traffic served by the new
  version, while the rest of the fleet remains healthy.

**Containment strategy:**
1. Halt the rollout immediately — do not proceed to 100% traffic.
2. Route traffic away from the unhealthy instances/canary back to the last known good
   version (traffic shifting, not a full redeploy, when the platform supports it).
3. Release Owner investigates logs/metrics from the affected instances to confirm
   whether the new release is the cause.

**Recovery path:**
- If confirmed release-caused: proceed to the Immediate Rollback Procedure for the
  remaining/affected instances.
- If not release-caused (e.g., unrelated infrastructure issue): document findings,
  resume rollout only after the unrelated issue is resolved and health checks are
  re-verified.

---

## Full Deployment Failure Procedure

**What happened:** The deployment itself fails to complete (e.g., the release cannot
reach a healthy state at all, or the failure is widespread and customer-impacting
across the full service, similar in severity to Incident 1's 30-minute impact window).

**Escalation process:**
1. Release Owner declares a **production incident** and pages the on-call Incident
   Commander (IC) if impact is customer-facing or ongoing beyond a few minutes.
2. IC takes overall command of the incident; Release Owner remains the technical
   point of contact for the release itself and executes the rollback.
3. If rollback does not resolve the issue within a defined SLA (e.g., 15 minutes),
   escalate to the Service Tech Lead and Engineering Manager on-call.

**Communication steps:**
1. Post an initial incident notice to the shared incident channel within 5 minutes of
   declaration: service, impact, start time, current status.
2. Provide updates at a fixed cadence (e.g., every 15 minutes) until resolved.
3. On resolution, post a closure notice with root cause summary (preliminary) and link
   to the release record.
4. Within 2 business days, complete a post-incident review and attach it to the release
   record, addressing the same gaps identified in `docs/deployment-history.md`
   ("Missing information": who approved, what was validated, was rollback confirmed).

**Recovery workflow:**
1. Execute rollback to the last known good version via the pipeline.
2. Verify health (smoke tests + monitoring) against the restored version before
   declaring the incident resolved.
3. Freeze further releases to this service until the post-incident review identifies
   and addresses the root cause (see release freeze process in
   `DEPLOYMENT-WORKFLOW.md`).
4. Update the release record and incident log with full timeline, decisions made, and
   time-to-restore — closing the documentation gap seen across all prior incidents in
   `docs/incident-log.md`.

---

## Principles Behind This Playbook

- **No discovery under pressure:** every rollback target and procedure is decided and
  documented *before* deployment, not during an incident (directly addressing
  Incident 2's 45-minute delay).
- **Pre-authorized action:** the Release Owner can execute rollback without seeking new
  approval, because rollback authority is granted alongside the original release
  approval.
- **Pipeline-only recovery:** rollback is always executed through the same controlled
  pipeline as deployment, never manual/ad hoc commands, to avoid the "wrong version"
  failure mode seen in Incident 4.
