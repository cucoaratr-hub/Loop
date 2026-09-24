# Loop — arhitectură și decizii

> Stare: propunere consolidată. Rolurile Planner–Coder–Verifier–Runtime sunt separate; software-ul concret este marcat selectat, candidat sau neales. Actualizat: 2026-09-24. `loop.md` este documentul autoritar; `spec.md` este obiectivul fiecărui produs.

## 1. Scop și constrângeri

Loop trebuie să transforme aceeași formă de specificație Markdown în șase produse: newsletter, analiză piață crypto, creare video, analiză PDF pentru investitori, constructor de baze de date cu audit și constructor de website-uri.

Utilizatorul este non-programator:

- nu citește cod, diff-uri, terminal sau JSON;
- nu scrie și nu întreține scripturi tehnice;
- aprobă milestone-uri numai prin comportamentul observabil al produsului;
- primește exact: **Ce s-a construit**, **Cum testezi**, **Ce rezultat trebuie să vezi pe ecran**.

Constrângeri fixe:

- Repozitoarele sunt private în GitHub și păstrează codul, specificația, PR-urile, starea și dovezile.
- 9router de la `localhost:20128`, cu configurația RTK și fallback-ul existent, este infrastructură fixă și nu se reconfigurează.
- Serverul local are V100 și Ollama cu modelul Qwen3.8-27B declarat de utilizator; tagul exact și performanța trebuie verificate.
- OpenCode Go și Merge Gateway sunt alocări existente, dar modelul/providerul concret rămâne agnostic momentan.
- Plannerul va putea folosi ulterior un model din oferta OpenCode Go; alegerea nu este făcută în acest document.
- Codarea și continuous improvement sunt locale; planificarea și verificarea sunt online, cu modele diferite.

## 2. Separarea proprietarilor

| Proprietar | Responsabilitate | Nu are voie să facă |
|---|---|---|
| `spec.md` | Obiectivul și criteriile produsului | Nu conține implementare |
| Planner | Transformă obiectivul în task-uri verificabile, așteaptă verdictul și decide următoarea etapă | Nu scrie cod, nu rulează implementarea, nu declară singur succesul |
| Coder | Implementează un singur task într-un workspace izolat și predă dovezi | Nu schimbă planul, `spec.md`, criteriile sau aprobă/îmbină propriul rezultat |
| Verifier | Verifică independent codul și dovezile contra planului și `spec.md` | Nu implementează corecția și nu înlocuiește aprobarea umană |
| Runtime | Rulează workflow-ul durabil, pause/resume, retry, lock, idempotency și recuperarea | Nu decide sensul produsului și nu este modelul |
| GitHub/CI | Păstrează versiuni și execută verificări deterministe | Nu interpretează singur intenția produsului |
| Utilizator | Aprobă sau respinge comportamentul observabil | Nu trebuie să inspecteze codul |

## 3. System Component Matrix

| Componentă software | Rol exact | Local/online și model | Status |
|---|---|---|---|
| `spec.md` | Definește obiectivul și criteriile | GitHub privat; fără model | Selectat ca input standard |
| OpenClaw Gateway | Canal Telegram, rutare pe proiect/agent, conversație planner și livrare HITL | Server local; modelul planner este configurabil online | **Candidat practic pentru Planner** |
| Telegram | Interfața utilizatorului pentru cereri, clarificări, aprobări și feedback | Online; fără model | Canal ales pentru planner |
| Planner model/provider | Transformă `spec.md` în planuri și task-uri și replanifică după verdict | Online; provider/model neselectat, ulterior posibil din OpenCode Go | **Agnostic / neselectat** |
| Google AX | Runtime/orchestrator candidat pentru Task, Workspace, sandbox, resurse, rețea și suspend/resume | Server/cluster local; fără model propriu | **Candidat Runtime; nevalidat** |
| Coder executor | Un singur executor ales dintre DSH și OpenCode | Local pe V100, Qwen3.8-27B prin Ollama | **DSH vs OpenCode: benchmark necesar** |
| Verifier software | Agent/workflow separat care citește direct spec, task, commit și CI | Online, model diferit de Planner; software concret încă neselectat | **Neselectat** |
| GitHub | Repository privat, branch, PR, commit, artefacte și istoric | Online; fără model | Selectat |
| CI runner | Teste, build, lint, type, security și teste de UI unde se aplică | Runner verificat; fără model | Selectat ca rol; implementarea exactă de verificat |
| Playwright | Teste browser și trace pentru produse cu UI web | Runner de test; fără model | Opțional per proiect |
| `.loop/loop-state.json` | Stare durabilă, attempts, SHA-uri, task, verdict și recovery metadata | GitHub privat; fără model | Contract obligatoriu |
| Jev/System One | Model/API de decizii tipizate, dacă va fi folosit; nu planner, coder sau runtime | Online; provider TypeSafe AI, endpoint și cost de verificat | **Opțional decision layer; nu este necesar pentru prima versiune** |

## 4. Ce face Plannerul

Plannerul execută următoarea buclă:

```text
Telegram request
  -> OpenClaw identifică project_id
  -> citește spec.md și starea proiectului
  -> planner model online creează/actualizează planul
  -> produce un singur task/goal verificabil
  -> trimite task-ul coderului prin Runtime
  -> așteaptă candidate_result și verdictul Verifierului
  -> decide: corecție, task nou, realiniere, HITL sau finalizare
```

Plannerul nu transmite conversația completă coderului. Transmite un pachet versionat:

```json
{
  "project_id": "...",
  "spec_sha": "...",
  "plan_sha": "...",
  "milestone_id": "...",
  "task_id": "...",
  "goal": "un singur obiectiv verificabil",
  "scope": ["..."],
  "acceptance_criteria": ["..."],
  "required_tests": ["..."],
  "starting_head_sha": "...",
  "iteration": 1,
  "max_attempts": 3,
  "stop_condition": "...",
  "privacy_class": "..."
}
```

### Cerințele Plannerului

- **PL1:** Păstrează `project_id` și `spec_sha` exacte.
- **PL2:** Extrage obiectivul, cerințele și criteriile fără să inventeze implementarea.
- **PL3:** Transformă cerințele în milestone-uri și task-uri mici, verificabile.
- **PL4:** Fiecare task are goal unic, precondiții, scope, criterii, teste și condiție de oprire.
- **PL5:** Trimite task-ul structurat coderului și așteaptă `candidate_result`.
- **PL6:** Primește verdictul Verifierului: `pass`, `fail`, `unknown`, `drift`, `blocked`, `regression`.
- **PL7:** Transformă verdictul în următoarea acțiune.
- **PL8:** Compară periodic planul și progresul direct cu `spec.md`.
- **PL9:** Invalidează planurile/aprobările afectate de schimbări în `spec.md`, plan, scope sau head SHA.
- **PL10:** Izolează proiectele, workspace-urile, sesiunile și starea.
- **PL11:** Gestionează `blocked`, `stalled` și `exhausted` fără succes fals.
- **PL12:** Cere decizia utilizatorului când scopul este ambiguu sau trebuie schimbat.
- **PL13:** Produce mesajul HITL în cele trei secțiuni fără cod sau diff.
- **PL14:** Este agnostic față de model/provider și acceptă alegerea ulterioară OpenCode Go.

## 5. Ce face Coderul

Coderul primește un singur task, nu întregul proiect:

```text
AX/Runtime workspace izolat
  -> context nou
  -> citește task + fișiere relevante
  -> modifică numai scope-ul
  -> rulează testele relevante
  -> produce commit și candidate_result
  -> se oprește
```

Cerințe coder:

- pornește de la `spec_sha`, `plan_sha` și `head_sha` exacte;
- lucrează pe branch/workspace izolat;
- primește context nou la fiecare iterație;
- modifică numai scope-ul task-ului;
- păstrează rezultatele brute ale testelor;
- produce commit, modificări, teste, erori și restanțe;
- nu modifică `spec.md`, criteriile sau politica Loop;
- nu face merge și nu își aprobă rezultatul;
- are maximum trei încercări pentru aceeași problemă;
- codarea și continuous improvement folosesc local Qwen3.8-27B/Ollama;
- executorul concret este DSH **sau** OpenCode, ales prin benchmark, nu ambele simultan.

## 6. Ce face Verifierul

Verifierul este separat logic de Planner și Coder și preferabil folosește alt model online. Primește direct:

```text
spec.md la spec_sha
plan/task la plan_sha
candidate commit/head_sha
rezultate CI
candidate_result
artefacte de test
verdictul anterior
```

Pentru fiecare criteriu emite:

```text
pass | fail | unknown | blocked | drift | regression
+ evidence
+ risk
+ missing proof
+ next recommended correction
```

Cerințe verifier:

- nu se bazează numai pe rezumatul coderului;
- verifică fiecare criteriu;
- detectează drift față de `spec.md`;
- confirmă că testele nu au fost slăbite;
- leagă verdictul de head SHA și artefactele exacte;
- recomandă următoarea problemă, dar nu o implementează;
- nu aprobă în locul utilizatorului;
- software-ul concret al verifierului rămâne neselectat și trebuie ales/testat separat.

## 7. Cerințe Runtime

Runtime-ul trebuie să ofere:

- checkpoint după fiecare etapă importantă;
- multi-project isolation;
- pause/resume pentru HITL;
- crash recovery;
- retry cu clasificare transient/terminal;
- idempotency și deduplicare;
- un singur writer per proiect;
- correlation IDs și observabilitate;
- stop switch;
- model/provider agnostic;
- niciun script întreținut de utilizator.

AX rămâne candidat pentru acest rol, dar nu i se atribuie automat implementarea tuturor invariantelor Loop. OpenClaw rămâne candidat pentru interfața Telegram și planner gateway, nu memorie autoritară de codare.

## 8. Planner software — comparație și decizie

| Loc | Software candidat | Telegram | Multi-project | Plan structurat | Model agnostic | Durabilitate/HITL | Verdict |
|---:|---|---:|---:|---:|---:|---:|---|
| 1 | OpenClaw Gateway + workflow rules | 10 | 9 | 8 | 8 | 8 | Candidat practic actual pentru Planner; necesită contracte Loop explicite |
| 2 | LangGraph + Telegram adapter | 6 | 9 | 10 | 10 | 9 | Candidat tehnic puternic; necesită adapter și operare software |
| 3 | Microsoft Agent Framework Workflows | 5 | 9 | 10 | 10 | 10 | Candidat robust de workflow; Telegram extern |
| 4 | Temporal + planner agent | 5 | 10 | 8 | 10 | 10 | Candidat foarte robust pentru durabilitate; planner AI separat |
| 5 | PydanticAI + Temporal | 4 | 9 | 10 | 10 | 10 | Candidat type-safe și durabil; complex |
| 6 | Mastra Workflows | 4 | 8 | 9 | 9 | 8 | Candidat TypeScript cu suspend/resume |
| 7 | CrewAI Flows | 4 | 8 | 9 | 9 | 8 | Candidat pentru flows, necesită implementare |
| 8 | Google ADK | 3 | 8 | 9 | 8 | 8 | Candidat graph-agent; Telegram extern |
| 9 | Hermes | 9 | 6 | 6 | 8 | 6 | Bun pentru Telegram/agent personal, mai slab pentru planner durabil |
| 10 | OpenAI Agents SDK/AutoGen-style | 3 | 7 | 7 | 8 | 6 | Agent framework, nu planner complet |

### Decizia curentă

- **Planner gateway și Telegram:** OpenClaw Gateway — candidat practic actual.
- **Planner model:** agnostic; va fi ales ulterior, probabil din OpenCode Go.
- **Planner workflow runtime:** nu se declară automat OpenClaw complet; contractul Loop și testele de durabilitate sunt obligatorii.
- **LangGraph, Microsoft Agent Framework, Temporal, PydanticAI+Temporal, Mastra, CrewAI, Google ADK, Hermes și OpenAI/AutoGen:** candidați de comparație, nu componente instalate simultan.
- **AX:** runtime candidat, separat de Planner.
- **Jev:** optional decision layer; nu este software-ul Plannerului.

## 9. Fluxul informațional complet

```text
Telegram
 -> OpenClaw Gateway
 -> project_id + session isolated
 -> spec.md + loop state
 -> planner model online
 -> plan/milestone/task package
 -> Runtime/AX
 -> one coder: DSH OR OpenCode + local Qwen3.8-27B
 -> candidate commit/result
 -> CI deterministic checks
 -> separate online Verifier
 -> verdict per criterion
 -> Planner
 -> correction / replan / goal alignment / HITL / finalization
 -> human 3-step approval
 -> protected merge
 -> verified checkpoint
```

Verdictul este singurul feedback care poate genera următorul task; conversația veche a agentului nu este transferată. Dacă `spec_sha`, `plan_sha`, `head_sha`, testele obligatorii sau scope-ul se schimbă, dovezile și aprobările afectate se invalidează.

## 10. State machines

### Idea-to-Product

`INGEST_SPEC -> VALIDATE_SPEC -> PLAN_ONLINE -> GOAL_TRACE_CHECK -> HUMAN_PLAN_GATE -> RUNTIME_TASK_CREATE -> ONE_CODER_TASK -> CI_TEST -> INDEPENDENT_VERIFY -> REPLAN_OR_NEXT -> HUMAN_BEHAVIOR_GATE -> FRESHNESS_CHECK -> MERGE -> CHECKPOINT -> NEXT_MILESTONE`.

### Goal realignment

`READ_ORIGINAL_SPEC_SHA -> MAP_REQUIREMENT_TO_TASK_TEST_EVIDENCE -> INDEPENDENT_GOAL_AUDIT -> ALIGNED | DRIFTED | AMBIGUOUS | INCOMPLETE | REGRESSION`.

- `DRIFTED`: oprește codarea și reface planul.
- `AMBIGUOUS`: cere decizia utilizatorului.
- `INCOMPLETE`: creează task pentru criteriul lipsă.
- `REGRESSION`: redeschide criteriul anterior.
- `ALIGNED`: continuă către HITL sau următorul milestone.

### Self-improvement

`IDLE -> BOUNDED_LOCAL_AUDIT -> RUNTIME_TASK_CREATE -> LOCAL_CODER -> CI -> INDEPENDENT_VERIFY -> GOAL_ALIGNMENT -> HUMAN_GATE -> MERGE_OR_REJECT -> IDLE`.

### Recovery

`RESTART -> LOAD_CHECKPOINT -> RECONCILE_GITHUB_CI_RUNTIME -> RESUME | ESCALATE`. Maximum trei încercări pentru eșec tranzient; eroare terminală sau efect extern ambiguu oprește bucla fără retry orb.

## 11. Non-Technical HITL

Fiecare milestone are exact:

1. **Ce s-a construit.**
2. **Cum testezi în interfața normală.**
3. **Ce rezultat trebuie să vezi pe ecran.**

Utilizatorul răspunde `Aprob`, `Respinge` sau `Schimbă`. Nu citește cod, diff, terminal sau JSON. Aprobarea este legată de `project_id`, milestone, PR, `head_sha`, teste și timestamp. Orice schimbare relevantă cere reverificare. Operațiile financiare, publicarea, ștergerea și datele reale au o poartă separată.

## 12. Log of Disregarded Options

| Opțiune | Decizie |
|---|---|
| Planner nedefinit ca rol fără software | Corectat: OpenClaw Gateway este candidatul concret pentru planner gateway; modelul rămâne neselectat |
| Cerințe planner/coder/verifier amestecate | Corectat prin secțiuni și proprietari separați |
| OpenClaw ca memorie autoritară permanentă | Respins: risc de drift; adevărul este în artefactele versionate |
| AX ca planner, verifier sau sursă de adevăr | Respins: AX este runtime candidat |
| Jev ca planner/orchestrator | Respins: Jev este model/API de decizii tipizate, opțional |
| AX + DSH + OpenCode simultan | Respins fără benchmark; se alege un singur executor |
| Cloud pentru fiecare iterație de codare | Respins implicit: codarea locală; online planner și verifier |
| Merge autonom sau aprobare prin diff | Respins: HITL comportamental obligatoriu |
| Retry orb, context perpetuu, green checks = succes | Respinse: stare explicită, context nou, dovadă și limită de încercări |

## 13. Stare de validare

Arhitectura definește acum roluri și candidați software fără a pretinde că integrarea este instalată. Nu sunt încă validate: OpenClaw Telegram pe configurația utilizatorului, izolarea celor șase proiecte, contractele de output, software-ul verifierului, integrarea AX, alegerea DSH versus OpenCode, modelul plannerului din OpenCode Go, recovery după crash și testele end-to-end. Până atunci nu folosim afirmațiile `24/7`, `self-healing` sau `production-ready`.
