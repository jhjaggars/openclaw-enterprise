---
rfc: ../rfcs/0019-openshell-managed-workspaces.md
---

# Implementation plan: managed OpenShell workspaces

- **ID:** TASK-0046
- **Delivery status:** Planned
- **Owner:** Enterprise platform
- **Authority:** [RFC-0019 managed OpenShell workspaces](../rfcs/0019-openshell-managed-workspaces.md)
- **Source baseline:** `9ffd3059d` (branch `openshell-external-gateway-oidc`)

## Outcome and scope

The bundled OpenShell SandboxDriver supports `workspaceMode: managed`: the
whole workspace lifecycle runs through the gateway RPC, the gateway owns every
Kubernetes object on the sandbox side, and the Installation carries no
kubeconfig or RBAC for the sandbox cluster. The connection is the
`openshell` Backend's OIDC client-credentials mode proven in TASK-0045.

Non-goals: OpenShell `shared` mode, operator-mode changes, and the
`executionCluster` path are unchanged.

## Contract and source touchpoints

Evidence base: the live external OpenShell 0.1.5 gateway trial (private log;
neutral in repository artifacts) proved the OIDC Backend, Health,
GetCurrentUser, and the full workspace lifecycle
(create → active → get → delete → NOT_FOUND) with an admin-role client
identity. OpenShell 0.1.5 managed mode auto-creates namespaces named
`openshell-{gateway_id}-{workspace}` with the sandbox ServiceAccount.

- `apps/controller/src/drivers/sandbox/openshell.ts` —
  `requireOperatorWorkspaceMode` currently fails managed mode closed; the
  driver's `ensureNamespace`/`cleanup` own operator-mode Kubernetes mutations.
- `apps/controller/src/drivers/sandbox/openshell-gateway-client.ts` — the
  wire client is fenced to v0.1.3-pre.2 semantics with its bundled proto.
- `apps/controller/src/drivers/compute/kubernetes/index.ts` —
  `sandboxNamespaceContext` hands the driver the compute plane's Kubernetes
  client; `providerHarnessReady` observes Pods for provider-fenced harnesses.
- `apps/controller/src/drivers/sandbox/openshell-oidc-token.ts` — the
  client-credentials token source from TASK-0045.

Selected behavior:

- Managed mode reconciles the Workspace over RPC (`GetWorkspace`, then
  `CreateWorkspace` with request-replay handling) using the existing
  `oce-`+digest name; `ensureNamespace` performs no Kubernetes mutation and
  must not require the Kubernetes object client.
- Managed mode rejects `operatorNamespaceLabels`,
  `operatorWorkspaceResources`, `networkPolicyResources`, and the operator
  `readiness` Service observation: OCC cannot apply or observe them.
- Harness readiness and route validation use `GetSandbox` and `GetService`
  responses only. Cleanup deletes the Sandbox, and OCC Namespace retirement
  additionally deletes the Workspace.
- The managed path is fenced to OpenShell 0.1.5 semantics for the RPCs OCC
  uses; operator mode stays v0.1.3-pre.2. The bundled wire proto gains the
  0.1.5 field shapes the managed path requires, verified against the 0.1.5
  protos.
- The OCC client identity requires the gateway's admin role for workspace
  create/delete (scope enforcement is off in the target deployment); document
  the role requirement for operators.

## Implementation

1. Driver configuration: accept `managed`, reject operator-only options in
   that mode with the fail-closed message naming the boundary.
2. Managed workspace reconciliation in `ensureNamespace` (RPC only) and
   Workspace deletion in Namespace cleanup.
3. Readiness and `harnessEndpoint` validation from gateway-reported state;
   remove the Kubernetes Pod observation from the managed path.
4. Wire client: 0.1.5 field-shape pass for the managed RPC set (workspace
   create/get/delete, sandbox create/get/delete, service exposure, providers,
   logs), keeping the pre.2 operator path byte-identical.
5. Credential Gateway Driver: verify provider RPCs unchanged against 0.1.5;
   adjust only where field shapes moved.
6. Tests: conformance coverage for managed-mode driver behavior against the
   in-process wire server; `sandbox-driver-startup` admission and rejection
   cases; the real external gateway remains the live proof vehicle.
7. Documentation: SandboxDriver reference (managed mode section, identity and
   role requirements), the external-gateway guide, and the provisioning flow
   doc.

## Verification

| Required outcome                                 | Real check and prerequisites                                        | Result or remaining proof |
| ------------------------------------------------ | ------------------------------------------------------------------- | ------------------------- |
| OIDC Backend + workspace lifecycle on live gateway | Trial probes (TASK-0045 log)                                       | Pass                      |
| Managed mode admitted; operator-only options rejected | `sandbox-driver-startup` integration test                      | Pass (2 tests)            |
| Workspace reconcile/create/delete over RPC        | Conformance test with in-process wire server                        | Pass (driver-level)       |
| Harness provisioning + readiness without Kubernetes reads | Conformance test + live gateway                           | Not run                   |
| Real model turn through a managed workspace       | Live external gateway with an authorized model credential          | Not run                   |

## Open decisions

None blocking; the 0.1.5 field-shape deltas are enumerated during step 4 and
recorded here if any require a design choice.

## Delivery record

- 2026-10-23 (branch `openshell-external-gateway-oidc`):
  - Step 1 landed: `validateOptions` rejects the four operator-only gateway
    options in managed mode with a boundary-naming message; `ensureNamespace`
    dispatches on workspace mode (operator applies labels, resources, and
    readiness; managed reconciles the Workspace over RPC only); Namespace
    cleanup keeps the RPC Workspace deletion in both modes and skips the
    OCC-applied resource deletion in managed mode. Harness provisioning,
    harness endpoints, revision cleanup, and log reads remain fail-closed in
    managed mode pending steps 3-4.
  - Tests: the former fail-closed managed test became the managed reconcile
    and cleanup behavior test (no Kubernetes events, RPC-only workspace
    lifecycle, idempotent retry); a second test covers managed admission and
    rejection of each operator-only option through Installation loading.
    `sandbox-driver-startup` and `openshell-gateway-wire` pass (56 tests),
    plus `pnpm typecheck`, `pnpm lint`, and `pnpm format:check`.

## Manual Notes

[keep this for the user to add notes. do not change between edits]

## Changelog

- 2026-10-23: Initial plan from RFC-0019 and the live external-gateway trial
  evidence (`9ffd3059d`).
