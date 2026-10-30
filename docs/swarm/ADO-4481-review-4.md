# ADO-4481 — review giro 4

Verdetto: **approve**. Nessun finding.

## Base e commit

- Commit dichiarato `a14daf5a73474497946e75344fec6e74e836bdc4`: presente nel worktree.
- `a14daf5..HEAD` contiene solo `aff0f4e`, firmato Loom, che tocca solo `docs/swarm/ADO-4481.md` (documento del collector, non imputabile ad ACT). ACT in questo giro non ha introdotto modifiche.
- Il contenuto del task (commit `41f94fc` e `bb392be` dello stesso nodo) è stato rigiudicato da zero sul file a HEAD.

## Riapertura

`reopened_for` contiene solo la risposta di una persona (commento 9043454): il criterio skill-validator v1.5.6 è verificato dalla CI della PR (`.github/workflows/skill-lint.yml`, XG-ADR-045), non in locale. `act_result.addressed_findings` è vuoto: nessuna affermazione da verificare.

## Check

| # | Check | Esito |
|---|---|---|
| 1 | Documentazione richiesta | pass — il task non richiede documenti oltre la skill |
| 2 | Scope | pass — `c4ec5ee..HEAD` tocca solo `skills/authoring-dataverse-plugin-scenarios/SKILL.md` e `docs/swarm/ADO-4481.md` (collector) |
| 3 | File vietati | pass — workflow, README e l'altra skill non toccati |
| 4 | Criteri di accettazione | pass — vedi sotto |
| 5 | Verifica | pass — validator delegato alla CI della PR per decisione esplicita della persona |
| 6 | Commit | pass |
| 7 | Regressioni (giro > 1) | pass — nessuna modifica di contenuto rispetto al giro precedente |

## Criteri di accettazione (verifica diretta su SKILL.md a HEAD)

- Stato del run / precedenza / nessuna scrittura su scenario o Dataverse: righe 207-216; affermazione obsoleta sulla coerenza limitata al Retrieve successivo assente, sostituita da riga 182 (vale in mock, in auto solo per record che Dataverse non ha).
- Tombstone → 0x80040217 su Update/Delete in ogni modalità (riga 241); auto/live rispondono da Dataverse (179-180); precedenza dichiarata per mock (170).
- Percorso felice Update/Delete mock vs auto/live (247-250); typeName completo FaultException`1 (200, 270).
- Update conserva gli attributi, Delete tombstone, nessuna scrittura live (227-228, 214, 417).
- Marker di versione distinto (208, 310) e nota 1.7.6 ristretta (62).
- Execute: CRUD delegati su ogni licenza, quattro metadata Pro, NotImplementedException per Custom API (310-321, 351).
- Asimmetria retrieveResponses/RetrieveMultiple mantenuta (140).
- Tabella verbo→effetto→fault con righe e codici richiesti (222-233), riga Create con 0x80040237 e record cancellato non noto.
- Frase sul live ristretta a Retrieve/RetrieveMultiple (415).

## Osservazioni (non finding)

- Il gate del validator è ora sulla CI della PR: se skill-lint fallisce, il task si riapre con quell'evidenza, come stabilito dalla persona.
