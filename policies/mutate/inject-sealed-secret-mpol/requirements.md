# Policy Requirements: Inject CoCo Sealed Secret Configuration

## Purpose
Inject sealed secret configuration into pods that have the label 'coco: enabled' and annotation 'coco.io/sealed-secret-config' to enable containers to retrieve secrets from KBS (Key Broker Service).

## Match Criteria
- **Resources**: Pod
- **Namespaces**: All namespaces
- **Conditions**: 
  - Pod must have label `coco: enabled`
  - Pod must have annotation `coco.io/sealed-secret-config` with a non-empty value

## Policy Logic
Inject environment variables into all containers in matching pods:
- Retrieve configuration from a ConfigMap specified by the `coco.io/sealed-secret-config` annotation
- Add `COCO_SECRET_RESOURCE` environment variable with value from ConfigMap's `secret_resource` field
- Add `COCO_SECRET_TARGET_PATH` environment variable with value from ConfigMap's `target_path` field
- Apply to all containers in the pod

## Edge Cases
- ConfigMap referenced in annotation must exist in the same namespace as the pod
- If ConfigMap is missing required fields (`secret_resource` or `target_path`), the mutation should still proceed but may result in empty values
- Existing environment variables with the same names should be preserved/not overwritten

## Expected Behavior
- **PASS**: Pods with `coco: enabled` label and valid `coco.io/sealed-secret-config` annotation get the environment variables injected
- **FAIL**: Pods without the required label or annotation are not modified

## Checklist
- [ ] Step 1: Detect source type and load sub-skill
- [ ] Step 2: Analyze source and create requirements.md
- [ ] Step 3: Generate policy with `generate_policy` tool
- [ ] Step 4: Generate chainsaw tests
- [ ] Step 5: Execute tests
- [ ] Step 6: Iterate and fix issues if tests fail
- [ ] Step 7: Re-check requirements to ensure accuracy and completion
- [ ] Step 8: Update checklist in requirements.md for each step completed
- [ ] Step 9: Report success with artifacts