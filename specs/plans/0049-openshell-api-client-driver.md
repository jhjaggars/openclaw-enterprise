---
rfc: null
---

# Implementation plan: OpenShell sandbox driver as a gateway API client

- **ID:** TASK-0049
- **Delivery status:** Planned
- **Owner:** Enterprise platform
- **Authority:** current architecture — [OpenShell Backend contract](../../docs/reference/backends.md#openshell-gateway); supersedes the managed-mode scope of RFC-0019 / TASK-0046
- **Source baseline:** `702ce0fd6` (branch `openshell-external-gateway-oidc`)

## Outcome and scope

The bundled OpenShell SandboxDriver becomes a thin client to the OpenShell
gateway RPC API. It creates workspaces and sandboxes, resolves service
endpoints, reads logs, and deletes what it created — all through the Backend's
authenticated gRPC connection. The driver has no workspace-mode awareness, no
Kubernetes mutation in the sandbox cluster, and no infrastructure
configuration. How the gateway internally organizes namespaces, storage, and
isolation is the gateway's business.

Non-goals: OpenShell gateway lifecycle management (install, upgrade, remove),
workspace-mode-specific behavior, Kubernetes resource application in the
sandbox cluster, and storage layout management.

## Contract and source touchpoints

- `apps/controller/src/drivers/sandbox/openshell.ts` — the driver refactor
- `packages/contracts/src/index.ts` — simplify the options type
- `apps/controller/src/composition/installation-config.ts` — composition
- `apps/controller/src/drivers/compute/kubernetes/index.ts` — the
  `harnessReady` hook and remote `providerHarness` remain; their dispatch is
  no longer conditional on sandbox driver workspace mode
- Tests and docs affected by the contract change

The OIDC Backend auth (`oidcClientCredentials`), the wire client, the dev
launcher external gateway support, and the remote `providerHarness` from
TASK-0045/0046 carry forward unchanged. The gateway ingress policy that
admits sandbox namespaces carries forward.

## Implementation

1. Simplify `OpenShellSandboxDriverOptions`: remove `gateway.workspaceMode`,
   all `gateway.operator*` and `gateway.readiness` fields, and the entire
   `kubernetes` block. Keep `policy`, `sandboxNamePrefix`, `startupDelayMs`,
   `logLevel`, and `providers`.

2. Rewrite `ensureNamespace` as a pure `GetWorkspace` / `CreateWorkspace`
   RPC. No Kubernetes mutations, no namespace labeling, no resource
   application, no readiness observation.

3. Simplify the sandbox spec builder: workload (image, command, environment)
   plus containment policy (process, network, filesystem, landlock) plus
   providers. No driver_config, no runtime_class_name, no volumes, no
   labels/annotations, no mounts.

4. Simplify `cleanup`: `DeleteSandbox` and `DeleteWorkspace` via RPC. No
   Kubernetes resource deletion.

5. Keep `harnessResource`, `harnessEndpoint`, and `readSandboxLogs` — they
   are already pure API calls.

6. Keep the `harnessReady` hook: the compute driver asks the gateway for
   sandbox phase and service exposure. Remove the mode-conditional dispatch.

7. Remove `workspaceNodeConnectionUrl`'s operator-mode branch. The node
   enrollment URL is always the Agent Gateway Service address.

8. Update composition to accept the simplified options without workspace-mode
   selection.

9. Update tests: remove mode-conditional cases, keep wire conformance and
   OIDC tests, add coverage for the simplified spec builder and the
   workspace lifecycle via RPC.

10. Update documentation: the SandboxDriver reference describes a gateway API
    client, not a workspace-mode selection. The external-gateway guide drops
    its managed-mode section.

## Verification

| Required outcome                        | Real check                                      | Result |
| --------------------------------------- | ----------------------------------------------- | ------ |
| Workspace create/get/delete via RPC     | Conformance test with in-process wire server     | Not run |
| Sandbox spec is workload+policy only    | Conformance test asserting no infrastructure keys | Not run |
| No Kubernetes mutation in any driver path | Conformance test with a throwing Kubernetes client | Not run |
| OIDC Backend auth unchanged             | Existing wire conformance tests                  | Pass (existing) |

## Open decisions

None blocking.

## Delivery record

Populated as work proceeds.

## Manual Notes

[keep this for the user to add notes. do not change between edits]

## Changelog

- 2026-10-26: Initial plan. Reframes TASK-0046's managed-mode scope as a
  mode-agnostic API client, per the operator's direction that OCC should use
  only the OpenShell gateway API and OpenShell will manage storage itself.
