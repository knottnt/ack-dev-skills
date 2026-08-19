# Field Plan Output Schema

The Planner (in field mode) must produce a document with exactly these sections. Every section is required. Use "N/A" for genuinely inapplicable items — never leave blanks.

This schema is for adding a **single field to an existing resource**. It is the field-mode counterpart to `plan-output.md` (which covers whole new resources).

---

## Field Summary

- **Service:** `<aws-service-name>` (e.g., `backup`, `ecr`, `s3`)
- **Resource:** `<ResourceName>` (must already exist in the controller — e.g., `BackupVault`, `Repository`)
- **Field:** `<OriginalFieldName>` → `<RenamedField>` (if renamed)
- **Type:** string | int | bool | list | map | nested structure (give the SDK shape name for structures)
- **Required:** Yes/No (per the API model, for the operation the field is set on)

## Source Operations

Which operations the field appears in. This determines Spec vs Status placement, renames, and whether `from.operation` is needed.

| Operation | AWS API Name | Field present? | Input/Output | Notes |
|-----------|-------------|----------------|--------------|-------|
| Create    |             | Yes/No         | Input/Output |       |
| Read      |             | Yes/No         | Output       | Describe vs Get |
| Update    |             | Yes/No         | Input        |       |
| List      |             | Yes/No         | Output       |       |

## SDK Version

The single most common add-field failure is the field not existing at the SDK model version code-gen actually uses. Resolve the version from `apis/<version>/ack-generate-metadata.yaml` (+ `go.mod` for the concrete service module tag) — NOT from an arbitrary on-disk SDK clone.

- **Resolved model source:** `aws_service_sdk_version` or `aws_sdk_go_version` value in use
- **Concrete service module tag:** from `go.mod` `service/<svc>` line
- **Field present at that version?** Yes/No
- **Required bump (if No):** target `aws_service_sdk_version` / `aws_sdk_go_version` + `go.mod` service module tag that first contains the field

## Classification

- **Spec vs Status:** Spec (user-settable — appears in an Input shape) or Status (AWS-assigned — output only)
- **Mutable vs Immutable:** Can it be changed after creation? (A field being "required" in Update does NOT make it mutable — verify against docs)
- **Currently ignored?** Is the field presently listed in `ignore.field_paths` (or otherwise suppressed) and needs to be un-ignored?

## Configuration

The generator.yaml options to apply for this field. List only non-default options.

| Option | Value | Rationale |
|--------|-------|-----------|
| `from.operation` / `from.path` | | Field only in Describe/Get output |
| `is_read_only` | | Output-only field |
| `late_initialize` (+ `skip_incomplete_check: {}`) | | Server-defaulted field |
| `print.name` | | Show in `kubectl get` |
| `is_immutable` | | Cannot change after creation |
| custom field `type:` / wrapper handling | | Field outside a wrapper, type override, etc. |

## Renames

| Original Field | Renamed To | Operations Requiring Rename | Input/Output |
|---------------|-----------|---------------------------|--------------|

List every operation where the renamed field appears. Missing a rename in any operation causes build failures (`could not find field with path`).

## Cross-Resource References

| Referenced Resource | Referenced Service | Field Path | Same-Service? |
|--------------------|--------------------|------------|--------------|

Same-service references omit `service_name` in generator.yaml. Cross-service references must include it. Use "N/A" if the field does not reference another resource.

## Custom Hooks Required

| Hook Point | Purpose | Key Logic | Why generator.yaml is insufficient |
|-----------|---------|-----------|-----------------------------------|

Custom hooks are a LAST RESORT. Only list a hook if no declarative generator.yaml option achieves the same result, and cite the specific limitation. Use "N/A" if none are needed.

## Test Plan

- **Existing e2e test to extend:** path (e.g., `test/e2e/tests/test_<resource>.py`)
- **Coverage to add:** set the field on create; if mutable, update it and re-verify; assert via both the CR and the AWS API
- **test-infra / boto3 pin bump needed?** Yes/No — if the field is new enough that boto3 assertions can't see it, bump `acktest` in `test/e2e/requirements.txt`

## Implementation Notes

Any non-standard patterns, gotchas, or special considerations that don't fit above (e.g., field lives outside a request wrapper, eventual-consistency concerns, special serialization).
