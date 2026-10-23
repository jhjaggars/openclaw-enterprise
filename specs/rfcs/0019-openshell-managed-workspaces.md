---
status: Proposed
implementation_status: Not implemented
author: jhjaggars
---

# Proposal: managed OpenShell workspaces for operator-owned sandbox clusters

- **ID:** RFC-0019
- **Created:** 2026-10-23
- **Last updated:** 2026-10-23
- **Related:** [OpenShell SandboxDriver](../../docs/reference/drivers/openshell-sandbox.md),
  [OpenShell Backend](../../docs/reference/backends.md#openshell-gateway),
  [RFC-0016 sandbox credential injection](0016-sandbox-credential-injection.md),
  [TASK-0045 external gateway plan](../plans/0045-openshell-external-gateway-oidc.md)

## Summary

Support `workspaceMode: managed` in the bundled OpenShell SandboxDriver. OCC
creates one OpenShell Workspace per OCC Namespace through the gateway's RPC,
and the gateway owns every Kubernetes object on the sandbox side: the
per-workspace namespace, the sandbox ServiceAccount, and sandbox policies. OCC
holds no Kubernetes access to the sandbox cluster at all — only the
authenticated gateway connection the `openshell` Backend already owns.

## Motivation

Operator-owned OpenShell deployments run the gateway in a cluster OCC does not
control and cannot pre-provision. The current driver implements operator
workspace mode only: `ensureNamespace` creates and labels the Kubernetes
namespace, applies workspace resources and NetworkPolicies, and the v0.1.3-pre.2
contract requires the OpenShell Workspace name to equal that pre-provisioned
namespace. Managed mode is recognized by configuration but fails closed before
any mutation (`requireOperatorWorkspaceMode` in
`apps/controller/src/drivers/sandbox/openshell.ts`), a deliberate deferral while
the trust boundary was unqualified.

OpenShell 0.1.5 defines managed mode concretely: the gateway's Kubernetes
driver creates and deletes per-workspace namespaces named
`openshell-{gateway_id}-{workspace}`, creates the sandbox ServiceAccount
inside them, and applies its own sandbox policies; all workspace lifecycle is
driven through the gateway RPC, which now carries a workspace role model
(`workspace:write` scope, workspace `user`/`admin` roles, `platform_admin`).
The RPC set OCC uses — workspace CRUD, sandbox CRUD, service exposure,
providers, logs — is retained from v0.1.3-pre.2 with field-level deltas.

The external-gateway trial under TASK-0045 has already proven the connection
half against a live 0.1.5 gateway: OIDC client-credentials Backend
authentication, token caching, and gRPC Health over TLS.

## Goals

- A trusted Installation selects `workspaceMode: managed`; the driver performs
  the whole workspace lifecycle through the gateway RPC and performs no
  Kubernetes mutations.
- No kubeconfig, ServiceAccount, or RBAC for the sandbox cluster appears
  anywhere in the Installation or the OCC control plane.
- Harness readiness and cleanup use gateway-reported state instead of Pod
  observation.
- The managed path targets OpenShell 0.1.5 semantics for the RPCs OCC uses.
- Proof against a real external 0.1.5 gateway: workspace create and adoption,
  sandbox creation, a dedicated harness model turn, and cleanup.

## Non-goals

- OpenShell `shared` workspace mode.
- Changing operator mode, which stays fenced to v0.1.3-pre.2 semantics; the
  0.1.5 operator namespace allowlist is a separate follow-up.
- The `executionCluster` two-cluster compute path, which remains the way to
  run OCC-owned harnesses in a remote cluster.
- Any OCC-owned object in the sandbox cluster.

## Proposal

### Workspace lifecycle

The driver accepts `workspaceMode: managed` and rejects operator-only options
in that mode (`operatorNamespaceLabels`, `operatorWorkspaceResources`,
`networkPolicyResources`): with no Kubernetes access, OCC cannot apply them,
and silently ignoring them would misstate the boundary. The workspace name
keeps the existing stable scheme — `oce-` plus a digest of the OCC Namespace —
which already satisfies the DNS-1123 and length constraints and keeps the
gateway-derived namespace within the 63-character limit.

- `ensureNamespace` in managed mode reconciles the Workspace over RPC:
  `GetWorkspace`, then `CreateWorkspace` when absent, with the existing
  request-replay handling. It performs no Kubernetes mutation and must not
  require the compute driver's Kubernetes object client.
- `provisionHarness` is unchanged in shape — `CreateSandbox` in the Workspace,
  with the service exposure and provider attachments as today — but readiness
  and route validation come from `GetSandbox` and `GetService` rather than Pod
  observation.
- `cleanup` deletes the Sandbox, and Namespace retirement additionally deletes
  the Workspace; the gateway deletes its namespace.

### Trust boundary

In operator mode OCC owns the namespace, its RBAC, and its policies, and the
gateway reconciles inside them. In managed mode the gateway owns all three,
and OCC enforces tenancy at the RPC layer instead: one OpenShell Workspace per
OCC Namespace, with OCC's gateway identity admitted through the gateway's own
workspace roles. OCC can no longer verify the harness Pod through the
Kubernetes API; the driver trusts gateway-reported sandbox state, and
qualification moves from local cluster proof to live-gateway proof. Credential
delivery is unchanged: providers are registered over the Backend's protected
transport (TLS with OIDC client-credentials), preserving RFC-0016's guarantees.

### Compute integration

Kubernetes Compute keeps owning its control-plane namespace — Agent Gateway,
routing, and revision management — in OCC's own cluster. The sandbox driver's
`harnessEndpoint` capability already routes the Agent Gateway to the
OpenShell-advertised origin instead of a local Service; the managed path makes
that the only harness route. Pod-observation readiness in the provider-fenced
path is replaced by gateway status. Agent Gateway egress must admit the
advertised service endpoints by their resolved addresses, extending the
external-gateway policy work from TASK-0045.

### Wire compatibility

The driver today is fenced to v0.1.3-pre.2 wire semantics. The managed path is
fenced to 0.1.5 semantics for the RPCs OCC uses, verified field-by-field
against the 0.1.5 protos: workspace CRUD, sandbox CRUD and request replay,
service exposure authorization modes, provider profiles and credentials, and
logs. Divergent behaviors — admission limits, replay reasons, exposure
authorization — are version-fenced rather than mixed.

### Failure behavior

Gateway unavailability keeps the existing transient-dependency handling.
Uncertain create effects keep the request-ID retry and adoption semantics,
re-verified against 0.1.5. A workspace that fails creation fails OCC Namespace
admission; a workspace deleted out from under OCC fails sandbox provisioning
closed rather than recreating tenant state implicitly.

## Qualification

The qualification contract's operator-mode items that relied on OCC-applied
resources — workload identity, writable runtime state bounds, sandbox
policies — become gateway guarantees that must be proven live: supervisor
credentials inaccessible to the Agent, bounded writable state, provider file
lifecycle, and the control-plane/service port separation blocker. The live
external gateway from the TASK-0045 trial is the proof vehicle.

## Open questions

- The trial gateway is a downstream build of 0.1.5; its divergence from
    upstream in workspace and RPC behavior must be confirmed with its
    operators.
- Whether OCC's gateway identity operates as `platform_admin` or per-workspace
    roles provisioned through `AddWorkspaceMember`; per-workspace roles are
    the tighter fit for one-workspace-per-Namespace tenancy.
- The gateway identifier's length budget for derived namespace names.

## Implementation plan

To be allocated under `specs/plans/` when this RFC's direction is accepted.
