---
id: ADR-0003
title: Pipelines MultiKueue Plugin
date: 2026-08-06
status: proposed
authors:
  - "@pramodbindal"
reviewers:
  - "@vdemeester"
tags:
  - multicluster
  - multikueue
  - kueue
  - acm
---
<!-- Status lifecycle:
- Proposed: Under discussion, not yet accepted
- Accepted: Approved, ready for implementation
- Implemented: Done and reflected in the product
- Deprecated: Not required anymore
- Superseded: Replaced by a newer ADR (link to it)
- Rejected: Not accepted (stays in repo for reference)
-->
---

# ADR-0003: Pipelines MultiKueue Plugin

## Context

The MultiCluster feature shipped in OpenShift Pipelines v1.22.0 as Tech Preview. It lets Kueue
dispatch `PipelineRun`s from a hub cluster to worker (spoke) clusters through MultiKueue, but
onboarding is almost entirely manual today.

### Existing problem

- The Pipelines, Kueue and cert-manager operators must be installed by hand on the hub and on
  every spoke.
- The Kueue CR, queues, admission check, MultiKueue config and per-cluster kubeconfig secrets must
  be created by hand on the hub.
- A service account, its token and the matching RBAC must be created by hand on every spoke.
- All of the per-spoke steps must be repeated for every cluster added to the fleet, and redone
  whenever a spoke token is rotated.

Cluster management is the biggest cost here: the manual work grows linearly with fleet size, it is
easy to get wrong, and it slows down our own dev and QE environments as much as it slows down
users.

### Constraints

- The fleet admin provides the clusters and their credentials; the plugin must not invent cluster
  membership of its own.
- Kueue must never be handed broad cluster credentials — MultiKueue only needs a narrowly scoped
  identity on each spoke.
- Spoke credentials expire and are rotated by the platform; the hub-side kubeconfig secrets have
  to follow the rotation automatically.
- Operators are installed through OLM, which is asynchronous: bootstrap has to tolerate resources
  that are not ready yet, and every step has to be idempotent because reconciliation repeats.
- The hub and spokes are separate API servers; the controller needs a distinct client per spoke.

### Business goals driving this change

Manual onboarding is the main adoption blocker for taking MultiCluster beyond Tech Preview, so it
needs to be solved before the feature is productized (target: OpenShift Pipelines 1.25.0).

### Systems, teams or processes impacted

OpenShift Pipelines operator, Kueue operator, cert-manager operator, Advanced Cluster Management
(ACM/OCM) on the hub, and the QE and dev workflows that build multicluster environments.

## Decision

We will automate MultiCluster fleet setup with a dedicated hub-side Kubernetes controller, the
**Pipelines MultiKueue Plugin**, shipped as its own component from
[openshift-pipelines/pipelines-multikueue-plugin](https://github.com/openshift-pipelines/pipelines-multikueue-plugin),
so that the only user action left is importing a cluster into the fleet.

- Chosen option: `Dedicated hub-side controller` over `documented manual steps and scripts`,
  `logic inside the OpenShift Pipelines operator`, and `ACM policies`.

The first implementation targets ACM/OCM hubs and uses the `ManagedCluster` API as its source of
cluster membership.

### Architectural Overview

The controller is a controller-runtime manager with two units:

- **Hub bootstrap** (`ClusterBootstrap`) — a manager runnable that configures the hub once at
  startup.
- **Cluster reconciler** (`MultiKueueReconciler`) — reconciles one `ManagedCluster` at a time and
  configures the corresponding spoke through a per-spoke client.

Both clusters are bootstrapped by the same code path; a `ClusterRole` field (`Spoke` or unset for
the hub) selects the few steps that differ, so the two environments cannot drift apart in what
they configure.

Watches:

- `ManagedCluster` — primary resource. Clusters labelled `local-cluster=true` are skipped.
- `ManagedServiceAccount` and the token `Secret`s it owns — mapped back to the owning cluster, so
  an ACM-driven token rotation re-renders the MultiKueue kubeconfig secret.

### Implementation Details

**Reconciled on the hub:**

- OLM `Subscription`s (creating the namespace and `OperatorGroup` when missing) for
  `openshift-pipelines-operator-rh`, `kueue-operator` and `openshift-cert-manager-operator`.
  A package that is already subscribed anywhere on the cluster is reused rather than duplicated.
- The `Kueue` CR `cluster`, registering `tekton.dev/v1 pipelineruns` as an external framework under
  both `integrations` and `multiKueue`. An existing CR is patched, appending the Tekton framework
  only if absent, so user-configured frameworks survive.
- `ResourceFlavor` `default-flavor`, `ClusterQueue` `pipelines-multicluster-kueue` (quota 100 of
  `tekton.dev/pipelineruns`), `LocalQueue` `pipelines-queue` in `default`, and `AdmissionCheck`
  `pipelines-multikueue-ac` wired to the MultiKueue config.
- Per spoke: a `ManagedServiceAccount` `pipelines-multikueue` (24h TTL, 24h rotation), a kubeconfig
  `Secret` built from its token, a `MultiKueueCluster`, and the cluster's entry in
  `MultiKueueConfig` `pipelines-multikueue-config`.

**Reconciled on each spoke:**

- A `ClusterRole`/`ClusterRoleBinding` delivered through a `ManifestWork`, granting the managed
  service account only what the rest of the bootstrap and MultiKueue need (namespaces, OLM
  resources, `kueue.openshift.io/kueues`, `kueue.x-k8s.io/workloads`, `batch/jobs` watch,
  `tekton.dev/pipelineruns`, and `get`/`patch` on `operator.tekton.dev/tektonconfigs`).
- The same three operators and the same `Kueue` CR as on the hub, applied through the scoped
  managed-service-account client.
- The singleton `TektonConfig` named `config`, patched to enable the Tekton scheduler
  (`spec.scheduler.disabled: false`) so Kueue can admit `PipelineRun`s there. The patch is a JSON
  merge patch against an unstructured object, deliberately avoiding a dependency on the Tekton
  operator Go types.

**Affected components/systems:** pipelines-multikueue-plugin (new), Kueue operator, OpenShift
Pipelines operator (`TektonConfig`), cert-manager operator, ACM/OCM.

## Security & Compliance

- The fleet admin's credentials are never given to Kueue. The controller provisions a dedicated
  service account on each spoke via ACM's `ManagedServiceAccount`, and only that scoped token is
  imported into the hub-side kubeconfig secret that MultiKueue consumes. This is a smaller blast
  radius than the credentials a human would otherwise paste into a secret by hand.
- Spoke RBAC is an explicit allow-list shipped as a `ClusterRole` through `ManifestWork`, limited
  to what bootstrap and MultiKueue need.
- Tokens have a 24h TTL and are rotated every 24h by ACM; the controller re-renders the hub-side
  kubeconfig secret on rotation, so expired credentials are not left in place.
- New attack surface: the controller itself is highly privileged. It installs operators and
  creates cluster-scoped resources on the hub *and* on every managed cluster, and it holds the
  ability to mint scoped identities across the fleet. Compromise of the hub controller is
  equivalent to compromise of fleet-wide configuration authority.
- No PII is handled. The only secret material is the per-spoke service account token, stored in
  hub-side `Secret`s that MultiKueue already requires.
- Security sign-off has not yet been obtained and should be part of accepting this ADR.

## Compatibility & Migration

- MultiCluster is Tech Preview in 1.22.0; this plugin targets productization in 1.25.0. There is
  no supported installed base to migrate, so no compatibility guarantee is broken.
- Existing manually configured environments are tolerated rather than converted: an existing
  `Kueue` CR is patched in place and the Tekton framework appended only if absent, and an existing
  operator subscription for a required package is reused rather than duplicated.
- The plugin takes ownership of shared cluster state (`Kueue` CR, `TektonConfig`) on clusters that
  may already be in use. Users who configured those objects by hand may see their values
  supplemented, though not replaced.
- Rollback: uninstalling the controller stops further reconciliation, but nothing it created is
  removed today — there is no cleanup on cluster detach or plugin uninstall. A clean rollback
  requires the finalizer work listed under Consequences.

## Testability

- Unit coverage for the reconcilers and bootstrap steps landed with
  [SRVKP-13456](https://issues.redhat.com/browse/SRVKP-13456).
- Integration and e2e coverage on real hub/spoke clusters is still outstanding and is the main
  testing gap for productization.
- Critical paths that need e2e coverage: first-time cluster import, token rotation re-rendering
  the kubeconfig secret, reuse of a pre-existing operator subscription, patching a pre-existing
  `Kueue` CR without dropping user frameworks, and a spoke whose OLM catalog is slow or broken.

## Operational Readiness

- Bootstrap waits inline for each OLM subscription to report an installed CSV (up to 30 minutes),
  so a slow or broken catalog blocks that cluster's reconcile. This is the most likely production
  failure mode and needs to be visible rather than silent.
- Observability is not yet defined beyond controller-runtime defaults. Before productization the
  plugin should surface per-cluster bootstrap state (conditions and/or events on `ManagedCluster`)
  and metrics for reconcile failures and time spent waiting on OLM.
- Support impact: a failure here blocks onboarding of a cluster rather than breaking running
  workloads, so it should not be a paging-class failure, but it does need a clear diagnostic path
  for support.

## Scope

- **In-Scope:**
  - Automated hub and spoke bootstrap: operator installation, `Kueue` CR, queues, admission check,
    MultiKueue wiring and spoke RBAC.
  - Per-spoke scoped credential provisioning and continuous rotation handling.
  - ACM/OCM hubs, using `ManagedCluster` as the source of fleet membership.
- **Out-of-Scope:**
  - Non-ACM fleets and alternative cluster-discovery mechanisms.
  - Cleanup on cluster detach or plugin uninstall.
  - Operator lifecycle beyond initial install (upgrades, ownership of pre-existing installs).
  - Configurable queue names, quotas, channels, and namespace management strategy.
  - Scheduling policy itself — how Kueue chooses a spoke is Kueue's concern, not the plugin's.

## Acceptance Criteria

The decision is considered accepted when:

- [ ] Problem, scope, and non-goals are clear
- [ ] Alternatives were considered and compared
- [ ] Chosen option's trade-offs (not just benefits) are documented
- [ ] Backward compatibility / migration path addressed
- [ ] Security/RBAC impact assessed
- [ ] Observability and rollback plan defined
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked
- [ ] Decision is feasible to implement
- [ ] Documented in internal knowledge base

## Considered Options

| Option | Pros | Cons |
| ------ | ---- | ---- |
| Dedicated hub-side controller (chosen) | Onboarding reduces to importing a cluster; token rotation handled continuously; hub and spoke share one bootstrap path; scoped credentials by construction | A new component to build, ship and support; highly privileged; takes ownership of shared cluster state; ACM dependency |
| Document the manual steps and ship scripts | No new component; quick to deliver | Toil grows with fleet size; does not handle token rotation or later cluster imports at all; error-prone |
| Put the logic in the OpenShift Pipelines operator | No new deliverable; reuses an existing install path | The operator is a per-cluster component with no view of the fleet; pulls ACM/OCM APIs into the core operator for a feature most installations do not use |
| Express the fleet configuration as ACM policies | Uses an existing ACM mechanism; no new controller | Bootstrap is sequenced and stateful (wait for a CSV, wait for a token, patch existing CRs in place), which is awkward as policy; leaves no good place for hub-side MultiKueue wiring |
| Do Nothing | No effort required | MultiCluster stays manual, which is the main blocker to taking it beyond Tech Preview |

## Decision Drivers

- Adoption: manual onboarding is the main blocker to productizing MultiCluster.
- Security: Kueue must hold only a narrowly scoped spoke identity, never fleet admin credentials.
- Operational cost: setup effort must not grow linearly with fleet size, and token rotation must
  not be a manual repair task.
- Maintainability: one bootstrap code path for hub and spoke, kept out of the per-cluster
  Pipelines operator.
- Team expertise and the existing ACM footprint on hub clusters.

## Consequences

- Positive outcomes:
  - Enabling MultiCluster becomes "import the cluster"; the lengthy manual setup guide disappears.
  - Dev and QE get the same one-step setup, so multicluster environments are cheaper to create and
    recreate.
  - Token rotation is handled continuously instead of being a manual repair task.
  - Kueue only ever holds a narrowly scoped spoke identity.
  - Hub and spoke share one bootstrap code path, so the two environments cannot drift apart.
- Risks:
  - The controller is highly privileged on the hub and on every managed cluster.
  - It takes ownership of shared cluster state (`Kueue` CR, `TektonConfig`) on clusters that may
    already be in use, and today's defaults (queue names, the quota of 100, the `default`
    namespace for the local queue) are not configurable.
  - The inline wait for OLM (up to 30 minutes per subscription) blocks a cluster's reconcile when
    a catalog is slow or broken.
  - The ACM dependency is real: without `ManagedCluster`, `ManagedServiceAccount` and
    `ManifestWork` the controller cannot discover or reach a spoke. Non-ACM fleets are unsupported.
  - One more component to build, ship, support and keep in step with Kueue, ACM and the Pipelines
    operator.
- Technical debt or follow-up ADRs required:
  - Nothing is cleaned up when a cluster leaves the fleet — the `MultiKueueCluster` and the entry
    in `MultiKueueConfig` stay behind. Handle detach and uninstall via finalizers.
  - Support non-ACM fleets by discovering spokes from labelled/annotated kubeconfig secrets,
    keeping the same reconcilers behind a swappable discovery mechanism
    ([SRVKP-12921](https://issues.redhat.com/browse/SRVKP-12921) covers the credential model).
  - Take ownership of the operators the plugin installs: owner references, upgrades by moving the
    subscription channel, and leaving pre-existing installations alone
    ([SRVKP-13217](https://issues.redhat.com/browse/SRVKP-13217)).
  - Install the plugin on the hub automatically when ACM is enabled in the scheduler configuration
    ([SRVKP-12890](https://issues.redhat.com/browse/SRVKP-12890)).
  - Decide the namespace management strategy: which namespaces get a `LocalQueue`, and how users
    opt additional namespaces in.
  - Make the queue, quota and channel defaults configurable.
  - Add integration and e2e coverage on real clusters.

## Dependencies

- ACM/OCM on the hub for the `ManagedCluster`, `ManagedServiceAccount` and `ManifestWork` APIs.
- Kueue operator (`kueue.openshift.io`) with MultiKueue and external-framework support for
  `tekton.dev/v1 pipelineruns`.
- cert-manager operator, required by Kueue.
- OpenShift Pipelines operator with the Tekton scheduler (`TektonConfig.spec.scheduler`) on hub and
  spokes; target release OpenShift Pipelines 1.25.0.
- OLM for operator installation on every cluster in the fleet.

## References

- [ADR-0000: ADR template](./0000-adr-template.md)
- [Implementation: openshift-pipelines/pipelines-multikueue-plugin](https://github.com/openshift-pipelines/pipelines-multikueue-plugin)
- [JIRA SRVKP-12898: ACM Pipelines Addon controller implementation (Epic)](https://issues.redhat.com/browse/SRVKP-12898)
- [JIRA SRVKP-14524: Phase 1 — Pipelines Controller for Multicluster Operator Provisioning](https://issues.redhat.com/browse/SRVKP-14524)
- [Kueue MultiKueue documentation](https://kueue.sigs.k8s.io/docs/concepts/multikueue/)

## Change Log

| Date       | Author         | Change Summary                                      |
| ---------- | -------------- | --------------------------------------------------- |
| 2026-08-06 | @pramodbindal  | Created initial ADR                                  |
| 2026-10-28 | @pramodbindal  | Restructured to the current ADR template             |

---
