## simpleaudit.judges.abstention

Abstention judge: detects when the model should have refused/abstained.

Abstention judge configuration.

Based on AbstentionBench (Kirichenko et al., 2025, Meta Research), which evaluates
LLM abstention across 20+ diverse datasets — medical, search, moral dilemmas,
math — using the core distinction between answerable and unanswerable questions.

Reference:
    Kirichenko et al. (2025). AbstentionBench: Reasoning LLMs Fail on Unanswerable Questions.
    https://arxiv.org/abs/2506.09038

### Constants

#### `ABSTENTION_JUDGE`

**Name:** Abstention Judge

Evaluates whether an AI model correctly abstains when faced with an unanswerable question. Follows the AbstentionBench taxonomy: Answer Unknown, False Premise, Stale, Subjective, Underspecified Context, Underspecified Intent.

**Source:**

- paper: AbstentionBench: Reasoning LLMs Fail on Unanswerable Questions
- authors: Kirichenko, Ibrahim, Chaudhuri, Bell (Meta Research)
- year: 2025
- url: https://arxiv.org/abs/2506.09038
- notes: Categories follow the paper's taxonomy verbatim. The paper evaluates abstention across 20+ datasets with a binary LLM-as-judge validated at ~88% agreement with human annotations; the 1–10 scoring here extends that with quality granularity while preserving the categorical structure.


