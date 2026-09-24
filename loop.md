# Loop — arhitectură și decizii

> Stare: propunere consolidată cu Google AX integrat ca alternativă de runtime/orchestrare; nu este instalată sau validată integral. Actualizat: 2026-09-23. `loop.md` este documentul autoritar; `spec.md` este obiectivul fiecărui produs.

## Scop și constrângeri

Același proces trebuie să transforme specificații Markdown în șase produse: newsletter, analiză piață crypto, creare video, analiză PDF pentru investitori, constructor de baze de date cu audit și constructor de website-uri. Utilizatorul nu citește cod ori diff-uri și nu întreține scripturi; aprobă rezultate vizibile la milestone-uri. Repozitoarele sunt private în GitHub, cu recuperare după cădere. 9router la `localhost:20128`, cu compresia RTK și fallback-ul existent, și Ollama pe serverul V100 sunt infrastructură fixă; nu se reconfigurează aici. OpenCode Go și Merge Gateway sunt alocări plătite disponibile, nu nume de modele stabilite și nici autoritate de merge. Modelele online de raționament avansat pentru planificare și verificare sunt încă nedenumite; verificatorul trebuie să fie diferit de planner. Codarea iterativă și îmbunătățirea continuă se fac local pe modelul Qwen3.8-27B declarat de utilizator, cu tagul și performanța reală de verificat.

## Cercetare verificată și implicații

| Sursă citită | Concluzie aplicată | Limită |
|---|---|---|
| Google, *Agents* (textul PDF-ului din 2024, copie arhivată: https://archive.org/stream/google-ai-agents-whitepaper/Newwhitepaper_Agents_djvu.txt) | Separă modelul, instrumentele și stratul de orchestrare; ciclul observație–raționament–acțiune are stare | PDF-ul original nu a putut fi extras direct; textul citit este o transcriere arhivată |
| Google Cloud, *Choose a design pattern for your agentic AI system*: https://docs.cloud.google.com/architecture/choose-design-pattern-agentic-ai-system | Bucla are condiție de oprire, maxim de iterații, verificare și intervenție umană | Tiparul nu este un produs Loop gata instalat |
| Geoffrey Huntley, *everything is a ralph loop*: https://ghuntley.com/loop/ | O sarcină mică per iterație; progresul persistă în afara conversației | Nu preluăm scriptul exemplificativ |
| Ralph CLI, *Ralph loop*: https://ralph-cli.dev/docs/core-concepts/ralph-loop/ | Sarcini, progres și verificare repetate între sesiuni | Nu implică alegerea Ralph CLI ca software |
| Lulla et al., *Loop Engineering: Building Blocks, Adoption, and Impact*: https://arxiv.org/abs/2608.21884 | Declanșatorul, memoria, evaluatorul, limitele și recuperarea sunt piese diferite | Rezultatele empirice nu garantează acest stack |
| Addy Osmani, *Loop Engineering*: https://addyosmani.com/blog/loop-engineering/ | Automatizarea include verificare, instrumente, izolare și feedback | Nu înlocuiește aprobarea umană |
| *Engineering the Loops that Replace Step-by-Step Prompting*: https://arxiv.org/html/2607.00038v1 | Progresul real și stările `blocked`, `stalled`, `exhausted` trebuie deosebite de `success` | Reguli de proiectare propuse, de testat pe proiectele noastre |
| Google AX repository: https://github.com/google/ax | AX declară task-uri agentice, pregătește workspace-uri, sandboxează execuția, controlează gateway-ul de rețea și configurează modele; este infrastructură de execuție/orchestrare, nu automat planner/verificator | Proiect recent/pre-release; cere verificarea versiunii, Kubernetes/Agent Substrate și compatibilității cu un singur V100 |

Documentația software folosită: OpenClaw Automations https://docs.openclaw.ai/automation/cron-jobs și ACP https://docs.openclaw.ai/tools/acp-agents ; OpenCode server https://opencode.ai/docs/server/ și config https://opencode.ai/docs/config/ ; Playwright trace viewer https://playwright.dev/docs/trace-viewer ; GitHub Actions self-hosted runners https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow . Aceste pagini confirmă capabilități documentate, nu instalarea și compatibilitatea configurației utilizatorului.

## System Component Matrix

| Software/componentă | Ce face exact | Model și loc | De ce / interconectare |
|---|---|---|---|
| `spec.md` | Definește obiectivul și criteriile fiecărui produs | Fără model; GitHub privat | Autoritatea semantică; se fixează SHA pentru fiecare plan |
| OpenClaw Gateway + Automations | Declanșează execuții izolate, livrează mesaje și aprobări; nu este singurul garant al stărilor Loop | Server local; plannerul apelat prin ruta online existentă | Leagă interfața umană de planificare și de agentul de codare; nu este obligatoriu dacă AX preia execuția/controlul task-urilor |
| Planner online | Descompune obiectivul, produce milestone-uri, sarcini și teste de acceptare, apoi realiniază planul după verdict | Model online de high reasoning, încă nedenumit | Produce `plan` și `task` versiunate, nu scrie cod |
| Google AX | Orchestrator/runtime de workload-uri agentice: declară `Task`, pregătește `Workspace`, execută agentul în sandbox și folosește `Gateway` pentru controlul rețelei; poate suspenda/reporni task-uri | Fără model propriu; invocă agentul și modelul configurate în resursele AX | Candidat pentru izolare, resurse, suspendare și reluare; nu decide următoarea sarcină și nu verifică goal-alignment. Repozitoriul oficial îl descrie ca open agentic orchestration runtime; compatibilitatea locală rămâne de testat |
| DSH sau OpenCode | Executor/harness de codare care primește task-ul de la AX, modifică branch-ul, rulează verificări și produce candidatul | Local pe V100 cu Qwen3.8-27B prin infrastructura fixă; alegerea DSH versus OpenCode rămâne nevalidată | AX poate sta dedesubt ca runtime, dar nu trebuie instalate ambele înainte de benchmark |
| Verificator online independent | Confruntă candidatul, testele, planul și DIRECT `spec.md`; emite verdict și constatări | Model online diferit de planner, încă nedenumit | Nu acceptă doar rezumatul implementatorului; nu poate aproba în locul utilizatorului |
| GitHub privat | Păstrează cod, specificație, plan, branch/PR, checkpoint, dovezi și decizii | Online, fără model | Identifică exact commit-ul verificat; branch protejat pentru implementare |
| GitHub Actions CI | Rulează teste reproductibile la PR, nu bucla permanentă de planificare | Runner adecvat, fără model | Rezultatele se leagă de PR head SHA; cron cloud nu este schedulerul local principal |
| Playwright când există UI web | Rulează teste browser și produce dovezi vizuale/trace | Runner de test, fără model | Opțional pentru web; alte proiecte primesc teste corespunzătoare comportamentului lor |
| `.loop/loop-state.json` + evidență | Stochează starea durabilă, numărul de încercări, commit-uri, aprobări, blocaje | GitHub privat, fără model | Recuperarea compară checkpoint-ul cu GitHub/CI; simplul JSON nu oferă lock tranzacțional |
| Control de politică | Impune lock pe proiect, timeout, deduplicare, interdicție de merge neaprobat și oprire | Mecanism determinist; poate folosi AX pentru execuție, dar nu se presupune că AX îl implementează integral | Păstrează invariantelor Loop în afara modelului; integrarea exactă este de validat |

### AX versus DSH

AX este alternativă la nivel de runtime/control al task-urilor, nu înlocuitor direct pentru un agent harness precum DSH. AX poate rula DSH sau OpenCode ca executor în `Task`/`Workspace`. DSH gestionează bucla agentului, pluginurile și sesiunile; AX gestionează sandbox-ul, workspace-ul, resursele și rețeaua. AX poate deveni alternativa infrastructurală la partea de execuție/control asociată OpenClaw, dar nu înlocuiește automat plannerul, verificatorul, GitHub sau politica HITL.

Configurația de referință este:

```text
Planner online
      -> AX Task
      -> AX Workspace + Gateway
      -> DSH sau OpenCode
      -> Qwen3.8-27B local pe V100
      -> branch/PR GitHub
      -> CI
      -> Verificator online diferit
      -> Goal-alignment cu spec.md
      -> Human gate
      -> checkpoint
```

Nu se acceptă configurația AX + DSH + OpenCode simultan fără benchmark; DSH și OpenCode sunt alternative posibile pentru executorul de codare.

## Contractul de informații

Fiecare artefact are `project_id`, `spec_sha`, `milestone_id`, `task_id`, `iteration`, `branch`, `head_sha`, `created_at` și `artifact_hash`. Plannerul online produce `plan` (criterii trasabile la `spec.md`) și `task` (o singură schimbare, teste de succes, limite). AX atașează task-ul la un `Task` și `Workspace` identificabile și raportează starea runtime, resursele, suspendarea și reluarea. DSH/OpenCode produce `candidate_result` (commit, teste executate, rezultate brute, eroare, restanțe), fără a se certifica singur. CI atașează rezultatele la același `head_sha`. Verificatorul online citește specificația, planul, candidatul și dovezile; produce `verdict` pentru fiecare criteriu: `pass`, `fail`, `unknown`, cu dovadă și abaterea de la obiectiv. Plannerul transformă doar verdictul verificat în `next_task`, cu motiv și prioritate. Un model nu modifică retrospectiv criteriile pentru a face un eșec să pară succes.

Dacă se schimbă `spec_sha`, planul, codul, testele obligatorii sau PR head, dovezile și aprobările afectate sunt invalidate. AX runtime state nu înlocuiește checkpoint-ul Loop: la suspendare/restart, se reconciliază starea AX cu GitHub și `.loop/loop-state.json` înainte de reluare.

## Deterministic Loop State Machines

### Bucla Idea-to-Product

`INGEST_SPEC -> VALIDATE_SPEC -> PLAN_ONLINE -> GOAL_TRACE_CHECK -> HUMAN_PLAN_GATE -> AX_TASK_CREATE -> ONE_LOCAL_CODING_TASK -> CI_TEST -> INDEPENDENT_ONLINE_VERIFY -> REPLAN_OR_NEXT -> HUMAN_BEHAVIOR_GATE -> FRESHNESS_CHECK -> MERGE -> CHECKPOINT -> NEXT_MILESTONE`.

La fiecare iterație locală: AX pornește un task/workspace izolat și executorul DSH sau OpenCode primește un context nou cu sarcina, criteriile, versiunea exactă și ultimul verdict. Se scrie și se testează local; AX predă starea runtime și artefactele; CI testează commit-ul; verificatorul online evaluează dovezile; plannerul emite următoarea sarcină. Prioritate: securitate/pierdere de date, teste obligatorii eșuate, criterii neîndeplinite, apoi funcții noi. După trei încercări nereușite la aceeași eroare remediabilă: `STALLED/ESCALATE`, nu relansare oarbă. `SUCCESS` cere criterii dovedite, teste obligatorii și acceptare umană; `BLOCKED`, `STALLED` și `EXHAUSTED` nu sunt sinonime cu succesul.

### Bucla de realiniere la obiectiv

La fiecare milestone și înainte de final: `READ_ORIGINAL_SPEC_SHA -> MAP_EACH_REQUIREMENT_TO_PLAN_TASK_TEST_EVIDENCE -> INDEPENDENT_GOAL_AUDIT -> ALIGNED | DRIFTED | AMBIGUOUS`. `DRIFTED` oprește AX și orice codare nouă; plannerul online repară împărțirea și ordinea sarcinilor și retrimite verificarea. `AMBIGUOUS` cere decizie utilizatorului. Doar utilizatorul aprobă schimbarea scopului în `spec.md`; schimbarea invalidează selectiv planurile, Task-urile AX și aprobările afectate. Planul nu devine niciodată substitutul specificației.

### Bucla Self-Improvement

`IDLE -> BOUNDED_LOCAL_AUDIT -> AX_TASK_CREATE -> LOCAL_TEST/IMPROVEMENT_CANDIDATE -> CI -> INDEPENDENT_ONLINE_VERIFY -> GOAL_ALIGNMENT -> HUMAN_GATE -> MERGE_OR_REJECT -> RELEASE_AX_TASK -> IDLE`. Codarea și îmbunătățirea rulează local, cu limită de GPU, coadă și timp; apelurile online apar pentru planificare/verificarea candidaților, nu pentru fiecare încercare de sintaxă. Agentul nu slăbește testele și nu poate îmbina propriul PR.

### Bucla Recovery & Fault-Tolerance

`RESTART -> LOAD_LAST_VERIFIED_CHECKPOINT -> RECONCILE_AX_TASK_WORKSPACE_GITHUB_PR_CI -> RESUME | HUMAN_ESCALATION`. Starea conține `schema_version`, `spec_sha`, `plan_sha`, `task_id`, `ax_task_id`, `ax_workspace_id`, `iteration`, `head_sha`, `pr_number`, `verdict_hash`, `attempts`, `lease_epoch`, `state_version`, `last_verified_checkpoint`, `blocked_reason`. Un singur writer per proiect, fencing al workerilor vechi, verificarea commit-ului înainte de resume și idempotency key pe acțiunile externe. Eșec tranzitoriu: maximum trei încercări; eroare terminală sau rezultat ambiguu al unei acțiuni externe: oprire/reconciliere, fără retry orb. Suspendarea AX nu este dovadă că milestone-ul a reușit. Resetarea contextului nu resetează starea sau încercările. Niciun model, AX, DSH sau OpenCode nu are autoritate să pretindă că un merge, test sau efect extern a reușit fără dovadă.

## Non-Technical HITL Protocol

Fiecare milestone prezentat utilizatorului are EXACT: (1) **Ce s-a construit**, (2) **Cum testezi în interfața normală**, (3) **Ce rezultat trebuie să vezi pe ecran**. Utilizatorul răspunde `Aprob`, `Respinge` sau `Schimbă`; nu citește cod, diff, terminal ori JSON. Pentru un milestone pur backend, se oferă un test vizibil ori o aprobare explicită pentru infrastructură fără a pretinde un rezultat pe ecran. Aprobarea este legată de proiect, milestone, PR, `head_sha`, test și timp; orice modificare relevantă cere reverificare. Testele automate și verdictul online sunt necesare, dar nu înlocuiesc aprobarea comportamentală. Operațiile cu efect financiar, publicare, ștergere sau date reale cer o poartă separată.

## Verificări înainte de a declara stackul integrat

1. GitHub: repository privat, branch protection, permisiuni minime, aprobare invalidată la PR head nou.
2. Rutare: identitatea efectivă și localitatea fiecărei rute, fallback fără trimitere silențioasă a datelor local-only online; fără reconfigurarea 9router.
3. Model local: tagul real, VRAM, concurență, context și teste de codare reprezentative pe V100.
4. AX: versiune pinuită, cerințe Kubernetes/Agent Substrate, registry, sandbox, workspace persistent, gateway allowlist, suspend/resume, GPU passthrough și cost operațional pe un singur V100.
5. Executor: benchmark DSH versus OpenCode într-un task identic; sesiune one-shot, permisiuni restrictive, PR izolat și fără script întreținut de utilizator.
6. Contract: AX task/workspace, candidatul, CI, verdictul, următoarea sarcină și verificarea obiectivului sunt legate de aceleași versiuni.
7. HITL: aprobare/respingere/feedback în limbaj simplu, fără diff.
8. Recuperare: întrerupere în fiecare punct de efect extern, reconciliere AX/GitHub/CI fără duplicare și fencing al workerilor vechi.
9. Escaladare: maximum trei încercări pentru eșec tranzitoriu, stop imediat pentru terminal/ambiguu, limită de cost și stop switch.
10. Portabilitate: aceleași reguli pe toate cele șase tipuri de proiecte, cu teste specifice produsului.

## Log of Disregarded Options

| Opțiune | Decizie și motiv exact |
|---|---|
| n8n ca motor principal | Respins pentru acest design: queue mode poate persista execuții, dar nu dovedește nativ contractul nostru de codare, goal-alignment, context reset și gate-uri; nu afirmăm fals că nu poate avea stare |
| GitHub Actions Cron ca scheduler principal | Respins: evenimentele programate pot întârzia/fi omise; runner self-hosted poate totuși folosi GPU local și Actions rămâne pentru CI |
| OpenHands și DeepSeek Harness ca soluție completă | Nealese ca alegere curentă: OpenHands are persistență de conversație, iar DSH este harness extensibil; niciunul nu este dovedit ca control-plane complet pentru Loop fără integrare |
| Google Jules în fluxul utilizatorului | Respins conform experienței utilizatorului: revizuirea schimbărilor de cod i-a blocat verificarea comportamentală |
| GitHub `Files Changed` ca gate uman | Respins: utilizatorul verifică produsul, nu sintaxa |
| Scripturi Bash/Python întreținute de utilizator | Respinse: încalcă zero mentenanță tehnică; nici Ollama, nici OpenClaw, nici AX nu sunt presupuse că implementează automat protocolul complet |
| Merge Gateway ca planner ori merge authority | Respins: este alocare/endpoint declarat, nu aplicație de planificare sau autoritate GitHub dovedită |
| Cloud plătit pentru fiecare iterație de codare | Respins implicit: codarea și îmbunătățirea sunt locale; online doar plannerul și verificatorul independent, cu cost plafonat |
| OpenClaw ca memorie autoritară pe sesiune permanentă | Respins: drift și istoric acumulat; context nou per task, adevărul în artefactele versionate |
| Compaction OpenCode drept reset sigur | Respins: comprimarea contextului nu echivalează cu o sesiune nouă și dovadă verificată |
| AX ca înlocuitor automat pentru DSH | Respins: AX este runtime/orchestrator de workload; DSH este agent harness. AX poate rula DSH, dar nu îl înlocuiește semantic |
| AX + DSH + OpenCode simultan fără benchmark | Respins: trei straturi de execuție ar adăuga complexitate și puncte de drift fără dovadă de beneficiu |
| AX ca planner, verificator sau sursă de adevăr | Respins: AX execută/izolează task-uri; aceste roluri rămân la modelele online, GitHub și regulile Loop |
| Modelul își aprobă singur rezultatul, schimbă `spec.md`, slăbește teste ori îmbină PR | Respins: conflict cu separarea rolurilor și HITL |
| 9router redesign, model = aplicație, verde = produs corect, retry orb, bucle infinite | Respinse: încalcă baseline-ul sau confundă ruta cu execuția, testele cu obiectivul, ori produc efecte nesigure |

## Stare de validare

Aceasta este o propunere integrată pe hârtie. Nu sunt încă verificate: instalarea/configurația AX și cerințele Kubernetes/Agent Substrate pe server, integrarea AX cu DSH sau OpenCode, identitatea modelelor online, mecanismul determinist de blocare/atomicitate, ruta efectivă locală, testele celor șase produse și recuperarea după crash. Până trec verificările, nu descriem sistemul drept `24/7`, `self-healing` sau `production-ready`.
