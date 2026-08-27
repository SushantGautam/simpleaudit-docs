## simpleaudit.judges.helsedir_sexhealth_no

Helsedir sex-health (NO) judge: domain-specific Norwegian health judge.

Helsedirektoratet sexual-health judge — generic variant.

Norwegian-language judge for AI assistants that answer young people's
questions about sexual health, regardless of whether the bot uses RAG or
relies on baked-in knowledge. Six criteria including BEVARING AV USIKKERHET
OG NYANSER, plus a neutral 1–5 Norwegian severity scale that is mapped
silently to the framework's {critical, high, medium, low, pass} enum.

Based on a domain-expert review from Helsedirektoratet.

This variant softens the RAG-specific wording ("dokumentene") to
"etablerte faglige kilder og retningslinjer" so the judge applies to bots
without a fixed source corpus. For the RAG-faithful version, see
helsedir_sexhealth_no_rag.

### Constants

#### `HELSEDIR_SEXHEALTH_NO_JUDGE`

**Name:** Helsedirektoratet seksuell helse

Norsk faglig vurdering av KI-svar om seksuell helse til unge. Seks kriterier inkludert bevaring av usikkerhet og nyanser. Severitet rapporteres som {critical, high, medium, low, pass}; faglig skala 1–5 (uakseptabelt avvik … tilfredsstillende) speiles i prompten og mappes ved utdata. Generisk variant — ikke avhengig av RAG-kildekorpus.

**Source:**

- type: domain_expert_review
- organisation: Helsedirektoratet
- review_date: 2026-05-14
- notes: Kriteriene 1–5 og den faglige 1–5-skalaen er hentet fra et faglig tilbakemeldingsskjema. Kriterium 6 (BEVARING AV USIKKERHET OG NYANSER) ble lagt til av domeneeksperten som tillegg utover SimpleAudits standard fem dimensjoner. Per-scenario forventet atferd ble ikke spesifisert; denne dommeren vurderer på generelle kriterier alene. Denne varianten har myknet opp RAG-spesifikk formulering («dokumentene») til «etablerte faglige kilder og retningslinjer» slik at vurderingen også gjelder boter uten fast kildekorpus.


