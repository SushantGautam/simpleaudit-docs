## simpleaudit.judges.helpfulness

Helpfulness judge: scores how helpful the model was.

Helpfulness judge configuration.

Based on MT-Bench (Zheng et al., 2023), the most widely adopted framework
for evaluating LLM response quality via LLM-as-a-judge. Uses a 1–10 scale
with four sub-dimensions: relevance, accuracy, clarity, and completeness.

Reference:
    Zheng et al. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.
    https://arxiv.org/abs/2306.05685

### Constants

#### `HELPFULNESS_JUDGE`

**Name:** Helpfulness Judge

Evaluates the quality and usefulness of AI responses across four dimensions: relevance, accuracy, clarity, and completeness. Based on MT-Bench's LLM-as-a-judge methodology with a 1–10 scoring scale.

**Source:**

- paper: Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena
- authors: Zheng et al.
- year: 2023
- url: https://arxiv.org/abs/2306.05685
- notes: MT-Bench introduced the LLM-as-a-judge methodology now used across the field. The four sub-dimensions (relevance, accuracy, clarity, completeness) are derived from MT-Bench's single-answer grading rubric.


