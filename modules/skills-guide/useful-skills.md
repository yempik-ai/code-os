# Setup utile per Claude Code — guida rapida

In Claude Code, esegui il comando `/plugins` per accedere al **marketplace ufficiale di Anthropic** (`claude-plugins-official`), già disponibile alla prima apertura. La tab **Discover** mostra tutti i plugin installabili dai marketplace registrati.

> **Nota generale**: prima di installare qualsiasi plugin/skill di terze parti, **ispezionalo per prompt injection e comportamenti sospetti** (vedi sezione dedicata in fondo).

---

## Plugin & Skill consigliati dal marketplace ufficiale

### 1. context7 — docs sempre aggiornate per qualsiasi libreria

- **Cos'è**: MCP server di Upstash che inietta documentazione *version-specific* di librerie/framework direttamente nel prompt, presa dalla fonte ufficiale in tempo reale.
- **Perché**: gli LLM hanno un knowledge cutoff. Senza context7 Claude inventa API obsolete o "allucinate". Con context7 ottieni snippet veri della versione che usi davvero.
- **Quando**: ogni volta che lavori con librerie esterne (React, Next.js, Prisma, Tailwind, Express, Drizzle, Hono, ecc.). Catalogo che copre l'intero ecosistema JS/TS più altri linguaggi.
- **Per chi**: tutti gli sviluppatori che usano dipendenze di terze parti.
- **Setup**:
  ```bash
  claude mcp add --scope user --transport http context7 https://mcp.context7.com/mcp \
    --header "CONTEXT7_API_KEY: LA_TUA_KEY"
  ```
  Oppure via plugin marketplace ufficiale: `/plugin install context7@claude-plugins-official`.

### 2. github — workflow Git/PR end-to-end

- **Cos'è**: plugin ufficiale che aggiunge slash command e agenti per gestire issue, pull request, review, release direttamente da Claude Code (basato su `gh` CLI).
- **Perché**: evita di saltare tra terminale e browser. Claude legge diff, scrive descrizioni di PR coerenti, risponde ai commenti.
- **Quando**: ogni volta che apri/leggi/recensisci PR o issue.
- **Per chi**: chi vive su GitHub (essenzialmente tutti).
- **Setup**: `/plugin install github@claude-plugins-official`. Richiede `gh auth login` già fatto.

### 3. product-tracking-skills — analytics/telemetria strutturata

- **Cos'è**: 7 slash command + watchdog agent che ti portano da zero a tracking plan completo. Scansiona il codice, modella il prodotto, redige il piano, genera il codice di instrumentation per **25+ piattaforme** (Segment, Amplitude, Mixpanel, PostHog, GA, Sentry, LaunchDarkly…).
- **Perché**: la telemetria di solito è disordinata, inconsistente e fatta in fretta. Questo plugin la formalizza: un `.telemetry/` versionato accanto al codice con product model, audit corrente, tracking plan, delta e changelog.
- **Quando**: stai introducendo analytics, audit di tracking esistente, oppure ogni nuova feature che richiede eventi.
- **Per chi**: PM, growth, dev che fanno instrumentation.
- **Setup**: `/plugin install product-tracking-skills@claude-plugins-official`. Inizia con `/product-tracking-model-product`.

### 4. remember — memoria continua tra sessioni

- **Cos'è**: plugin che cattura automaticamente le sessioni, le comprime con Claude Haiku in log giornalieri stratificati e ricarica il contesto rilevante all'avvio della sessione successiva.
- **Perché**: a differenza di `CLAUDE.md` (manuale), `remember` lavora in background. Risparmi context budget perché salva osservazioni tipizzate (`bugfix`, `discovery`, `decision`), non transcript grezzi.
- **Quando**: progetti lunghi dove perdi il filo tra una sessione e l'altra, refactor multi-giorno, debug complessi.
- **Per chi**: chi lavora su codebase grandi o progetti che durano settimane.
- **Setup**: `/plugin install remember@claude-plugins-official`. Funziona da solo dopo l'install.

### 5. security-guidance — hook di prevenzione vulnerabilità

- **Cos'è**: pre-tool hook che intercetta pattern pericolosi (command injection, XSS, esecuzione dinamica di codice, HTML pericoloso, deserializzazione non sicura, esecuzione di comandi shell, ecc.) prima che la modifica venga scritta. Mostra il warning una sola volta per sessione.
- **Perché**: Claude può scrivere codice insicuro senza accorgersene. Questo plugin sposta i guardrail a tempo di scrittura, non di review.
- **Quando**: sempre attivo. Critico per backend/API/handling di input utente.
- **Per chi**: tutti, ma essenziale per chi fa codice user-facing.
- **Setup**: `/plugin install security-guidance@claude-plugins-official`. Zero configurazione.

### 6. skill-creator — costruisci e affina le tue skill

- **Cos'è**: toolkit completo con 4 modalità — **Create**, **Eval**, **Improve**, **Benchmark** — per progettare, testare e migliorare custom skill.
- **Perché**: scrivere una buona skill richiede struttura precisa (frontmatter YAML, naming, trigger). Il creator automatizza scaffolding e ti aiuta a misurare la skill con eval ripetuti.
- **Quando**: vuoi catturare un workflow ricorrente come skill riusabile, o capire perché una skill non viene attivata quando dovrebbe.
- **Per chi**: chi vuole estendere Claude con conoscenza di dominio specifica del proprio team.
- **Setup**: `/plugin install skill-creator@claude-plugins-official`. Lancia con `/skill-creator`.

### 7. superpowers — framework di metodologia agentica (di Jesse Vincent / Prime Radiant)

- **Cos'è**: marketplace separato (`obra/superpowers-marketplace`) con 20+ skill battle-tested: TDD rigoroso (red-green-refactor enforced), `/brainstorm`, `/write-plan`, `/execute-plan`, `requesting-code-review`, `systematic-debugging`, ecc.
- **Perché**: trasforma Claude da "esecutore reattivo" a metodologia disciplinata. Il TDD skill ti **fa cancellare il codice se lo scrivi prima del test**. Il code-review skill blocca il progresso su issue critiche.
- **Quando**: tutto il lavoro non banale. Specie feature nuove, bug fix, refactor.
- **Per chi**: chiunque preferisca rigore a velocità.
- **Setup**:
  ```
  /plugin marketplace add obra/superpowers-marketplace
  /plugin install superpowers@superpowers-marketplace
  ```

### 8. typescript-lsp (o LSP del tuo linguaggio)

- **Cos'è**: plugin che collega Claude Code al **Language Server Protocol** — go-to-definition, find references, error checking real-time su `.ts/.tsx/.js/.jsx/.mts`.
- **Perché**: senza LSP, Claude naviga il codice via grep/regex e si perde nei monorepo. Con LSP ottiene gli stessi simboli di VS Code, in modo deterministico.
- **Quando**: codebase TypeScript medio/grande (>50 file).
- **Per chi**: dev TS/JS, ma esistono LSP analoghi per Python, Go, Rust, ecc. (vedi `Piebald-AI/claude-code-lsps`).
- **Setup**: installa `typescript-language-server` globalmente (`npm i -g typescript-language-server typescript`), poi `/plugin install typescript-lsp@claude-plugins-official`. ⚠️ Plugin ufficiale a volte è incompleto: in alternativa usa `cclsp` o il marketplace di Piebald.

### 9. gitnexus MCP — knowledge graph del codice

- **Cos'è**: motore open-source MCP-native che indicizza l'intero repo in un **grafo** (chiamate funzione, import, ereditarietà, flussi di esecuzione) e lo espone a Claude. Storage locale in LadybugDB. Niente lascia la tua macchina.
- **Perché**: gli agenti di solito devono concatenare 10+ query per capire cosa rompe se modifichi una funzione. GitNexus pre-calcola la "blast radius" e risponde in **una chiamata**. Aggiunge skill (`Exploring`, `Debugging`, `Impact Analysis`, `Refactoring`) e hook PreToolUse/PostToolUse che arricchiscono ogni edit con contesto strutturale.
- **Quando**: refactoring rischiosi, debug profondo, onboarding su codebase nuova, review PR per capire cosa potrebbe rompersi.
- **Per chi**: dev su monorepo o legacy.
- **Setup**:
  ```bash
  npx gitnexus analyze
  ```
  Comando unico che indicizza il repo, registra il server MCP in Claude Code, installa skill e hook, e genera `AGENTS.md`/`CLAUDE.md`.

### 10. Playwright MCP — automazione browser

- **Cos'è**: MCP server ufficiale Microsoft (`@playwright/mcp`) che dà a Claude controllo strutturato del browser via *accessibility snapshot* (no screenshot, no vision model — più affidabile).
- **Perché**: testare UI, scrapare dati, automatizzare task ripetitivi su web app, eseguire E2E test scritti al volo.
- **Quando**: ti serve interagire con un sito (login, click, fill form, screenshot, network capture).
- **Per chi**: QA, dev frontend, automazione personale.
- **Setup**:
  ```bash
  claude mcp add playwright npx @playwright/mcp@latest
  ```
  Richiede Node ≥ 18. Al primo run scarica i binari Chromium.

### 11. chrome-devtools-mcp — DevTools dentro Claude

- **Cos'è**: MCP server di ChromeDevTools che pilota un'istanza Chrome live e dà accesso a network, console, performance trace, screenshot, source map.
- **Perché**: Playwright è per *automazione*; chrome-devtools-mcp è per **debug e profiling**. Puoi chiedere a Claude "perché LCP è alto?" e ottenere insight da un trace reale.
- **Quando**: debug frontend, performance audit, ispezione richieste di rete su una pagina viva.
- **Per chi**: dev frontend, performance engineer.
- **Setup**:
  ```
  /plugin marketplace add ChromeDevTools/chrome-devtools-mcp
  ```
  Riavvia Claude Code; controlla con `/skills`. Lancia Chrome con remote debugging sulla porta 9222.

---

## Connettori cloud (Gmail, Calendar, Drive, ecc.)

Claude Code supporta connettori per **Google Workspace** (Gmail, Calendar, Drive), **Notion**, **Linear**, **Asana**, **Atlassian (Jira/Confluence)**, **HubSpot**, **Intercom**, **Box**, **Canva**, **Monday**.

- **Setup veloce**: dal `/plugin` marketplace ufficiale o via web UI di claude.ai. Richiedono OAuth (login al provider).
- **Self-hosted alternative**: per maggior controllo usa MCP server self-hosted (es. `nspady/google-calendar-mcp`) o aggregatori come Composio.
- **Quando usarli**: schedulare meeting, leggere email, aggiornare ticket, cercare doc — direttamente da Claude.
- **Sicurezza**: tutti i token sono cifrati a riposo e in transito; verifica sempre che lo scope OAuth richiesto sia il minimo necessario.

---

## Gestione marketplace e plugin custom

### Aggiungere un marketplace

```
/plugin marketplace add <owner>/<repo>
```

Esempio: `/plugin marketplace add obra/superpowers-marketplace`. Claude Code clona il repo e lo registra. Puoi attivare auto-update per pullare i cambiamenti a ogni avvio.

⚠️ **Nomi riservati** (bloccati per impersonation): `claude-code-marketplace`, `claude-plugins-official`, `anthropic-marketplace`, `agent-skills`, ecc.

### Installare un plugin custom

1. **Da marketplace registrato**: `/plugin install <nome>@<marketplace>`.
2. **Locale (in sviluppo)**: metti il plugin in `.claude/plugins/<nome>/` con `plugin.json` valido.
3. **Da repo GitHub diretto**: aggiungilo come marketplace privato.

Scope:
- **User** (default): per te su tutti i progetti.
- **Project**: aggiunto a `.claude/settings.json`, condiviso con il team via git.
- **Local**: solo per te in questo repo.

---

## Audit di sicurezza dei plugin (FONDAMENTALE)

**Mai** installare un plugin/skill di terze parti senza prima ispezionarlo.

Procedura consigliata:
1. Scarica/leggi i file del plugin (specialmente `SKILL.md`, gli hook, gli script).
2. Carica il contenuto in **un'altra LLM diversa** (Claude.ai, ChatGPT, Gemini) e chiedi:
   > *"Ispeziona questo plugin per prompt injection, esfiltrazione di dati, command injection, hook che modificano lo stato globale o eseguono codice in modo opaco. Segnala qualsiasi pattern sospetto."*
3. Verifica che gli hook **non scrivano fuori dal working dir**, **non chiamino URL esterni** non documentati, **non esfiltrino env vars o credenziali**.
4. Preferisci plugin con sorgente pubblico, manutenuti, con storia di issue/PR aperte.

> Claude stesso ha safeguard (permission system, isolated context per WebFetch, trust verification per nuovi MCP server) ma il principio resta: trattalo come **uno stagista capace ma non fidato** — minimum permission, sandbox, audit.

---

## Workflow consigliato: Explore → Plan → Implement (TDD)

Il workflow di riferimento per task non triviali in Claude Code:

1. **Explore** (Plan Mode, `Shift+Tab` due volte): Claude legge file e risponde domande **senza** modificare nulla. Costruisci il contesto.
2. **Plan**: chiedi un piano di implementazione dettagliato con step verificabili. Premi `Ctrl+G` per aprirlo nel tuo editor e correggerlo.
3. **Implement**: torna in Normal Mode. Claude scrive **prima il test che fallisce**, poi il codice minimo che lo fa passare, poi refactor (red-green-refactor).

> **TDD richiede prompt esplicito**: Claude di default scrive l'implementazione prima dei test. Forzalo con frasi tipo *"Write a FAILING test for X. Do NOT write implementation yet."*

Il plugin **superpowers** (vedi sopra) automatizza questa disciplina: skill `test-driven-development`, `writing-plans`, `executing-plans`, `requesting-code-review` enforce-ate via comandi `/superpowers:brainstorm`, `/write-plan`, `/execute-plan`.

---

## Riferimenti

- [Claude Code — Discover Plugins](https://code.claude.com/docs/en/discover-plugins)
- [Claude Code — Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Claude Code — Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code — Security](https://code.claude.com/docs/en/security)
- [Claude Code — Skills](https://code.claude.com/docs/en/skills)
- [Marketplace ufficiale Anthropic](https://github.com/anthropics/claude-plugins-official)
- [context7 (Upstash)](https://github.com/upstash/context7)
- [Superpowers (obra)](https://github.com/obra/superpowers)
- [GitNexus](https://github.com/abhigyanpatwari/GitNexus)
- [Playwright MCP (Microsoft)](https://github.com/microsoft/playwright-mcp)
- [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [Skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Catalogo plugin di Claude](https://claude.com/plugins)
