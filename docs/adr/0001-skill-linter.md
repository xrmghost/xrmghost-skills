# ADR 0001: Skill linter per xrmghost-skills

> Questa ADR riguarda solo la validazione strutturale del repo (`skills/*/SKILL.md`). Per il contratto architetturale piu ampio con cui la CI deve restare coerente — come le skill si inseriscono nel prodotto XrmGhost — vedi [docs.xrmghost.tech/architecture/](https://docs.xrmghost.tech/architecture/).

## Contesto

Il repository `xrmghost-skills` contiene skill Markdown-only pubblicate su GitHub. Serve un validatore OSS per `skills/*/SKILL.md` che:

- non generi false positive sulle 2 skill esistenti
- rilevi almeno frontmatter `name` mancante e `description` vuota
- sia compatibile con repo pubblico
- sia gestibile da un maintainer singolo
- supporti SARIF in modo nativo oppure tramite wrapper leggero

Durante la valutazione è stato necessario un fix di baseline autorizzato su `skills/authoring-dataverse-plugin-scenarios/SKILL.md`: la `description` del frontmatter non era quotata e i tre candidati la interpretavano come YAML invalido. Il fix è limitato al ripristino della baseline valida del repo.

## Opzioni considerate

| Candidato | Esito su skill valide | Esito su fixture invalida | SARIF | Licenza | Ultimo aggiornamento | Note |
| --- | --- | --- | --- | --- | --- | --- |
| `William-Yeh/agent-skill-linter` | exit 0, ma con warning extra su README/CI/metadata fuori boundary repo | exit 1, rileva `name` mancante e `description` vuota | No nativo; output JSON adattabile | Apache-2.0 | 2026-05-19 | Richiede Python/uv; orientato anche a publishing readiness, non solo spec validation |
| `agent-ecosystem/skill-validator` | **exit 0, zero warning sulle 2 skill dopo il fix baseline** | exit 1, rileva `name` e `description` richiesti | No nativo; output JSON/annotations facilmente wrappabile | MIT | 2026-04-29 | Binario standalone, buon fit per repo Markdown-only |
| `thedaviddias/skill-check` | exit 0, ma con 2 warning `description.use_when_phrase` sulle skill esistenti | exit 1, rileva `name` mancante e `description` vuota | **Sì, nativo** | MIT | 2026-05-18 | Migliore ergonomia `npx`, ma non soddisfa il requisito zero-false-positive sul baseline attuale |

## Decision

Adottare **`agent-ecosystem/skill-validator`** come validatore di riferimento per `xrmghost-skills`.

## Rationale

`skill-validator` è il candidato che meglio bilancia i criteri richiesti:

- è l'unico dei tre che, sul baseline del repo corretto, valida entrambe le skill esistenti senza errori né warning
- rileva correttamente la fixture invalida con exit code non-zero
- usa un binario standalone già pubblicato per Linux/macOS, senza introdurre runtime extra nel repo
- ha licenza MIT e manutenzione attiva recente
- non ha SARIF nativo, ma l'output JSON è strutturato e sufficiente per un wrapper leggero in un task separato

Gli altri candidati sono stati scartati per motivi specifici:

- `agent-skill-linter` introduce controlli di README, badge e CI fuori dal boundary di questo task, aumentando il rumore operativo
- `skill-check` ha SARIF nativo e UX `npx` migliore, ma sul baseline attuale segnala warning sulle due skill esistenti; questo contraddice il criterio prioritario di zero false positive

## Configurazione minima

### Run locale

```bash
./tools/skill-validator check skills -o json
```

### Bootstrap minimo suggerito

```bash
mkdir -p tools && \
curl -L https://github.com/agent-ecosystem/skill-validator/releases/download/v1.5.6/skill-validator_1.5.6_linux_amd64.tar.gz \
  | tar -xz -C tools
```

### Integrazione SARIF minima

Poiché `skill-validator` non esporta SARIF nativamente, l'integrazione minima raccomandata è:

1. eseguire `./tools/skill-validator check skills -o json`
2. convertire il JSON in SARIF con un wrapper dedicato minimale in task separato

Questo mantiene il validatore scelto allineato al requisito zero-false-positive senza introdurre ora modifiche a `.github/` o CI.
