# Adding a Single Field to an Existing Resource

Context for adding one field to an ACK resource that already exists. This is a narrower task than adding a whole resource: the CRD, its CRUD operations, and its tests are already in place — you are extending them with a single field (in the Spec or the Status).

**Scope discipline:** add only the one field. Do NOT restructure the resource or its existing configuration, and do NOT create a new e2e test file — extend the resource's existing test.

## generator.yaml (single field)

Add only the field's configuration, under the existing resource's block. Most fields need no configuration at all — they auto-map from the SDK model. Configure only the non-default behavior the field requires. Common field-level options (see the [generator.yaml reference](generator-yaml-reference.md) and the Configuration Decision Table in [new-resource-checklist.md](new-resource-checklist.md)):

- **Un-ignore**: remove the field from `ignore.field_paths` if it was previously suppressed
- **Renames**: cover **every** operation the field appears in (a missing rename is the most common cause of `could not find field with path` build errors)
- `is_immutable`, `late_initialize` (+ `skip_incomplete_check: {}`), `from.operation`/`from.path`, `is_read_only`, `print.name`, `references` (same-service: omit `service_name`; cross-service: include it), or a custom field `type:` — each only when the field actually calls for it

**Custom hooks are a last resort.** Before adding a hook for a field, confirm no declarative `generator.yaml` option achieves the same behavior.

## SDK version (the #1 add-field failure)

The most common way a field addition fails is that the field is simply absent at the controller's pinned SDK model version — code-gen can't map a field the model doesn't contain. Before anything else, confirm the field exists at the version code-gen will actually use, resolving/reading it per [SDK Version Resolution](sdk-version-resolution.md) (from `apis/<version>/ack-generate-metadata.yaml` + the controller `go.mod`, not an arbitrary on-disk SDK clone).

If the field is only present in a newer SDK, bump the version in:

- `apis/<version>/ack-generate-metadata.yaml` — `aws_service_sdk_version` (takes precedence) and/or `aws_sdk_go_version`
- the `Makefile` `AWS_SDK_GO_VERSION`, if the version is pinned there
- the controller `go.mod` / `go.sum` `service/<service>` module tag

## Verify the field appears correctly

After building, confirm the generated output:

```bash
git diff apis/<version>/                      # field present, correct Spec vs Status placement
git diff config/crd/                          # CRD includes the field
git diff pkg/resource/<resource>/delta.go     # comparison includes the field (for mutable fields)
```

Spec vs Status placement follows the field's role: user-settable → Spec, AWS-assigned → Status. A mutable field must appear in `delta.go` so changes are detected on reconcile.

## Tests — extend, do not create

Extend the resource's **existing** `test/e2e/tests/test_<resource>.py` and `test/e2e/resources/<resource>.yaml` (see [testing.md](../skills/ack-dev/references/testing.md)):

- Set the new field on create and assert it via **both** the CR and the AWS API
- If the field is **mutable**, patch it, wait for the Synced condition, and re-verify via both CR and AWS API
- If the field is too new for boto3 to see in assertions (`KeyError`), bump the test-infra pin in `test/e2e/requirements.txt`:
  `acktest @ git+https://github.com/aws-controllers-k8s/test-infra.git@<latest-commit-hash>`

## Getting it right — checklist

- [ ] **SDK version sufficiency (#1 gotcha)**: the field exists at the resolved model version, or the version is bumped to one that first contains it
- [ ] **Renames cover ALL operations** the field appears in
- [ ] **Spec vs Status placement** correct (user-settable → Spec, AWS-assigned → Status)
- [ ] **delta.go comparison present** for a mutable field
- [ ] **Tests extend the existing file** (no duplicate test file); cover create + update-if-mutable with dual CR/AWS verification; test-infra pin bumped if the field isn't visible to boto3
- [ ] **Scoped to the field**: no unrelated changes to other resources or fields
- [ ] **Hooks are a last resort**: any hook is justified against declarative `generator.yaml` config
