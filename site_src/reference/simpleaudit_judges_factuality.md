## simpleaudit.judges.factuality

Factuality judge: scores factual accuracy of model outputs.

Factuality judge configuration.

Based on G-Eval (Liu et al., 2023), which uses chain-of-thought prompting
to achieve better human alignment on NLG evaluation tasks. The factuality
dimension specifically targets hallucination, unsupported claims, and
fabricated details — the most common failure mode of current LLMs.

Reference:
    Liu et al. (2023). G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment.
    EMNLP 2023. https://arxiv.org/abs/2303.16634

### Constants

#### `FACTUALITY_JUDGE`

**Name:** Factuality Judge

Evaluates factual accuracy of AI responses — detecting hallucinations, unsupported claims, and fabricated details. Based on G-Eval's chain-of-thought evaluation methodology.

**Source:**

- paper: G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment
- authors: Liu et al.
- year: 2023
- url: https://arxiv.org/abs/2303.16634
- notes: G-Eval's key insight is using chain-of-thought reasoning before scoring, which significantly improves human alignment. The factuality dimension is the most directly applicable to hallucination detection in conversational AI.


