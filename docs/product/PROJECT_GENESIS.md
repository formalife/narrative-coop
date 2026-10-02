Stiamo iniziando da zero un nuovo progetto: un **gioco narrativo cooperativo asimmetrico per due giocatori**, browser-first, nel quale due persone interpretano due personaggi differenti nello stesso mondo, ricevono informazioni differenti, prendono decisioni differenti e producono **un'unica timeline canonica**.
Questa chat deve occuparsi inizialmente soprattutto della **progettazione dell'architettura tecnica e del motore narrativo sottostante**.
NON partire ancora dalla scrittura della prima storia, dal character design, dal naming commerciale o dallo sviluppo del frontend finale.
Prima dobbiamo progettare un'infrastruttura sufficientemente robusta da poter sostenere molti episodi, molti generi narrativi, centinaia di tipi di decisioni e una crescita futura del prodotto senza dover rifare tutto da zero.
---
# 1. CONCETTO DEL PRODOTTO
Il gioco è progettato principalmente per **due persone che giocano contemporaneamente o quasi contemporaneamente dai propri smartphone/browser**.
Ogni giocatore interpreta un personaggio diverso.
I due personaggi:
- vivono nello stesso mondo;
- partecipano alla stessa storia;
- possono trovarsi nello stesso luogo oppure in luoghi differenti;
- possono avere ruoli differenti;
- possono conoscere informazioni differenti;
- possono avere obiettivi parzialmente differenti;
- possono controllare risorse differenti;
- possono avere capacità differenti;
- possono prendere decisioni senza conoscere quelle dell'altro;
- possono cooperare;
- possono entrare in conflitto;
- possono mentire;
- possono nascondere informazioni;
- possono involontariamente ostacolarsi;
- possono produrre conseguenze che modificano le decisioni successive dell'altro.
NON devono semplicemente rispondere alle stesse domande.
NON devono vivere due universi paralleli.
NON voglio un semplice compatibility test.
NON voglio un classico gioco "io vedo metà puzzle e tu l'altra metà" come unica meccanica.
La caratteristica centrale è:
# ASYMMETRIC AGENCY + CAUSAL MERGE
Le decisioni dei due giocatori vengono interpretate come **azioni reali all'interno dello stesso world state**.
Il sistema deve poi determinare cosa accade quando queste azioni:
- sono compatibili;
- sono complementari;
- competono per la stessa risorsa;
- sono temporalmente incompatibili;
- sono in conflitto;
- dipendono da informazioni differenti;
- producono conseguenze indirette;
- modificano lo stato futuro;
- cambiano il rapporto tra i personaggi;
- aprono o chiudono possibilità narrative.
Alla fine esiste sempre **una sola storia canonica**.
---
# 2. ESEMPIO PURAMENTE CONCETTUALE
Non assumere che questo sarà il primo scenario: serve soltanto a comprendere il sistema.
Scenario:
un piccolo aereo precipita su un'isola.
Player A interpreta il pilota.
Conosce:
- stato della radio;
- coordinate;
- carburante;
- danni strutturali;
- informazioni di navigazione.
Player B interpreta il medico.
Conosce:
- stato dei feriti;
- medicinali;
- condizioni fisiche;
- rischi sanitari;
- informazioni che il pilota non possiede.
Il pilota potrebbe decidere:
> usare immediatamente la batteria per trasmettere un SOS.
Il medico potrebbe contemporaneamente decidere:
> usare la stessa fonte di energia per un dispositivo medico.
Non devono essere create due timeline.
Il sistema deve sapere:
- quanta energia esiste;
- quanto consumano le due azioni;
- quale avviene prima;
- quali priorità/autorità hanno i ruoli;
- cosa sanno i due personaggi;
- quali conseguenze sono plausibili.
Il conflitto può diventare esso stesso un evento narrativo.
---
# 3. RISULTATO FINALE DEL PRODOTTO
Immagino inizialmente esperienze:
- browser-first;
- mobile-first;
- zero download;
- idealmente nessun account obbligatorio per iniziare;
- due giocatori;
- 20–35 minuti circa per episodio;
- forte replayability con partner differenti;
- episodi differenti acquistabili singolarmente o in season;
- primo episodio potenzialmente gratuito;
- modello "one payer / friend plays free";
- risultato finale facilmente condivisibile.
La vera unità di replay non dovrebbe essere solamente:
`scenario`
ma:
`scenario × partner`
Lo stesso scenario giocato con persone differenti dovrebbe produrre storie significativamente differenti.
---
# 4. VIRAL LOOP
La viralità deve essere incorporata nell'esperienza, non aggiunta artificialmente dopo.
Il primo giocatore invita una seconda persona tramite link.
Il secondo giocatore deve poter iniziare rapidamente.
Alla fine il sistema crea una sintesi condivisibile della storia comune.
NON una semplice percentuale:
> "compatibilità 78%".
Piuttosto qualcosa come:
> Luca ha distrutto le prove.
> Marco aveva già inviato una copia.
> Nessuno dei due sapeva cosa avesse fatto l'altro.
> 17 minuti dopo avevano provocato una crisi diplomatica.
Il risultato finale dovrebbe poter diventare:
- card;
- recap testuale;
- mini timeline;
- breve video verticale;
- statistiche interessanti;
- "first divergence";
- decisione più importante;
- maggior conflitto;
- conseguenza più imprevista;
- ending;
- CTA per giocare lo stesso scenario con un'altra persona.
Ma NON progettare ancora il sistema di recap nei dettagli prima di aver progettato correttamente il motore narrativo.
---
# 5. RICERCA DI MERCATO GIÀ SVOLTA
La categoria cooperative/asymmetric games è validata.
Riferimenti importanti già individuati:
- It Takes Two;
- Split Fiction;
- We Were Here;
- The Past Within;
- BOKURA;
- Tick Tock: A Tale for Two;
- Operation: Tango;
- Keep Talking and Nobody Explodes;
- CouchEscape;
- The Click.
Sono importanti soprattutto per:
- informazione asimmetrica;
- ruoli differenti;
- comunicazione obbligatoria;
- friend pass;
- one-payer model;
- cooperazione;
- narrative puzzle design.
Il nostro obiettivo NON è copiarli.
Il gap individuato è:
> la maggior parte si concentra su **asymmetric information**.
Noi vogliamo concentrarci molto di più su:
> **asymmetric agency + semantic actions + causal merge + emergent narrative**.
Il prodotto dovrebbe essere molto più decision-centric e consequence-centric rispetto a puzzle-centric.
---
# 6. ARCHITETTURE ESISTENTI DA STUDIARE
Non voglio inventare tutto da zero.
Studia criticamente e, dove utile, riutilizza principi provenienti da architetture già validate.
In particolare:
## Event Sourcing
Per rappresentare gli eventi canonici e permettere di ricostruire come il mondo è arrivato allo stato attuale.
NON applicarlo indiscriminatamente a tutto.
Va usato solo dove genera valore.
## CQRS / projections
Separare:
- modifiche al world state;
- viste ottimizzate per gameplay/UI;
- analytics;
- recap;
- narrative context.
## Entity-Component architecture
I personaggi e gli oggetti del mondo dovrebbero essere composti da componenti modulari.
Studiare in particolare il pattern:
`Entity → Components → Actions`
## Google DeepMind Concordia
Riferimento importante per:
- entity/component;
- Game Master;
- action proposal;
- action resolution;
- social simulation;
- sequential/simultaneous action engines;
- checkpoint/state.
NON assumere che dobbiamo usare direttamente l'intero framework.
Valutare cosa prendere come pattern e cosa evitare.
## Generative Agents
Riferimento per:
- observations;
- memories;
- reflections;
- planning;
- retrieval di memoria rilevante.
## AI Town
Riferimento per:
- separazione tra game engine;
- agent layer;
- UI;
- game state;
- input;
- agent state.
Studia anche altre architetture se trovi riferimenti migliori.
---
# 7. PRINCIPIO FONDAMENTALE: L'LLM NON È IL MONDO
Una AI generativa non deve avere il diritto di modificare direttamente lo stato canonico.
Flusso corretto concettualmente:
`PLAYER CHOICE`
↓
`SEMANTIC ACTION`
↓
`VALIDATION`
↓
`CONFLICT/CAUSAL RESOLUTION`
↓
`DOMAIN EVENTS`
↓
`WORLD STATE UPDATE`
↓
`NARRATIVE REALIZATION`
L'LLM può aiutare a:
- interpretare;
- descrivere;
- scrivere;
- proporre;
- trasformare eventi strutturati in narrativa naturale.
Ma NON deve decidere arbitrariamente la fisica o i fatti del mondo.
Mai:
`LLM → UPDATE DATABASE`
Il determinismo e la coerenza del sistema vengono prima della creatività linguistica.
---
# 8. SEMANTIC ACTION LAYER
Questa è una componente centrale.
Una scelta del giocatore non deve ridursi a:
`risk += 0.2`
Serve una rappresentazione semantica dell'azione.
Esempio concettuale:
```json
{
  "actor": "PLAYER_A_CHARACTER",
  "intent": "conceal_information",
  "target": "PLAYER_B_CHARACTER",
  "object": "DISCOVERY_014",
  "resources_used": [],
  "world_effects": {},
  "relationship_effects": {},
  "knowledge_effects": {}
}
```
La forma definitiva deve essere progettata molto meglio di questo esempio.
Dobbiamo poter rappresentare semanticamente azioni come:
- dire;
- nascondere;
- mentire;
- spostarsi;
- usare una risorsa;
- distruggere;
- salvare;
- sacrificare;
- aspettare;
- investigare;
- fidarsi;
- tradire;
- negoziare;
- comprare;
- spendere;
- promettere;
- rifiutare;
- condividere;
- bloccare;
- attaccare;
- fuggire;
- proteggere;
- comunicare;
- ecc.
Studia come progettare una action ontology sufficientemente generale senza diventare infinitamente complessa.
---
# 9. VETTORI NASCOSTI
Le decisioni devono modificare anche molte dimensioni numeriche interne.
Non limitarti agli esempi seguenti.
## Personaggio
- risk tolerance;
- self-preservation;
- empathy;
- honesty;
- curiosity;
- patience;
- ambition;
- impulsivity;
- loyalty;
- pragmatism;
- dominance;
- sacrifice propensity;
- rule adherence;
- suspicion;
- optimism;
- aggression;
- cooperation;
- resource conservation;
- emotional stability;
- moral flexibility;
- ecc.
## Relazione
- trust;
- affection;
- respect;
- resentment;
- dependence;
- fear;
- reciprocity;
- perceived competence;
- loyalty;
- information asymmetry;
- secrets;
- promises;
- debts;
- power imbalance;
- conflict;
- cohesion;
- ecc.
## World
dipende dallo scenario:
- money;
- energy;
- health;
- food;
- time;
- danger;
- visibility;
- stability;
- territory;
- resources;
- threat;
- reputation;
- public attention;
- access;
- evidence;
- etc.
## Narrative state
- unresolved threat;
- mystery progress;
- moral debt;
- open promise;
- hidden information;
- relationship tension;
- objective progress;
- unresolved conflict;
- dramatic irony;
- Chekhov object;
- faction allegiance;
- foreshadowing state;
- ecc.
NON decidere arbitrariamente una lista globale infinita.
Studia piuttosto un sistema:
`core dimensions + scenario-specific dimensions`.
---
# 10. WORLD STATE
Il world state deve essere indipendente dalla prosa.
Dovrebbe poter esistere anche senza nessun testo narrativo.
Deve rappresentare almeno:
- tempo;
- posizione;
- entità;
- inventario;
- risorse;
- salute;
- ambiente;
- oggetti;
- conoscenza;
- relazioni;
- commitments;
- goals;
- secrets;
- permissions;
- accessibility;
- threats;
- active narrative threads;
- historical events.
Deve essere possibile chiedere:
> "Che cosa è vero nel mondo in questo preciso momento?"
e ottenere una risposta strutturata.
---
# 11. FACT ≠ KNOWLEDGE ≠ BELIEF
Questa separazione è obbligatoria.
Esempio:
FACT:
> Marco ha sabotato la radio.
KNOWLEDGE:
> Marco sa di aver sabotato la radio.
KNOWLEDGE:
> Giulia non sa che Marco ha sabotato la radio.
BELIEF:
> Giulia pensa che la radio si sia rotta accidentalmente.
Servirà quindi un modello strutturato per:
- objective facts;
- observations;
- known facts per character;
- beliefs;
- misinformation;
- lies;
- suspicions;
- secrets;
- evidence.
Non affidarti alla memoria dell'LLM per questo.
---
# 12. CONFLICT RESOLUTION
Dobbiamo definire formalmente come il sistema unisce le azioni.
Studia almeno questi casi:
## Compatible actions
A e B possono avvenire entrambe.
## Shared-resource conflict
Entrambe richiedono la stessa risorsa limitata.
## Mutually exclusive actions
Solo una può fisicamente avvenire.
## Temporal conflict
L'ordine delle azioni cambia il risultato.
## Authority conflict
Ruoli differenti possiedono livelli differenti di controllo su una decisione.
## Information conflict
Una persona prende una decisione basandosi su informazioni false o incomplete.
## Intent conflict
Uno vuole rivelare, l'altro nascondere.
## Causal interaction
L'azione di A modifica le condizioni in cui B compie la propria.
## Hidden action
B non sa ancora che A abbia fatto qualcosa.
## Delayed consequence
Una decisione non produce effetto immediato, ma modifica un evento futuro.
Per ogni categoria dobbiamo decidere:
- regola;
- priorità;
- determinismo;
- eventuale randomness;
- registrazione degli eventi;
- cosa viene mostrato ai giocatori.
---
# 13. TEMPORAL MODEL
Non assumere che le due persone scelgano sempre nello stesso istante.
Dobbiamo poter supportare:
- decisioni simultanee;
- decisioni sequenziali;
- decisioni segrete;
- decisioni condizionali;
- finestre temporali;
- deadline;
- azioni con durata;
- eventi automatici;
- conseguenze ritardate.
Il tempo deve essere parte del motore.
---
# 14. STRUCTURE OF PLAY
Ipotesi iniziale:
## ACT I
I due giocatori ricevono:
- personaggio;
- ruolo;
- informazioni;
- prime decisioni.
Il sistema fonde gli effetti.
## ACT II
Le nuove scene e decisioni dipendono dallo stato prodotto dal primo atto.
I ruoli possono divergere ulteriormente.
## ACT III
Climax e convergenza.
Le decisioni finali devono essere fortemente influenzate da:
- relazione;
- risorse;
- segreti;
- conseguenze precedenti;
- obiettivi;
- stato del mondo.
Questa è un'ipotesi.
Valuta se una struttura diversa è tecnicamente migliore.
---
# 15. ROLE SYSTEM
I due personaggi devono avere ruoli semanticamente differenti.
Non soltanto:
Player A / Player B.
Ogni Role potrebbe definire:
- permissions;
- knowledge scope;
- capabilities;
- resources;
- responsibilities;
- authority;
- communication channels;
- blind spots;
- objectives;
- restrictions.
Esempi:
Commander / Scientist.
Pilot / Doctor.
Detective / Journalist.
Engineer / Diplomat.
Parent / Child.
CEO / Whistleblower.
Non progettare ancora gli episodi.
Progetta il sistema che permetta di creare ruoli di questo tipo.
---
# 16. NARRATIVE ENGINE
Il narrative engine NON deve decidere arbitrariamente il mondo.
Deve prendere:
- world state;
- events;
- character knowledge;
- relationship state;
- active narrative threads;
- scene goals;
e trasformarli in:
- scena;
- descrizione;
- dialogo;
- conseguenza leggibile;
- nuovi decision points.
Dobbiamo distinguere almeno:
### Simulation layer
Cosa accade realmente.
### Narrative director
Quali eventi vengono mostrati e quando.
### Narrative realization
Come vengono raccontati.
Questo è fondamentale.
La stessa sequenza di domain events può essere raccontata in modi diversi senza cambiare il canon.
---
# 17. NARRATIVE THREADS
Il sistema deve tenere traccia di thread aperti.
Esempio:
```text
THREAT-012
status: active
SECRET-003
known_by: PlayerA
hidden_from: PlayerB
PROMISE-008
actor: PlayerB
target: NPC-14
status: unresolved
```
Servirà un sistema per:
- aprire thread;
- mantenerli;
- aumentare/decrementare tensione;
- risolverli;
- abbandonarli deliberatamente;
- evitare che elementi importanti vengano dimenticati.
---
# 18. MEMORY MODEL
Distingui:
### Canonical/Event memory
cosa è realmente accaduto.
### Episodic memory
ricordi specifici dei personaggi.
### Semantic memory
conoscenze consolidate.
### Reflection
interpretazioni più astratte.
### Working context
ciò che serve per la scena corrente.
### Relationship memory
eventi specifici che spiegano fiducia/risentimento ecc.
Progetta come vengono:
- creati;
- recuperati;
- aggiornati;
- compressi;
- archiviati.
E soprattutto quali sono:
**deterministici**
e quali possono essere:
**AI-derived**.
---
# 19. DATA MODEL
Prima di scrivere codice, voglio un data model approfondito.
Analizza almeno queste entità:
```text
World
Scenario
Episode
Session
Act
Scene
Entity
Character
Role
NPC
Place
Object
Resource
Relationship
Fact
Knowledge
Belief
Secret
Goal
Commitment
NarrativeThread
Action
Choice
Option
Decision
DomainEvent
WorldState
CharacterState
RelationshipState
Memory
Reflection
Condition
Rule
Conflict
Resolution
Ending
Player
Invite
Publication
Recap
```
Non assumere che questa lista sia corretta.
Red-teamala.
Elimina ciò che è ridondante.
Aggiungi ciò che manca.
Definisci:
- ownership;
- cardinalità;
- lifecycle;
- immutabilità;
- versioning.
---
# 20. EVENT TAXONOMY
Prima di progettare lo schema DB voglio una tassonomia degli eventi.
Per esempio:
```text
CharacterMoved
ResourceConsumed
InformationRevealed
InformationConcealed
RelationshipChanged
ObjectTransferred
PromiseMade
PromiseBroken
ThreatActivated
GoalCompleted
CharacterInjured
ChoiceResolved
...
```
Ma deve essere progettata con attenzione.
Non voglio:
- un evento per ogni verbo possibile;
- un unico GenericEvent senza semantica.
Trova il livello di astrazione corretto.
---
# 21. EVENT SOURCING: USARLO CON DISCIPLINA
Identifica esattamente cosa merita Event Sourcing.
Probabili candidati:
- canonical domain events;
- session progression;
- world-changing actions;
- relationships;
- inventory/resources;
- knowledge changes;
- decisions;
- resolutions.
Probabilmente NON event-sourced:
- analytics;
- page views;
- asset generation;
- temporary UI state;
- drafts;
- logs tecnici.
Proponi una strategia ibrida.
---
# 22. PROJECTIONS / READ MODELS
Il sistema dovrà generare viste differenti dallo stesso event log.
Esempi:
### WorldCurrentState
### PlayerAView
mostra solo ciò che il personaggio A può conoscere.
### PlayerBView
idem.
### NarrativeDirectorView
può vedere tutto.
### AdminView
debug completo.
### RecapView
soltanto eventi narrativamente rilevanti.
### AnalyticsView
dati comportamentali.
Questa separazione è estremamente importante.
---
# 23. DETERMINISM
Voglio capire esattamente:
quali parti devono essere completamente deterministiche;
quali possono usare randomness seeded;
quali possono usare LLM;
quali possono utilizzare modelli probabilistici.
Ogni componente non deterministica deve poter essere:
- registrata;
- riprodotta;
- debugged.
Se il sistema genera casualmente un risultato importante dobbiamo conoscere:
`seed`
`rule`
`inputs`
`output`.
---
# 24. VERSIONING
Problema fondamentale:
pubblichiamo Scenario v1.
100.000 persone lo giocano.
Poi modifichiamo una regola.
Non possiamo cambiare retroattivamente le vecchie sessioni.
Quindi dobbiamo versionare almeno:
- scenario;
- rules;
- resolver;
- action ontology;
- narrative templates;
- prompts;
- ending logic.
Ogni sessione deve sapere esattamente quale versione ha usato.
---
# 25. DEBUGGING / REPLAY
Dobbiamo poter prendere una sessione reale e fare:
> REPLAY SESSION XYZ
ottenendo:
- input Player A;
- input Player B;
- state prima;
- actions;
- conflicts;
- resolution;
- generated events;
- state dopo;
- narrative output.
Questo è un requisito architetturale, non una feature futura.
---
# 26. VALIDATORS
Prima di considerare una scena/sessione valida dovrebbero esistere validator come:
### State validator
nessuno stato impossibile.
### Resource validator
nessuna risorsa negativa senza spiegazione.
### Knowledge validator
nessun personaggio conosce automaticamente fatti non osservati.
### Timeline validator
nessuna contraddizione temporale.
### Relationship validator
eventi e relazioni coerenti.
### Action validator
azione fisicamente/semanticamente possibile.
### Canon validator
nessuna contraddizione con eventi precedenti.
### Scenario validator
tutti i node/branch necessari raggiungibili.
### Ending validator
ending raggiungibili e coerenti.
Progetta una vera strategia di validation.
---
# 27. ARCHITETTURA TECNICA
Non fissare automaticamente questo stack senza valutarlo, ma la baseline attuale è:
## Frontend
React + TypeScript.
Mobile-first.
PWA-ready.
## Backend
Supabase/Postgres come possibile soluzione MVP.
## Realtime
Supabase Realtime o alternativa se tecnicamente migliore.
## Hosting/API
Cloudflare Workers/Pages possibile baseline.
## Analytics
PostHog.
## Media
Cloudflare R2.
## Repository
GitHub.
## Archive/master assets
Google Drive.
Valuta:
- costo;
- lock-in;
- limiti;
- facilità di sviluppo;
- scalabilità;
- observability;
- migrazione futura.
Non dobbiamo ottimizzare prematuramente per milioni di utenti.
Ma non voglio nemmeno una soluzione usa-e-getta.
---
# 28. POSSIBILE ARCHITETTURA REPOSITORY
Non assumere che questa sia corretta.
Parti da qualcosa del genere e migliorala:
```text
narrative-coop/
│
├── apps/
│   ├── player-web/
│   ├── admin/
│   └── scenario-studio/
│
├── packages/
│   ├── domain/
│   ├── schemas/
│   ├── world-engine/
│   ├── action-engine/
│   ├── resolver/
│   ├── event-store/
│   ├── projections/
│   ├── memory/
│   ├── narrative/
│   ├── validation/
│   ├── replay/
│   └── analytics-events/
│
├── scenarios/
│
├── database/
│   ├── migrations/
│   └── seeds/
│
├── prompts/
│
├── docs/
│   ├── ADR/
│   ├── architecture/
│   ├── domain/
│   └── scenario-authoring/
│
├── tests/
│
└── tools/
```
Valuta attentamente se:
- monorepo;
- packages;
- scenario storage;
- shared engine;
siano le scelte corrette.
---
# 29. SCENARIO AUTHORING SYSTEM
Una volta che il motore esiste, non voglio dover modificare codice per creare ogni episodio.
Serve quindi un:
# Scenario Definition Language
o almeno un formato strutturato YAML/JSON/DSL.
Uno scenario dovrebbe poter definire:
- roles;
- initial world state;
- entities;
- resources;
- secrets;
- knowledge;
- relationship baseline;
- acts;
- scene triggers;
- decisions;
- possible semantic actions;
- conflict rules specifiche;
- narrative threads;
- possible ending conditions;
- media assets.
Il motore è generico.
Gli scenari sono dati/configurazione.
Questa separazione è obbligatoria.
---
# 30. SCENARIO STUDIO
Prevedi già architetturalmente, anche se non dobbiamo costruirlo subito, un futuro tool interno che permetta di:
- creare scenario;
- creare personaggi;
- configurare ruoli;
- definire scene;
- definire decisioni;
- simulare due player;
- visualizzare causal graph;
- vedere world state;
- testare branch;
- fare replay;
- validare ending;
- generare synthetic playthrough.
Non voglio che la produzione dei nuovi episodi richieda edit manuale di 40 JSON.
---
# 31. SYNTHETIC TESTING
Una caratteristica molto importante futura:
usare agenti artificiali come finti player.
Generiamo per esempio:
10.000 coppie sintetiche.
Profili differenti:
- risk-seeking;
- cautious;
- selfish;
- cooperative;
- deceptive;
- altruistic;
- chaotic.
Facciamo giocare automaticamente lo scenario.
Cerchiamo:
- dead ends;
- ending impossibili;
- ending troppo frequenti;
- loops;
- contraddizioni;
- resource bugs;
- scene noiose;
- decisioni irrilevanti.
Questa possibilità deve essere prevista nell'architettura.
---
# 32. INTERFACCIA UTENTE
Per ora NON progettare il visual design definitivo.
Definisci invece le esigenze funzionali.
Browser/PWA.
Flow indicativo:
### Player A
apre scenario.
START.
Invita un partner.
Sistema genera session link.
### Player B
apre link.
Entra nella stessa sessione.
### Role assignment
ognuno vede solo il proprio ruolo.
### Gameplay
scene.
informazioni.
decisioni.
eventuale comunicazione fuori dal gioco o dentro il gioco, a seconda dello scenario.
### Synchronization points
alcuni atti attendono entrambi.
### Merge
sistema risolve decisioni.
### Scene successiva
dipende dal world state.
### Ending
storia unica.
### Recap
contenuto condivisibile.
Valuta se sia opportuno:
- no account;
- magic link;
- guest session;
- local/session ID;
- eventuale account solo successivamente.
---
# 33. COMMUNICATION
Non assumere che tutti gli episodi debbano permettere la stessa comunicazione.
Uno scenario potrebbe consentire:
- comunicazione libera;
- nessuna comunicazione;
- messaggi limitati;
- comunicazione soltanto in alcune scene;
- messaggi strutturati;
- voice call esterna;
- informazioni che non possono legalmente essere condivise.
L'engine deve supportare queste differenze.
---
# 34. MONETIZZAZIONE
Non è priorità immediata, ma l'architettura deve poter supportare:
- Episode 0 gratuito;
- singoli episodi premium;
- season pass;
- one payer / second player free;
- gifting;
- replay con altro partner;
- eventuali subscription future.
Non implementare ancora sistemi di pagamento.
Definisci soltanto i boundary corretti.
---
# 35. PRIVACY / DATA MINIMIZATION
Voglio il minimo possibile di dati personali.
Per l'MVP:
idealmente due guest player possono giocare senza account.
Non raccogliere:
- nomi reali obbligatori;
- relazioni reali;
- informazioni personali non necessarie.
Dobbiamo poter produrre statistiche e recap usando nickname o nomi inseriti volontariamente.
Prevedi:
- session expiration;
- deletion;
- anonymous analytics;
- data retention;
- consent.
---
# 36. OBSERVABILITY
Il sistema dovrà permetterci di capire:
- dove abbandonano;
- tempo per scelta;
- quali decisioni vengono scelte;
- quanto spesso i partner divergono;
- quali scene producono più engagement;
- quale ending ottengono;
- invitation conversion;
- second-player join rate;
- completion rate;
- replay-with-new-partner;
- share rate.
Progetta un analytics event taxonomy.
NON inviare l'intero world state a PostHog.
---
# 37. TESTING STRATEGY
Voglio almeno:
- unit tests per resolver;
- state transition tests;
- property-based testing dove utile;
- scenario validation;
- deterministic replay tests;
- regression playthrough;
- load tests successivamente;
- synthetic agent playtesting.
Per il resolver le proprietà matematiche/logiche sono più importanti dei test sull'output narrativo.
---
# 38. COSA NON VOGLIO CHE TU FACCIA ADESSO
Non:
- scegliere definitivamente il nome del gioco;
- progettare logo;
- progettare grafica;
- scrivere una storia completa;
- creare il primo episodio;
- generare immagini;
- costruire frontend finale;
- implementare pagamenti;
- scrivere centinaia di righe di codice;
- iniziare il repository senza prima aver progettato il sistema;
- introdurre microservizi inutili;
- aggiungere blockchain/NFT;
- complicare il sistema per moda tecnologica.
Prima:
# ARCHITECTURE.
---
# 39. COME DEVI LAVORARE IN QUESTA CHAT
Procedi come:
- software architect;
- game systems designer;
- narrative systems designer;
- database/domain modeller;
- red-team engineer.
Sii critico.
Quando propongo qualcosa:
- verifica se regge tecnicamente;
- trova edge case;
- trova failure mode;
- confronta alternative;
- evita overengineering;
- evita soluzioni fragili;
- usa pattern già validati quando appropriati;
- non introdurre tecnologia solo perché nuova.
Ricerca sul web quando serve per:
- verificare framework;
- standard;
- architetture;
- repository open-source;
- giochi comparabili;
- tool.
Preferisci:
- documentazione ufficiale;
- paper;
- repository originali;
- fonti primarie.
---
# 40. FASE 0 — ARCHITECTURE DISCOVERY
Questa è la prima fase che voglio affrontare.
NON scrivere ancora codice.
La Fase 0 deve produrre almeno:
## A. System boundaries
Cosa appartiene al core engine e cosa no.
## B. Domain map
Entità principali e bounded context.
## C. Architecture decision
Event Sourcing/CQRS/ECS/relational DB ecc.
## D. Data ownership
Chi possiede ogni tipo di dato.
## E. State model
Come rappresentiamo world, characters, relationship, knowledge.
## F. Action model
Come rappresentiamo semantic actions.
## G. Resolution model
Come le azioni vengono fuse.
## H. Temporal model
Come rappresentiamo tempo e simultaneità.
## I. Narrative model
Separazione simulation/director/realization.
## J. Memory model
Facts, knowledge, belief, memories, reflections.
## K. Versioning model
Engine/scenario/rules/prompts.
## L. Replay model
Come riproduciamo esattamente una sessione.
## M. Validation architecture
Quali validator esistono.
## N. Scenario authoring model
Come costruiamo episodi senza toccare engine code.
## O. Testing architecture
Unit, integration, property-based, synthetic playtesting.
## P. Infrastructure baseline
Frontend/backend/database/storage/deployment.
## Q. Repository architecture
Monorepo/package boundaries.
## R. Analytics taxonomy
Eventi importanti.
## S. Security/privacy baseline.
---
# 41. ADR — ARCHITECTURE DECISION RECORDS
Ogni decisione strutturale importante deve essere formalizzata in un ADR.
Per esempio:
```text
ADR-001 Event Sourcing scope
ADR-002 Database
ADR-003 World state representation
ADR-004 Semantic action model
ADR-005 Resolver determinism
ADR-006 LLM boundaries
ADR-007 Scenario definition format
ADR-008 Versioning
ADR-009 Realtime architecture
ADR-010 Guest session model
```
Ogni ADR:
```text
Context
Decision
Alternatives considered
Why rejected
Consequences
Risks
Revisit conditions
```
Non voglio decisioni strutturali sepolte dentro una conversazione.
---
# 42. PRIMA CONSEGNA CHE VOGLIO DA TE
Comincia facendo un'analisi completa della **Fase 0 — Architecture Discovery**.
In particolare:
1. confronta criticamente le architetture già esistenti che possiamo prendere come riferimento;
2. proponi i bounded context;
3. proponi il domain model iniziale;
4. proponi quali parti devono essere event-sourced;
5. proponi world-state architecture;
6. progetta la prima versione della semantic action model;
7. progetta il conflict/resolution model;
8. definisci il ruolo dell'LLM e i suoi limiti;
9. proponi il modello knowledge/fact/belief;
10. proponi temporal model;
11. proponi la strategia di versioning/replay;
12. proponi la repository architecture;
13. proponi lo stack tecnico motivandolo;
14. fai un red-team approfondito;
15. individua tutte le decisioni ancora premature.
Non iniziare ancora a sviluppare.
Alla fine produci:
# ARCHITECTURE BASELINE v0.1
che deve essere abbastanza precisa da poter essere criticata prima di congelare il design.
Poi la red-teameremo e passeremo a:
# ARCHITECTURE BASELINE v0.2
Solo dopo inizieremo:
schema concreto → repo → skeleton → test engine → primo micro-scenario tecnico.
---
# 43. CRITERIO DI SUCCESSO
La Fase 0 sarà riuscita quando potremo prendere scenari completamente differenti come:
- incidente aereo;
- indagine criminale;
- missione spaziale;
- relazione familiare;
- spedizione archeologica;
- crisi politica fictional;
- survival;
- corporate thriller;
e rappresentarli **senza cambiare il core engine**.
Cambieranno:
- configurazione;
- entities;
- roles;
- resources;
- action vocabulary specifica;
- rules specifiche;
- scenario content.
Non l'architettura fondamentale.
Allo stesso tempo, NON voglio creare un "universal simulator" teoricamente capace di simulare qualsiasi cosa ma impossibile da sviluppare.
Deve essere:
> **general enough for many narrative co-op scenarios, specific enough to actually ship.**
Comincia ora dalla **Architecture Discovery**, non dalla storia.
