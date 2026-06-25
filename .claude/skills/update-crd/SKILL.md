# Update an Existing CRD

Use this skill when modifying CRD type definitions — adding fields, changing validation, updating defaults, or adjusting printer columns.

## Identify What to Change

CRD types live in `pkg/apis/kubeflow.org/v1/`. Each job type has:
- `<framework>_types.go` — struct definitions and kubebuilder markers
- `<framework>_defaults.go` — default value logic
- `common_types.go` — shared types (`JobStatus`, `ReplicaSpec`, `RunPolicy`, etc.)

## Step 1: Modify Type Structs

Edit the relevant `_types.go` file. Key patterns:

**Adding a field to a job spec:**
```go
type PyTorchJobSpec struct {
    RunPolicy   RunPolicy                              `json:"runPolicy"`
    PyTorchReplicaSpecs map[ReplicaType]*ReplicaSpec    `json:"pytorchReplicaSpecs"`
    NewField    *string                                 `json:"newField,omitempty"`  // new
}
```

**Adding validation via kubebuilder markers:**
```go
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Maximum=100
NewField *int32 `json:"newField,omitempty"`
```

**Adding a printer column:**
```go
//+kubebuilder:printcolumn:name="NewCol",type=string,JSONPath=`.spec.newField`
```

**Modifying shared types** (affects all job types):
Edit `common_types.go` for changes to `RunPolicy`, `ReplicaSpec`, `JobStatus`, `SchedulingPolicy`, etc.

## Step 2: Update Defaults

If the new field needs a default value, edit `<framework>_defaults.go`:

```go
func SetDefaults_PyTorchJob(job *PyTorchJob) {
    // existing defaults...
    if job.Spec.NewField == nil {
        defaultVal := "default"
        job.Spec.NewField = &defaultVal
    }
}
```

Shared defaulting helpers are in `defaulting_utils.go`.

## Step 3: Update Webhook Validation

If the new field needs validation beyond kubebuilder markers, update the webhook in `pkg/webhooks/<framework>/`:

```go
func validateSpec(spec trainingoperator.PyTorchJobSpec) field.ErrorList {
    var allErrs field.ErrorList
    // existing validations...
    if spec.NewField != nil && *spec.NewField == "" {
        allErrs = append(allErrs, field.Invalid(
            field.NewPath("spec").Child("newField"),
            spec.NewField,
            "must not be empty when set",
        ))
    }
    return allErrs
}
```

For immutable fields (cannot change after creation), add to `ValidateUpdate`:
```go
func (w *Webhook) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
    oldJob := oldObj.(*trainingoperator.PyTorchJob)
    newJob := newObj.(*trainingoperator.PyTorchJob)
    allErrs := validatePyTorchJob(oldJob, newJob)
    // Check immutability
    if oldJob.Spec.NewField != newJob.Spec.NewField {
        allErrs = append(allErrs, field.Forbidden(...))
    }
    return nil, allErrs.ToAggregate()
}
```

Reference: `pkg/webhooks/tensorflow/tfjob_webhook.go` for RunPolicy immutability checks.

## Step 4: Run Code Generation (mandatory)

```bash
make generate
```

This regenerates all derived artifacts:
- `zz_generated.deepcopy.go` — DeepCopy methods for modified structs
- `zz_generated.defaults.go` — default registration wrappers
- `zz_generated.openapi.go` — OpenAPI schema
- `manifests/base/crds/kubeflow.org_<plural>.yaml` — CRD YAML with updated schema
- `pkg/client/` — clientset, informers, listers, apply configurations
- `sdk/python/kubeflow/training/models/` — Python model classes

Never hand-edit any of these files.

## Step 5: Update Tests

- Controller tests in `pkg/controller.v1/<framework>/` — test new field handling in reconciliation
- Webhook tests in `pkg/webhooks/<framework>/` — test new validation rules
- Defaults tests (if any) — verify default values are set correctly

## Step 6: Backwards Compatibility

- New fields should be pointers with `omitempty` JSON tags to remain optional
- Existing fields must not change JSON tag names or types
- If removing a field, consider deprecation first (add `// Deprecated:` comment)
- RunPolicy changes affect all job types — test across frameworks

## Verification

```bash
make testall                                          # Full suite
go test ./pkg/controller.v1/<framework>/...           # Controller tests
go test ./pkg/webhooks/<framework>/...                # Webhook tests (if present)
hack/verify-codegen.sh                                # Verify generated code is current
```
