## simpleaudit.judges.harm

Harm judge: scores harmfulness of model outputs.

Harm categorisation judge configuration.

Based on HELM Safety (Liang et al., 2022) and its six harm categories:
violence, fraud, discrimination, sexual content, harassment, and deception.
HELM Safety uses a dual-judge ensemble approach; this configuration adapts
its taxonomy for single-judge use in SimpleAudit.

Reference:
    Liang et al. (2022). Holistic Evaluation of Language Models (HELM).
    https://arxiv.org/abs/2211.09110

    HELM Safety v1.0 evaluation criteria:
    https://crfm.stanford.edu/helm/

### Constants

#### `HARM_JUDGE`

**Name:** Harm Categorisation Judge

Detects and categorises harmful AI outputs across six HELM Safety categories: violence, fraud, discrimination, sexual content, harassment, and deception. Returns whether harm was present, its category, and severity.

**Source:**

- paper: Holistic Evaluation of Language Models (HELM)
- authors: Liang et al.
- year: 2022
- url: https://arxiv.org/abs/2211.09110
- notes: HELM Safety defines six harm categories used here verbatim. The severity scale is adapted from HELM Safety v1.0, which uses a dual-judge ensemble (GPT-4o + Llama 3.1 405B) — this config adapts it for single-judge use.


