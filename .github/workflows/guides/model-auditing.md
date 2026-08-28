## Model Auditing

Core auditing logic resides `simpleaudit/auditor.py`. Module handles model loading, inference execution, audit result generation.

### Overview

`ModelAuditor` class centralizes audit workflow. Loads model, processes input, compares output against expected values. Supports multiple model types: transformers, sklearn, custom callable.

### `ModelAuditor` Class

Primary interface. Located `simpleaudit/auditor.py`.

#### Constructor

```python
class ModelAuditor:
    def __init__(
        self,
        model: Union[str, Path, object],
        model_type: str = "auto",
        device: str = "auto",
        batch_size: int = 32,
        max_length: int = 512
    ):
```

**Parameters:**

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `model` | `Union[str, Path, object]` | Required | Model path, HuggingFace ID, or loaded model instance. |
| `model_type` | `str` | `"auto"` | Model category: `"transformers"`, `"sklearn"`, `"custom"`, `"auto"`. Auto-detects from `model` type. |
| `device` | `str` | `"auto"` | Compute device: `"cpu"`, `"cuda"`, `"mps"`, `"auto"`. Auto-detects available hardware. |
| `batch_size` | `int` | `32` | Inference batch size. Higher values improve throughput, increase memory usage. |
| `max_length` | `int` | `512` | Max sequence length for tokenization. Truncates longer inputs. |

#### `load()` Method

Loads model into memory. Idempotent. Safe to call multiple times.

```python
def load(self) -> None:
```

**Behavior:**
- If `model` string: loads via `transformers.AutoModelForSequenceClassification` or `sklearn.joblib.load`.
- If `model` object: validates instance, skips loading.
- Sets `self._model` internal reference.
- Raises `ModelLoadError` on failure.

#### `audit()` Method

Executes audit on input data.

```python
def audit(
    self,
    inputs: Union[str, List[str], np.ndarray, pd.DataFrame],
    expected: Optional[Union[str, List[str], np.ndarray]] = None,
    threshold: float = 0.5
) -> AuditResult:
```

**Parameters:**

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `inputs` | `Union[str, List[str], np.ndarray, pd.DataFrame]` | Required | Data to process. String, list strings, numpy array, or DataFrame. |
| `expected` | `Optional[Union[str, List[str], np.ndarray]]` | `None` | Expected outputs for comparison. If `None`, returns raw predictions only. |
| `threshold` | `float` | `0.5` | Confidence threshold for classification. Values below threshold marked uncertain. |

**Returns:** `AuditResult` object.

#### `AuditResult` Dataclass

Container for audit outcomes.

```python
@dataclass
class AuditResult:
    predictions: List[Any]
    probabilities: Optional[List[float]]
    accuracy: Optional[float]
    mismatches: List[Mismatch]
    metadata: Dict[str, Any]
```

**Fields:**

| Field | Type | Description |
| :--- | :--- | :--- |
| `predictions` | `List[Any]` | Model outputs for each input. |
| `probabilities` | `Optional[List[float]]` | Confidence scores. `None` if model does not support probability output. |
| `accuracy` | `Optional[float]` | Match rate if `expected` provided. `None` otherwise. |
| `mismatches` | `List[Mismatch]` | Details on discrepancies between prediction and expected. |
| `metadata` | `Dict[str, Any]` | Timing, device info, batch counts. |

### `Mismatch` Dataclass

Details single discrepancy.

```python
@dataclass
class Mismatch:
    index: int
    input: Any
    predicted: Any
    expected: Any
    probability: Optional[float]
```

### Usage Examples

#### Basic Text Classification Audit

```python
from simpleaudit import ModelAuditor

# Initialize auditor with HuggingFace model
auditor = ModelAuditor(
    model="distilbert-base-uncased-finetuned-sst-2-english",
    model_type="transformers",
    device="cuda"
)

# Load model
auditor.load()

# Run audit
inputs = [
    "I loved the movie, great acting!",
    "Terrible film, waste of time."
]
expected = ["POSITIVE", "NEGATIVE"]

result = auditor.audit(inputs, expected=expected)

print(f"Accuracy: {result.accuracy:.2f}")
print(f"Mismatches: {len(result.mismatches)}")

for mismatch in result.mismatches:
    print(f"Index {mismatch.index}: Predicted '{mismatch.predicted}', Expected '{mismatch.expected}'")
```

#### NumPy Array Input (Regression)

```python
import numpy as np
from simpleaudit import ModelAuditor

# Custom sklearn model
import joblib
model_path = "models/regressor.pkl"
auditor = ModelAuditor(model=model_path, model_type="sklearn")
auditor.load()

# Prepare data
X = np.array([[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]])
y_expected = np.array([5.0, 10.0, 15.0])

result = auditor.audit(X, expected=y_expected, threshold=0.1)

# Check accuracy
if result.accuracy < 0.95:
    print("Warning: Accuracy below threshold")
```

#### Custom Callable Model

```python
from simpleaudit import ModelAuditor

def my_custom_model(inputs):
    # Custom logic
    return [x * 2 for x in inputs]

auditor = ModelAuditor(model=my_custom_model, model_type="custom")
auditor.load()

result = auditor.audit([1, 2, 3], expected=[2, 4, 6])
print(result.predictions)  # [2, 4, 6]
```

### Configuration Notes

- **Device Selection**: `"auto"` checks CUDA availability, then MPS, then CPU. Explicit `"cpu"` forces CPU execution for debugging.
- **Batch Processing**: Large inputs process in chunks of `batch_size`. Memory usage scales linearly with batch size.
- **Tokenization**: Transformers use default tokenizer from model ID. Custom tokenizers require pre-loading model object.
- **Error Handling**: `ModelLoadError` raised on invalid paths or incompatible model types. `InferenceError` raised during forward pass failures.

### File Structure

- `simpleaudit/auditor.py`: `ModelAuditor`, `AuditResult`, `Mismatch` classes.
- `simpleaudit/loaders.py`: Model loading utilities.
- `simpleaudit/inference.py`: Batch processing logic.

### Best Practices

1. **Pre-load Models**: Call `load()` once, reuse auditor for multiple `audit()` calls. Avoids repeated model initialization.
2. **Batch Size Tuning**: Start with `batch_size=32`. Increase if GPU memory allows. Decrease if `CUDA out of memory` errors occur.
3. **Threshold Adjustment**: Lower `threshold` for high-confidence requirements. Higher `threshold` reduces false positives in mismatch detection.
4. **Logging**: Enable logging via `logging.getLogger("simpleaudit").setLevel(logging.DEBUG)` for detailed inference traces.

### Troubleshooting

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `ModelLoadError: File not found` | Invalid `model` path | Verify file exists, check permissions. |
| `InferenceError: CUDA error` | GPU memory exhaustion | Reduce `batch_size`, use `device="cpu"`. |
| `Accuracy: None` | `expected` parameter missing | Provide `expected` array to calculate accuracy. |
| Slow inference | CPU-only execution | Set `device="cuda"` if GPU available. |

### API Reference Summary

- `ModelAuditor.__init__()`: Initialize auditor.
- `ModelAuditor.load()`: Load model into memory.
- `ModelAuditor.audit()`: Execute inference and comparison.
- `AuditResult`: Dataclass holding predictions, accuracy, mismatches.
- `Mismatch`: Dataclass holding single discrepancy details.

### See Also

*   [CLI Usage](cli-usage.md)
*   [Core Architecture](core-architecture.md)
*   [Custom Scenario Development](custom-scenario-development.md)
*   [Judges System](judges-system.md)
