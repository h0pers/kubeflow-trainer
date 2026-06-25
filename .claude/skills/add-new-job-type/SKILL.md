# Add a New Distributed Job Type

Use this skill when adding support for a new distributed training framework (like JAX, PyTorch, TensorFlow).
JAXJob is the most recently added job type — use it as the primary reference.

## Files to Create

### 1. CRD Type Definition — `pkg/apis/kubeflow.org/v1/<framework>_types.go`

Define three structs and constants. Follow the marker pattern exactly:

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +resource:path=newjobs
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
//+kubebuilder:printcolumn:name="State",type=string,JSONPath=`.status.conditions[-1:].type`
//+kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type NewJob struct { ... }

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +resource:path=newjobs
//+kubebuilder:object:root=true
type NewJobList struct { ... }
```

Required constants (see `jax_types.go` for naming):

| Constant | Example Value |
|----------|---------------|
| `NewJobDefaultPortName` | `"newjob-port"` |
| `NewJobDefaultContainerName` | `"newjob"` |
| `NewJobDefaultPort` | `12345` |
| `NewJobDefaultRestartPolicy` | `RestartPolicyNever` |
| `NewJobKind` | `"NewJob"` |
| `NewJobPlural` | `"newjobs"` |
| `NewJobSingular` | `"newjob"` |
| `NewJobFrameworkName` | `"newjob"` |
| `NewJobReplicaTypeWorker` | `ReplicaType("Worker")` |

The `init()` function must register both the types and the defaulting functions:

```go
func init() {
    SchemeBuilder.Register(&NewJob{}, &NewJobList{})
    SchemeBuilder.SchemeBuilder.Register(addNewJobDefaultingFuncs)
}
```

### 2. Defaults — `pkg/apis/kubeflow.org/v1/<framework>_defaults.go`

Implement `SetDefaults_NewJob()` using helpers from `defaulting_utils.go`:
- `setDefaultPort()`, `setDefaultRestartPolicy()`, `setDefaultReplicas()`, `setTypeNameToCamelCase()`

Reference: `jax_defaults.go`

### 3. Controller — `pkg/controller.v1/<framework>/`

Create at minimum:

- **`<framework>_controller.go`**: Reconciler struct embedding `common.JobController`, implementing all methods of `common.ControllerInterface` (defined in `pkg/common/interface.go`). Key methods:
  - `NewReconciler()` — factory function
  - `Reconcile()` — main reconciliation entry point
  - `SetupWithManager()` — register watches for the job CRD, owned Pods, Services, and PodGroups
  - `SetClusterSpec()` — inject framework-specific env vars into pod templates
  - `IsMasterRole()` — identify the master replica
  - `UpdateJobStatus()` — update conditions and replica statuses

  Add RBAC markers:
  ```go
  //+kubebuilder:rbac:groups=kubeflow.org,resources=newjobs,verbs=get;list;watch;create;update;patch;delete
  //+kubebuilder:rbac:groups=kubeflow.org,resources=newjobs/status,verbs=get;update;patch
  //+kubebuilder:rbac:groups=kubeflow.org,resources=newjobs/finalizers,verbs=update
  ```

- **`envvar.go`**: Framework-specific environment variable setup (coordinator address, rank, world size, etc.)

- **Tests**: `<framework>_controller_test.go`, `<framework>_controller_suite_test.go`, `envvar_test.go`

### 4. Webhook — `pkg/webhooks/<framework>/<framework>_webhook.go`

Create a validating webhook with the marker:
```go
//+kubebuilder:webhook:path=/validate-kubeflow-org-v1-newjob,mutating=false,failurePolicy=fail,sideEffects=None,groups=kubeflow.org,resources=newjobs,verbs=create;update,versions=v1,name=validator.newjob.training-operator.kubeflow.org,admissionReviewVersions=v1
```

Implement `ValidateCreate`, `ValidateUpdate`, `ValidateDelete` on a `Webhook` struct. Validations should check:
- DNS-compliant job name
- RunPolicy fields (via `util.ValidateRunPolicy`)
- Replica specs: correct types, container images present, default container name exists

Reference: `pkg/webhooks/jax/jaxjob_webhook.go`

### 5. Examples — `examples/<framework>/`

Add at least one example YAML showing a minimal job spec.

## Files to Update

### 6. Controller Registration — `pkg/controller.v1/register_controller.go`

Add import and map entry in `SupportedSchemeReconciler`:
```go
import newjobcontroller "github.com/kubeflow/training-operator/pkg/controller.v1/newjob"

// In SupportedSchemeReconciler map:
kubeflowv1.NewJobKind: func(mgr manager.Manager, ...) error {
    return newjobcontroller.NewReconciler(mgr, gangSchedulingSetupFunc).SetupWithManager(mgr, controllerThreads)
},
```

### 7. Webhook Registration — `pkg/webhooks/webhooks.go`

Add import and map entry in `SupportedSchemeWebhook`:
```go
import "github.com/kubeflow/training-operator/pkg/webhooks/newjob"

// In SupportedSchemeWebhook map:
trainingoperator.NewJobKind: newjob.SetupWebhook,
```

### 8. Main Entrypoint — `cmd/training-operator.v1/main.go`

Add the new job kind to `enabledSchemes` and scheme registration.

## Code Generation (mandatory)

After all hand-written files are in place:

```bash
make generate    # Generates deepcopy, defaults, OpenAPI, clients, CRD YAMLs, Python SDK
```

This produces:
- `pkg/apis/kubeflow.org/v1/zz_generated.deepcopy.go` (updated)
- `pkg/apis/kubeflow.org/v1/zz_generated.defaults.go` (updated)
- `pkg/apis/kubeflow.org/v1/zz_generated.openapi.go` (updated)
- `manifests/base/crds/kubeflow.org_newjobs.yaml` (new)
- `manifests/base/rbac/role.yaml` (updated from RBAC markers)
- `pkg/client/` — clientset, informers, listers, apply configurations (updated)
- `sdk/python/kubeflow/training/models/kubeflow_org_v1_new_job*.py` (new)

Never hand-edit generated files.

## Python SDK Updates

After `make generate`, update the hand-written SDK files:

- `sdk/python/kubeflow/training/constants/constants.py` — add entry to `JOB_PARAMETERS` dict and `REPLICA_TYPES`
- `sdk/python/kubeflow/training/constants/constants.py` — add to `JOB_MODELS_TYPE` union

## Verification

```bash
make testall                    # Full suite: generate + fmt + vet + lint + test
go test ./pkg/controller.v1/<framework>/...   # Controller unit tests
```
