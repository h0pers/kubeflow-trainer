# Add a New Python SDK Method

Use this skill when adding a new method to the `TrainingClient` Python SDK.

## Architecture

The SDK has two layers:
- **Generated layer** (`sdk/python/kubeflow/training/models/`) — OpenAPI-generated model classes from Go types. Never hand-edit.
- **Hand-written layer** — the high-level client, constants, and utilities:
  - `sdk/python/kubeflow/training/api/training_client.py` — main `TrainingClient` class
  - `sdk/python/kubeflow/training/constants/constants.py` — job type registry and constants
  - `sdk/python/kubeflow/training/utils/utils.py` — helper functions

## Step 1: Add the Method to `TrainingClient`

File: `sdk/python/kubeflow/training/api/training_client.py`

Follow the existing method pattern:

```python
def new_method(
    self,
    name: str,
    namespace: Optional[str] = None,
    job_kind: Optional[str] = None,
    timeout: int = constants.DEFAULT_TIMEOUT,
) -> SomeReturnType:
    """Short description of what this method does.

    Args:
        name: Name of the training job.
        namespace: Namespace for the job. Defaults to client namespace.
        job_kind: Job kind (e.g., PyTorchJob). Defaults to client job_kind.
        timeout: Kubernetes API timeout in seconds.

    Returns:
        Description of return value.

    Raises:
        TimeoutError: On API timeout.
        RuntimeError: On API errors.
    """
    namespace = namespace or self.namespace
    job_kind = job_kind or self.job_kind

    if job_kind not in constants.JOB_PARAMETERS:
        raise ValueError(
            f"Job kind must be one of {list(constants.JOB_PARAMETERS.keys())}"
        )

    try:
        # Use self.custom_api for CRD operations
        response = self.custom_api.get_namespaced_custom_object(
            constants.GROUP,
            constants.VERSION,
            namespace,
            constants.JOB_PARAMETERS[job_kind]["plural"],
            name,
            _request_timeout=timeout,
        )
        # Deserialize if returning a model object
        return self.api_client.deserialize(
            response, constants.JOB_PARAMETERS[job_kind]["model"]
        )
    except multiprocessing.TimeoutError:
        raise TimeoutError(f"Timeout waiting for {job_kind}/{name}")
    except client.ApiException as e:
        raise RuntimeError(f"Failed: {e}")
```

Key conventions:
- `namespace` and `job_kind` default to `self.namespace` / `self.job_kind`
- Validate `job_kind` against `constants.JOB_PARAMETERS`
- Use `self.custom_api` (`CustomObjectsApi`) for training job CRDs
- Use `self.core_api` (`CoreV1Api`) for Pods, Services, ConfigMaps
- Wrap API calls with `try/except` for `ApiException` and `TimeoutError`
- Use `self.api_client.deserialize()` to convert responses to model objects

## Step 2: Add Constants (if needed)

File: `sdk/python/kubeflow/training/constants/constants.py`

If the method introduces new constants (labels, default values, condition types):

```python
NEW_CONSTANT = "some-value"
```

If adding a new job type, add to `JOB_PARAMETERS`:
```python
JOB_PARAMETERS = {
    ...
    "NewJob": {
        "model": "KubeflowOrgV1NewJob",
        "plural": "newjobs",
        "container": "newjob",
        "base_image": "docker.io/new-framework:latest",
    },
}
```

And update the `JOB_MODELS_TYPE` union type and `REPLICA_TYPES` dict.

## Step 3: Add Utility Functions (if needed)

File: `sdk/python/kubeflow/training/utils/utils.py`

For shared helpers (e.g., building pod templates, container specs).

## Step 4: Update Exports

File: `sdk/python/kubeflow/training/__init__.py`

If you added new public classes or functions, add them to `__init__.py` imports.
Note: most of this file is auto-generated — only the appended section at the bottom is hand-written.

## Step 5: Write Tests

File: `sdk/python/kubeflow/training/api/training_client_test.py`

Follow the existing test pattern using `unittest.mock`:

```python
def test_new_method(self):
    """Test the new_method functionality."""
    # Mock the Kubernetes API
    with patch.object(
        self.client.custom_api,
        "get_namespaced_custom_object",
        return_value=mock_response,
    ):
        result = self.client.new_method(
            name="test-job",
            namespace="default",
            job_kind="PyTorchJob",
        )
        self.assertEqual(result, expected_value)
```

Test cases to cover:
- Happy path with explicit parameters
- Default namespace/job_kind from client
- Invalid job_kind raises `ValueError`
- API timeout raises `TimeoutError`
- API error raises `RuntimeError`

## Verification

```bash
# Unit tests
pytest sdk/python/kubeflow/training/api/training_client_test.py -v

# Lint and format
pre-commit run black --files sdk/python/kubeflow/training/api/training_client.py
pre-commit run flake8 --files sdk/python/kubeflow/training/api/training_client.py
pre-commit run isort --files sdk/python/kubeflow/training/api/training_client.py
```
