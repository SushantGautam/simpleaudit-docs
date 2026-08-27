## simpleaudit.judges.helsedir_sexhealth_no_rag

Helsedir sex-health (NO) RAG judge: grounded variant with retrieval context.

Helsedirektoratet sexual-health judge — RAG variant.

Norwegian-language judge for AI assistants that answer young people's
questions about sexual health using a fixed source corpus (RAG-style).
Six criteria including BEVARING AV USIKKERHET OG NYANSER, plus a neutral
1–5 Norwegian severity scale that is mapped silently to the framework's
{critical, high, medium, low, pass} enum on output.

Based on a domain-expert review from Helsedirektoratet.

The RAG framing ("dokumentene") matches the bot configuration that was
reviewed. For non-RAG bots, see helsedir_sexhealth_no.

### Constants

#### `HELSEDIR_SEXHEALTH_NO_RAG_JUDGE`

**Name:** Helsedirektoratet seksuell helse (RAG)

Norsk faglig vurdering av KI-svar om seksuell helse til unge, for RAG-baserte boter med fast kildekorpus. Seks kriterier inkludert bevaring av usikkerhet og nyanser. Severitet rapporteres som {critical, high, medium, low, pass}; faglig skala 1–5 (uakseptabelt avvik … tilfredsstillende) speiles i prompten og mappes ved utdata.

**Source:**

- type: domain_expert_review
- organisation: Helsedirektoratet
- review_date: 2026-05-14
- notes: Kriteriene 1–5 og den faglige 1–5-skalaen er hentet direkte fra et faglig tilbakemeldingsskjema. Kriterium 6 (BEVARING AV USIKKERHET OG NYANSER) ble lagt til av domeneeksperten som tillegg utover SimpleAudits standard fem dimensjoner. Per-scenario forventet atferd ble ikke spesifisert; denne dommeren vurderer på generelle kriterier alene. RAG-rammingen («dokumentene») gjenspeiler at vurderingen ble utformet for en RAG-bot med fast kildekorpus.


