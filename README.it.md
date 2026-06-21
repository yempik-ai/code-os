<div align="center">

<img src="docs/banner.png" alt="code-os — il sistema operativo dell'ingegnere per Claude Code" width="100%" />

<sub><a href="README.md">English</a> · <b>Italiano</b></sub>

# `code-os`

### Il sistema operativo dell'ingegnere per Claude Code.

Rendi affidabili gli agenti di coding sul lavoro vero — non un autocomplete furbo. Un insieme di **harness drop-in, reference solide e strumenti testati sul campo** per far girare agenti long-running su codebase di produzione: un harness di sicurezza + memoria per gli agenti, un second brain su Obsidian, skill curate, un reference stack vendor-neutral e le persone che vale la pena seguire.

<br>

![Built by yempik.](https://img.shields.io/badge/built%20by-yempik.-E35B2D?style=for-the-badge)
![For Claude Code](https://img.shields.io/badge/for-Claude%20Code-111827?style=for-the-badge)
![State of the art](https://img.shields.io/badge/SOTA-June%202026-2D6CDF?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-6B7280?style=for-the-badge)

<sub><b>In produzione, non in slide.</b> · by <a href="https://yempik.com"><b>yempik.</b></a> · mantenuto da <a href="https://www.linkedin.com/in/simone-bova/"><b>Simone Bova</b></a></sub>

</div>

---

## Due founder, due sistemi, un solo cappello

`code-os` è la **metà builder** di una coppia.

| | [`cowork-os`](https://github.com/yempik-ai/cowork-os) — [Raffaele Zarrelli](https://www.linkedin.com/in/raffaele-zarrelli/) | **`code-os`** — [Simone Bova](https://www.linkedin.com/in/simone-bova/) |
|:--|:--|:--|
| **Superficie** | Claude **Cowork** | Claude **Code** |
| **Per** | team business & ops che non scrivono codice | ingegneri che fanno girare agenti su codebase reali |
| **Idea di fondo** | l'AI come *memoria aziendale*, non una chat usa-e-getta | l'AI come *collaboratore long-running affidabile*, non un autocomplete |
| **Cosa ti porti via** | un workspace che ricorda le tue decisioni | un harness che tiene gli agenti onesti su lavoro che dura giorni |

Stesso problema — *usare l'AI come un sistema, non come una sessione* — risolto da due teste diverse. Stesso cappello: **yempik.**

---

## Perché esiste

Out of the box, un agente di coding fallisce in modi prevedibili sulle codebase reali: la **compaction del contesto distrugge lo stato**, i task multi-giorno **vanno alla deriva**, i comandi distruttivi **passano inosservati** e il lavoro viene spacciato come **fatto, senza verifica**. `code-os` risolve tutto questo con *disciplina basata su file* — memoria survival-kit, loop spec→plan→execute, hook di enforcement e reference solide — invece che con trucchi da prompt.

Serve due lettori contemporaneamente: l'**operatore** (la persona che usa Claude Code) e **Claude stesso** quando viene puntato su questo repo.

---

## Cosa c'è dentro — cinque moduli autonomi

> Ogni modulo sotto [`modules/`](modules/) sta in piedi da solo. Usane uno senza gli altri.

| Modulo | Cartella | In breve |
|:--|:--|:--|
| ⚙️ **Agentic harness** | [`modules/agentic-harness/`](modules/agentic-harness/index.md) | Drop-in stack-agnostic per qualsiasi repo git: file survival-kit, loop spec→plan→execute, ~5 hook di enforcement, un `CLAUDE.md` su misura. L'eroe. |
| 🧠 **Second brain** | [`modules/second-brain/`](modules/second-brain/index.md) | Knowledge vault su Obsidian guidato da Claude Code. Architettura a tre livelli, slash command, hook, sync cross-device. |
| 🔌 **Skills & plugin** | [`modules/skills-guide/`](modules/skills-guide/useful-skills.md) | Guida ragionata ai plugin / MCP / workflow che vale la pena installare — *cosa / perché / quando / per chi / come*, più una procedura di security audit. |
| 📚 **Tech references** | [`modules/tech-references/`](modules/tech-references/index.md) | Reference stack + un compendio di tool-orchestration verificato in modo avversariale (SOTA, giugno 2026) + pattern di produzione vendor-neutral — e un workflow a brief per trasformarli in un piano. |
| 🛠️ **AI tools & persone** | [`modules/ai-tools/`](modules/ai-tools/useful-ai-tools.md) · [`modules/personas/`](modules/personas/people-to-follow.md) | I tool AI-native che vale la pena guardare, e i professionisti che stanno plasmando l'agentic engineering. |

---

## Installazione

**Come plugin di Claude Code** — il core eseguibile (una skill sempre attiva + cinque command):

```bash
/plugin marketplace add yempik-ai/code-os
/plugin install code-os@code-os
```

**Come kit completo** — clona il repo, poi incolla [`INSTALL.md`](INSTALL.md) in Claude Code (ti chiede l'obiettivo e ti indirizza al modulo giusto) oppure segui [`GETTING_STARTED.md`](GETTING_STARTED.md) modulo per modulo. Cosa può configurarti ogni modulo è catalogato in [`capabilities.md`](capabilities.md).

---

## Scegli il tuo percorso

```text
"Voglio che Claude Code sia affidabile sulla mia codebase."   ← parti da qui
   → modules/agentic-harness/index.md → DAY-0-CHECKLIST.md → BOOTSTRAP.md
   ⏱  ~10min di preparazione (umano) + ~1–2h di setup supervisionato

"Voglio un knowledge vault personale."
   → modules/second-brain/index.md → DAY-0-CHECKLIST.md → BOOTSTRAP.md
   ⏱  ~1h di preparazione (umano) + ~2h di setup supervisionato

"Quali plugin vale davvero la pena installare?"
   → modules/skills-guide/useful-skills.md        (reference; installa à la carte)

"Con cosa lo costruisco / come devono orchestrare i tool gli agenti?"
   → modules/tech-references/index.md              (reference stack + workflow a brief)
```

---

## Struttura del repo

```text
code-os/
├── README.md                 ← versione inglese
├── README.it.md              ← sei qui
├── index.md                  ← mappa completa del repo (parti da qui per navigare)
├── INSTALL.md                ← incollalo in Claude Code: ti chiede l'obiettivo e ti indirizza
├── GETTING_STARTED.md        ← i due percorsi di setup (guidato / per modulo)
├── capabilities.md           ← cosa può configurarti ogni modulo
├── CONTRIBUTING.md           ← come contribuire
├── LICENSE                   ← MIT
├── .claude-plugin/           ← manifest del marketplace (installabile in Claude Code)
├── plugins/code-os/          ← il plugin: core skill sempre attiva + 5 slash command
├── docs/                     ← asset README / social (banner, social preview)
└── modules/
    ├── agentic-harness/      ← harness drop-in di memoria + sicurezza per qualsiasi repo
    ├── second-brain/         ← knowledge vault su Obsidian con Claude Code
    ├── tech-references/      ← reference stack + ricerca su orchestration + pattern
    ├── skills-guide/         ← plugin / MCP / workflow curati
    ├── ai-tools/             ← tool AI-native che vale la pena guardare
    └── personas/             ← persone che vale la pena seguire
```

---

## Note per gli agenti che leggono questo repo

- **Ogni modulo è autonomo.** Tratta lo spec del modulo rilevante come autorevole; non incrociare lo spec di un altro modulo a meno che non te lo chiedano.
- **I file `BOOTSTRAP.md` sono prompt autonomi** dentro i delimitatori `===== BEGIN PROMPT =====` — leggi i file referenziati dall'inizio alla fine prima di agire, poi esegui la checklist di setup.
- **I tool rinviati lo sono per un motivo.** Fai resistenza a chi li propone al Day 1.
- **Le reference sono solide** — ogni affermazione è legata a una fonte versionata o a una URL recuperata. Mantieni quella disciplina quando le estendi.

---

<div align="center">
<sub>Built by <a href="https://yempik.com"><b>yempik.</b></a> · <i>L'AI che gli altri ti lasciano in slide, noi te la mettiamo in produzione.</i> · Per la mappa completa, vedi <a href="index.md"><code>index.md</code></a></sub>
</div>
