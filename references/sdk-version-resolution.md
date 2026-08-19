# Resolving the AWS SDK Model Version

Shared reference for locating and reading the correct AWS API model when planning ACK resource or field changes. Both the resource planner (`roles/planner.md`) and the field planner (`roles/field-planner.md`) rely on this discipline.

**CRITICAL: Do NOT use aws-sdk-go (v1). It is deprecated and its models are outdated. Always use aws-sdk-go-v2.** **Never** look in `github.com/aws/aws-sdk-go/` (no `-v2` suffix) — that is the deprecated v1 SDK.

**Resolve the version FIRST, then find that exact version. Do not read whichever SDK copy happens to be on disk.** A local `aws-sdk-go-v2` clone in the workspace is almost always pinned to some *unrelated* commit — it may be an older release that is missing the resource or field entirely, or a newer one with shapes code-gen will not use. Reading it is the single most common way this work goes wrong: you burn iterations hunting for an operation or field that "doesn't exist" when it simply isn't in the version you happened to open.

## Step 1: Determine which version and source code-gen uses

This is authoritative and comes from the controller itself, not from any SDK checkout. Read `apis/<version>/ack-generate-metadata.yaml`. Code generation fetches the Smithy model from one of two mutually exclusive sources, and **the service-specific version takes precedence:**

- `aws_service_sdk_version` (→ `--aws-service-sdk-version`): the **per-service** SDK tag, e.g. `service/<svc>/v1.29.0`. If this key is set (not `null`/empty), code-gen uses **this** model and ignores `aws_sdk_go_version` for model fetching.
- `aws_sdk_go_version` (→ `--aws-sdk-go-version`): the **core** `aws-sdk-go-v2` release tag, e.g. `v1.41.5`. Used when `aws_service_sdk_version` is unset. Note the core tag still implies a concrete service module version — cross-check the controller's `go.mod` `service/<svc>` line for the exact service module tag (e.g. `service/bedrockagentcorecontrol v1.29.0`) and read the model at that version.

## Step 2: Read the model at that resolved version

Locate it in this order:

1. The **Go module cache** at the resolved version: `$(go env GOMODCACHE)/github.com/aws/aws-sdk-go-v2/service/<service>@<version>/` (the version from `go.mod`). This is what the controller compiles against. If the module is not already in the cache (a clean build environment starts empty), populate it deterministically from the controller repo — do **not** go hunting for an on-disk copy:
   ```bash
   cd <controller-dir>
   svc_ver=$(go list -m -f '{{.Version}}' github.com/aws/aws-sdk-go-v2/service/<service>)
   go mod download github.com/aws/aws-sdk-go-v2/service/<service>
   ls "$(go env GOMODCACHE)/github.com/aws/aws-sdk-go-v2/service/<service>@${svc_ver}/"
   ```
   `go list -m` reads the version straight from `go.mod`, and `go mod download` fetches exactly that tag — so this works in an empty cache and always agrees with what the controller compiles against.
2. A local `aws-sdk-go-v2` clone **only if** you have checked out the exact resolved tag — otherwise do not use it.
3. AWS documentation via web (as a last resort to cross-check shapes).

## Why this matters

These tags can point at **different model contents** — AWS adds/removes shapes, operations, fields, and union members across versions. A resource or field that exists at `service/<svc>@v1.29.0` may be entirely absent from an older core release or an old local clone. If an operation (`Create<R>`, `Get<R>`, etc.) or a field appears to be "missing," suspect you are reading the wrong version **before** concluding the API lacks it.

Before relying on a field/operation/union member:
1. Confirm you resolved the version from `ack-generate-metadata.yaml` (+ `go.mod` for the concrete service tag), not from an arbitrary on-disk clone.
2. Confirm the field/shape/operation exists in **that** model (the one code-gen will fetch).
3. If the resolved model source and the `go.mod` service module disagree on a shape, note it — the controller's pinned versions have drifted, which a human should reconcile (often by setting/aligning `aws_service_sdk_version`).
