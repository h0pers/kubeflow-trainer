# Architecture

Kubernetes operator for distributed machine learning training jobs. Manages six framework-specific CRDs (PyTorchJob, TFJob, XGBoostJob, MPIJob, PaddleJob, JAXJob) using the `kubeflow.org/v1` API, providing pod/service lifecycle management, gang scheduling, and elastic training support.

## Controller architecture

```
cmd/training-operator.v1/main.go
  └─ controller-runtime Manager
       ├─ Per-job reconciler (one per enabled CRD)
       │    └─ Embeds common.JobController for shared logic
       ├─ Per-job validating webhook
       ├─ Cert rotation (pkg/cert/)
       └─ Metrics, health, leader election
```

### Reconciler pattern

Each job type has a reconciler in `pkg/controller.v1/<framework>/` that embeds `common.JobController` and implements `common.ControllerInterface` (defined in `pkg/common/interface.go`). The common controller handles pod/service lifecycle, expectations, status updates, and gang scheduling. Job-specific reconcilers provide:

- `SetClusterSpec()` — inject framework environment variables (coordinator address, rank, world size)
- `IsMasterRole()` — identify the master replica type
- `UpdateJobStatus()` — map replica states to job conditions

### Registration

Controllers and webhooks are registered via maps in two files:

| File | Map | Purpose |
|------|-----|---------|
| `pkg/controller.v1/register_controller.go` | `SupportedSchemeReconciler` | Maps job kind to reconciler setup function |
| `pkg/webhooks/webhooks.go` | `SupportedSchemeWebhook` | Maps job kind to webhook setup function |

`main.go` iterates enabled schemes and calls both setup functions.

### Gang scheduling

Pluggable via `GangSchedulingSetupFunc` — supports Volcano (`volcano.sh/v1beta1 PodGroup`) and scheduler-plugins (`scheduling.sigs.k8s.io/v1alpha1 PodGroup`). Configured via `--gang-scheduler-name` flag. The common controller syncs PodGroups based on `RunPolicy.SchedulingPolicy`.

## CRD types

All CRDs live in `pkg/apis/kubeflow.org/v1/`. Each job type has:

| File | Content |
|------|---------|
| `<framework>_types.go` | Job, JobSpec, JobList structs with kubebuilder markers |
| `<framework>_defaults.go` | `SetDefaults_<Job>()` for default ports, replicas, restart policy |
| `common_types.go` | Shared types: `RunPolicy`, `ReplicaSpec`, `JobStatus`, `SchedulingPolicy` |

### Job types and replica roles

| CRD | Replica types | Framework-specific features |
|-----|---------------|---------------------------|
| PyTorchJob | Master, Worker | Elastic training, init containers, HPA |
| TFJob | PS, Chief, Worker, Evaluator | Parameter server architecture |
| MPIJob | Launcher, Worker | MPI-based distributed training |
| XGBoostJob | Master, Worker | Gradient boosting |
| PaddleJob | Master, Worker | PaddlePaddle framework |
| JAXJob | Worker | JAX SPMD training |

### Shared labels

All pods and services created by the operator carry these labels:
- `training.kubeflow.org/operator-name` — framework name
- `training.kubeflow.org/job-name` — job name
- `training.kubeflow.org/replica-type` — replica type (lowercase)
- `training.kubeflow.org/replica-index` — replica index (0-based)

## Webhooks

Validating webhooks in `pkg/webhooks/<framework>/` check:
- DNS-compliant job names
- RunPolicy validity and immutability on update
- Replica spec requirements (container images, default container name, replica counts)

No mutating webhooks — defaults are applied via the scheme defaulting mechanism.

## Code generation

After modifying types in `pkg/apis/kubeflow.org/v1/`, run `make generate` to produce:

| Output | Generator |
|--------|-----------|
| `zz_generated.deepcopy.go` | controller-gen (from `+k8s:deepcopy-gen` markers) |
| `zz_generated.defaults.go` | k8s code-generator defaulter-gen |
| `zz_generated.openapi.go` | kube-openapi |
| `manifests/base/crds/*.yaml` | controller-gen (from kubebuilder markers) |
| `pkg/client/` | k8s code-generator (clientset, informers, listers, apply configs) |
| `sdk/python/kubeflow/training/models/` | OpenAPI Generator via `hack/python-sdk/` |

## Python SDK

Two layers in `sdk/python/kubeflow/training/`:

- **Generated** (`models/`) — OpenAPI model classes from Go types. Produced by `hack/python-sdk/gen-sdk.sh`.
- **Hand-written** — high-level `TrainingClient` class (`api/training_client.py`), job type registry (`constants/constants.py`), and utilities (`utils/utils.py`).

`TrainingClient` provides CRUD operations, status polling, pod/log retrieval, and a high-level `train()` method for LLM fine-tuning.

## Design proposals

The `docs/proposals/` directory contains design documents for significant features:

| Proposal | Topic |
|----------|-------|
| `2003-train-api/` | High-level Train/Fine-tune API for LLMs |
| `2145-jax-integration/` | JAX framework integration (JAXJob CRD) |
| `2170-kubeflow-training-v2/` | V2 API design using Kubernetes JobSet |

## Deployment

Kustomize manifests in `manifests/`:
- `base/` — CRDs, RBAC, webhook config, deployment
- `overlays/standalone/` — standalone deployment variant
- `rhoai/` — ODH/RHOAI-specific overlay with UBI base image
