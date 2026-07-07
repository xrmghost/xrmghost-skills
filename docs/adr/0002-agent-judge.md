# ADR 0002: agente-giudice consultivo per valutazione qualitativa Agent Skills

## Status

Accepted - approvazione maintainer registrata su ADO-1470 e ADO Wiki (`/Architecture/ADRs/xrmghost-skills/ADR-0002-agent-judge-consultivo`).

## Decisione

Raccomandazione: **buy ora, build solo se il volume cresce**. Per `xrmghost-skills` (repo pubblico, Markdown puro, single-maintainer) la scelta piu pragmatica e **SkillCheck Pro come strumento consultivo locale**, con output sempre sottoposto a revisione umana e **senza alcun gate automatico in CI**.

Non raccomando una build immediata: il costo operativo e basso, ma il problema principale non e il costo del modello; e la necessita di curare rubriche, soglie, calibrazione dei falsi positivi e manutenzione continua per evitare overreach su un repository piccolo.

## Contesto

- Repo target: `xrmghost-skills`
- Wave 2: repository pubblico, Markdown puro, nessuna implementazione di workflow in questo task
- Vincolo assoluto: qualsiasi soluzione deve essere **consultiva**, con **firma umana obbligatoria** e **mai** come gate automatico che blocca merge/PR
- Paper di riferimento: **arXiv:2603.16572 - Context Matters: Repository-Aware Security Analysis of the Agent Skill Ecosystem**

Il paper segnala che scanner che analizzano le skill in isolamento possono classificare come malevole fino al **46.8%** delle skill, mentre l'analisi repository-aware riduce i casi sospetti allo **0.52%**. Implicazione pratica: per questo repo bisogna privilegiare strumenti o prompt che leggano **contesto repo completo**, README, struttura `skills/` e convenzioni del progetto, evitando judgement description-only.

## Opzioni buy valutate

| Opzione | Pricing pubblico | Feature rilevanti | Integrazione GitHub | Fit per questo repo | Note |
|---|---:|---|---|---|---|
| **SkillCheck Pro** | **$79 lifetime**; Team **$49/mese** fino a 10 utenti (coming soon) | Validazione `SKILL.md`, check security/anti-slop/token budget, MCP server locale, GitHub Actions support | CLI/MCP locale; possibile GitHub Actions, ma qui da usare solo come advisory | **Alto** | Piu vicino al problema specifico Agent Skills; gira localmente e puo restare human-in-the-loop |
| **CodeRabbit** | Free **$0/mese/user** per public repo; Pro **$24/mese/user** annuale; Pro Plus **$48/mese/user** annuale | PR review, chat agentica, linked repository analysis, SAST/linters | GitHub app / PR review nativa | Medio | Molto conveniente per repo pubblico, ma e reviewer generalista e non skill-specific |
| **Qodo Teams** | **$38/mese/user** monthly o **$30/mese/user** annuale | PR review, CLI, context engine multi-repo, dashboard | Git integration / PR review | Medio-basso | Piu orientato a team e governance rispetto a un repo Markdown single-maintainer |
| **Greptile Pro** | **$30/seat/mese**, 50 review incluse, poi **$1/review** extra | PR review bot con codebase understanding, custom rules | GitHub PR review bot | Medio-basso | Transparente nei prezzi, ma sovradimensionato per il caso d'uso corrente |

## Opzione build

### Modello scelto

Se in futuro si costruisce internamente, raccomando **GPT-5.4 mini** invece del candidato iniziale GPT-4o-mini: oggi ha pricing pubblico chiaro e buon rapporto costo/prestazioni per analisi testuale e rubriche su repository piccoli.

### Prompt shape minima

Input per singola run:

1. README repo
2. skill modificate e diff PR
3. eventuali riferimenti condivisi in `skills/*/references/`
4. rubrica esplicita con pesi
5. istruzione di output consultivo con campi obbligatori:
   - verdict: `approve-with-review | needs-human-review | reject-suggestion`
   - confidence: 0-1
   - findings con evidenza testuale
   - possibili falsi positivi
   - nota finale: `non-blocking advisory only`

### Guardrail architetturali obbligatori

- Nessun `exit 1`, nessun required check, nessun blocco automatico su PR/merge
- Output pubblicato solo come commento advisory o artefatto di review
- Override umano sempre consentito
- Soglia alta per segnalazioni critiche; se la confidenza e bassa il sistema deve degradare a `needs-human-review`

## Costo

### Assunzioni build

- **20 PR/mese**
- **1 run advisory per PR**
- **35k token input/run** (README + skill + diff + rubric + contesto repo)
- **4k token output/run**
- Pricing OpenAI GPT-5.4 mini: **$0.75 / 1M input token**, **$4.50 / 1M output token**

### Costo build per run

- Input: 35,000 / 1,000,000 * 0.75 = **$0.02625**
- Output: 4,000 / 1,000,000 * 4.50 = **$0.01800**
- **Totale/run: $0.04425** (~$0.05)

### Costo build mensile

- 20 PR/mese * $0.04425 = **$0.885/mese**
- Con doppia run di riesame su meta dei PR: circa **$1.33/mese**

### Tabella comparativa mensile

| Opzione | Ipotesi | Costo mensile stimato |
|---|---|---:|
| SkillCheck Pro | ammortizzato su 12 mesi | **~$6.58/mese** |
| CodeRabbit Free | repo pubblico | **$0** |
| Qodo Teams | 1 maintainer | **$30-$38/mese** |
| Greptile Pro | 1 seat | **$30/mese** |
| Build interna | 20 PR/mese, 1 run/PR | **~$0.89/mese** |

## Rubrica

Questa rubrica si applica **solo se in futuro si attiva l'opzione build**. Non autorizza alcun gate automatico.

| Criterio | Peso | Soglia pratica |
|---|---:|---|
| Aderenza allo standard skill/plugin/MCP | 30% | Evidenza testuale diretta |
| Coerenza con contesto repo | 25% | Nessun finding description-only |
| Sicurezza/guardrail | 20% | Qualsiasi dubbio degrada a review umana |
| Chiarezza operativa e trigger quality | 15% | Finding supportati da esempi concreti |
| Rischio falso positivo | 10% | Se alto, non emettere suggerimento forte |

Soglia di confidenza minima accettabile per suggerimenti forti: **0.85**. Sotto **0.85** l'agente deve rispondere `needs-human-review`.

## Conseguenze

### Positive

- SkillCheck Pro minimizza il time-to-value senza introdurre workflow bloccanti
- Il fit e alto perche valuta proprio artifact di tipo skill/plugin/MCP, non solo codice applicativo
- Il riferimento a arXiv 2603.16572 spinge verso approccio repository-aware e review assistita, riducendo il rischio di falsi positivi rumorosi

### Negative / trade-off

- Build interna e piu economica sul solo costo LLM, ma non sul costo di manutenzione cognitiva
- CodeRabbit/Qodo/Greptile offrono integrazione GitHub matura, ma sono meno centrati sul dominio Agent Skills
- Anche la soluzione migliore deve restare consultiva: nessuna puo essere trattata come verdetto autonomo

## Rischi residui

- Anche con contesto repo completo, residuano falsi positivi e falsi negativi
- Pricing SaaS puo cambiare; la stima va riverificata prima dell'acquisto
- Una futura integrazione CI puo degenerare in gate di fatto se il team inizia a trattare il commento AI come approvazione automatica

## Riferimenti

- arXiv 2603.16572: https://arxiv.org/abs/2603.16572
- SkillCheck: https://www.getskillcheck.com/
- CodeRabbit pricing: https://www.coderabbit.ai/pricing
- Qodo pricing: https://www.qodo.ai/pricing/
- Greptile pricing: https://www.greptile.com/pricing
- OpenAI API pricing: https://openai.com/api/pricing/
- Architettura di prodotto XrmGhost con cui questa raccomandazione (giudizio solo consultivo, mai gate) deve restare coerente: https://docs.xrmghost.tech/architecture/
