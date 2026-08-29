## Advanced Evaluation Metrics

The `simpleaudit` library provides a suite of advanced evaluation techniques designed to move beyond basic pass/fail assertions. These metrics are essential for evaluating probabilistic AI models, where outputs may vary slightly between runs, or where the "correct" answer is subjective. This module focuses on **cross-judging**, **repeated result aggregation**, and **meta-evaluation** to provide robust, statistically significant insights into model performance.

### Overview

Standard unit tests assume deterministic behavior. However, Large Language Models (LLMs) and other probabilistic systems often exhibit non-determinism. `simpleaudit` addresses this by offering tools that:

1.  **Cross-Judge**: Use one model to evaluate the output of another, reducing bias from a single evaluator.
2.  **Aggregate Repeated Results**: Run the same test multiple times to calculate stability and consistency metrics.
3.  **Meta-Evaluate**: Assess the reliability of the evaluation process itself.

These features are primarily found in the `simpleaudit.metrics` and `simpleaudit.evaluator` modules.

### Cross-Judging

Cross-judging allows you to use a "Judge" model to evaluate the output of a "Target" model. This is particularly useful when ground-truth labels are unavailable or when evaluating qualitative aspects like tone, coherence, or safety.

#### The `CrossJudge` Class

The core component for this workflow is the `CrossJudge` class. It encapsulates the logic for prompting the judge model and parsing its response into a structured score.

```python
from simpleaudit.metrics import CrossJudge
from simpleaudit.models import OpenAIModel

# Initialize the judge model (e.g., a more capable or different model)
judge_model = OpenAIModel(model_name="gpt-4o", temperature=0.0)

# Define the cross-judge configuration
judge = CrossJudge(
    judge_model=judge_model,
    criteria={
        "accuracy": "Does the answer factually align with the provided context?",
        "clarity": "Is the response clear and concise?",
        "safety": "Does the response avoid harmful or biased content?"
    },
    scale_range=(1, 5),
    json_output=True
)
```

**Parameters:**

*   `judge_model`: An instance of a model compatible with the `simpleaudit.models` interface.
*   `criteria`: A dictionary mapping metric names to their descriptions. The judge model will score the target output based on these descriptions.
*   `scale_range`: A tuple `(min, max)` defining the scoring scale. Defaults to `(1, 5)`.
*   `json_output`: If `True`, the judge is instructed to return a JSON object, which `simpleaudit` parses automatically.

#### Usage Example

To use cross-judging within a test case, you pass the `CrossJudge` instance to the evaluation context.

```python
import simpleaudit as sa

@sa.test
def test_support_ticket_response():
    target_output = "We are sorry to hear about your issue. Please try restarting your device."
    
    # The judge evaluates the target_output against the defined criteria
    scores = judge.evaluate(
        input="My laptop is not turning on.",
        output=target_output,
        context="User reports hardware failure."
    )
    
    # Assert specific thresholds
    assert scores["accuracy"] >= 4, f"Accuracy score too low: {scores['accuracy']}"
    assert scores["safety"] == 5, "Safety score must be perfect"
```

### Repeated Results and Stability

Because LLMs are stochastic, a single run may not represent typical behavior. `simpleaudit` supports running tests multiple times and aggregating the results to calculate stability metrics.

#### The `RepeatedEvaluator`

The `RepeatedEvaluator` wraps a standard evaluator and executes the test `n` times. It then aggregates the results using statistical methods.

```python
from simpleaudit.evaluator import RepeatedEvaluator
from simpleaudit.metrics import StabilityMetric

# Create a base evaluator
base_evaluator = sa.Evaluator()

# Wrap it in a RepeatedEvaluator
repeated_evaluator = RepeatedEvaluator(
    base_evaluator=base_evaluator,
    num_runs=5,
    aggregation_strategy="mean"  # Options: 'mean', 'median', 'min', 'max'
)
```

**Parameters:**

*   `base_evaluator`: The underlying evaluator instance.
*   `num_runs`: The number of times to execute the test.
*   `aggregation_strategy`: How to combine the scores from multiple runs.
    *   `mean`: Average score.
    *   `median`: Middle value (robust to outliers).
    *   `min`: Worst-case performance.
    *   `max`: Best-case performance.

#### Stability Metrics

In addition to aggregated scores, `simpleaudit` calculates **stability metrics** to measure consistency.

*   **Variance**: Measures the spread of scores across runs.
*   **Pass Rate**: The percentage of runs that passed the assertion.

```python
# After running the repeated evaluator, access stability metrics
result = repeated_evaluator.run(test_case)

print(f"Mean Score: {result.aggregated_score}")
print(f"Pass Rate: {result.pass_rate:.2%}")
print(f"Variance: {result.variance:.4f}")

# Assert that the model is stable
assert result.pass_rate >= 0.8, "Model is too inconsistent"
assert result.variance < 0.5, "Scores vary too much between runs"
```

### Meta-Evaluation

Meta-evaluation involves evaluating the evaluator. This is critical for ensuring that your testing criteria are fair and that the judge model is reliable. `simpleaudit` provides tools to detect bias and inconsistency in the judging process.

#### Bias Detection

The `BiasDetector` class analyzes the distribution of scores given by the judge model across different inputs to detect potential biases.

```python
from simpleaudit.metrics import BiasDetector

# Collect a batch of evaluation results
results = [
    {"input": "Query A", "score": 5, "judge": "gpt-4o"},
    {"input": "Query B", "score": 2, "judge": "gpt-4o"},
    # ... more results
]

detector = BiasDetector(results=results)

# Check for length bias (longer responses getting higher scores)
length_bias = detector.check_length_bias()
print(f"Length Bias Correlation: {length_bias:.3f}")

# Check for self-preference bias (judge preferring its own model's outputs)
self_pref_bias = detector.check_self_preference()
print(f"Self-Preference Bias: {self_pref_bias:.3f}")
```

**Key Methods:**

*   `check_length_bias()`: Calculates the Pearson correlation between response length and score. A high positive correlation suggests the judge favors longer responses.
*   `check_self_preference()`: Compares scores given to outputs from the same model family versus different model families.

#### Judge Agreement

To ensure the judge model is reliable, you can measure **inter-rater agreement** by using multiple judge models or multiple runs of the same judge.

```python
from simpleaudit.metrics import AgreementMetric

# Compare scores from two different judges
judge_1_scores = [5, 4, 5, 3]
judge_2_scores = [5, 5, 4, 3]

agreement = AgreementMetric.calculate_cohen_kappa(judge_1_scores, judge_2_scores)
print(f"Cohen's Kappa: {agreement:.3f}")

# Kappa > 0.6 indicates substantial agreement
assert agreement > 0.6, "Judges disagree significantly"
```

### Configuration and Best Practices

When using advanced metrics, consider the following configuration options in your `simpleaudit.yaml` or Python configuration:

```yaml
evaluation:
  cross_judging:
    enabled: true
    judge_model: "gpt-4o"
    temperature: 0.0  # Keep low for consistency
  repeated_runs:
    count: 3
    aggregation: "median"
  meta_evaluation:
    bias_check: true
    min_agreement: 0.6
```

**Best Practices:**

1.  **Use Low Temperature for Judges**: Always set the judge model's temperature to `0.0` to ensure consistent scoring.
2.  **Define Clear Criteria**: Vague criteria in `CrossJudge` lead to noisy scores. Be specific in your `criteria` dictionary.
3.  **Monitor Bias Regularly**: Run `BiasDetector` periodically to ensure your judge model isn't developing systematic biases.
4.  **Balance Cost and Accuracy**: Cross-judging and repeated runs increase API costs. Use `num_runs=3` as a default for development, and increase to `5-10` for final validation.

### Conclusion

The advanced evaluation metrics in `simpleaudit` provide a robust framework for evaluating probabilistic AI systems. By leveraging cross-judging, repeated result aggregation, and meta-evaluation, developers can gain deeper insights into model performance, stability, and reliability. These tools are essential for building trustworthy AI applications where deterministic testing is insufficient.

### See Also

*   [Getting Started](getting-started.md)
