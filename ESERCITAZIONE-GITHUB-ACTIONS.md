# Esercitazione: Github Actions per Build e Deploy

## 🎯 Obiettivi dell'Esercitazione

L'obiettivo principale di questa esercitazione è imparare a creare e configurare **Github Actions** per automatizzare il processo di CI/CD della vostra applicazione React.

**Focus principale**: comprendere il funzionamento delle Github Actions, non il deploy!

### Cosa imparerete:
- ✅ Struttura di un workflow Github Actions
- ✅ Le fasi fondamentali di una pipeline: checkout, build, lint, test
- ✅ Gestione delle variabili d'ambiente e dei secrets
- ✅ Esecuzione di comandi npm all'interno delle Actions
- ✅ Utilizzo di actions esterne (marketplace)
- ✅ Integrazione con servizi di deploy (Vercel come esempio)

---

## 📚 Prerequisiti

- Repository Github con l'applicazione React+Vite già creata
- Conoscenza base di Git e Github
- Familiarità con npm e comandi di build

---

## 📝 Parte 1: Creare il Primo Workflow

### Step 1.1: Creare la struttura delle cartelle

Nel vostro repository, create la seguente struttura:

```
.github/
  └── workflows/
      └── ci.yml
```

### Step 1.2: Workflow Base - Checkout e Setup

Create il file `.github/workflows/ci.yml` con il seguente contenuto base:

```yaml
name: CI Pipeline

# Quando deve essere eseguito il workflow?
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
      # FASE 1: CHECKOUT del codice
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # FASE 2: Setup Node.js
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
```

**🤔 Domande di riflessione:**
- Cosa fa l'action `actions/checkout@v4`?
- Perché è importante specificare la versione di Node.js?
- Cosa succede se fate push su un branch diverso da `main`?

---

## 📝 Parte 2: Gestione delle Dipendenze e Cache

### Step 2.1: Installare le dipendenze con pnpm

Aggiungete questi step dopo il setup di Node.js:

```yaml
      # Setup pnpm
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 9
      
      # Cache delle dipendenze per velocizzare le build successive
      - name: Get pnpm store directory
        id: pnpm-cache
        shell: bash
        run: |
          echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT
      
      - name: Setup pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-store-
      
      # Installazione dipendenze
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
```

**🤔 Domande di riflessione:**
- Perché è utile la cache delle dipendenze?
- Cosa significa `--frozen-lockfile`?
- Come funziona il meccanismo della cache key?

---

## 📝 Parte 3: Lint e Build

### Step 3.1: Aggiungere il Linting

```yaml
      # FASE 3: LINT - Controllo qualità del codice
      - name: Run ESLint
        run: pnpm run lint
```

### Step 3.2: Aggiungere la Build

```yaml
      # FASE 4: BUILD - Compilazione dell'applicazione
      - name: Build application
        run: pnpm run build
```

**🤔 Domande di riflessione:**
- Cosa succede se il lint fallisce?
- Dove vengono salvati i file di build?
- Come possiamo verificare che la build sia andata a buon fine?

---

## 📝 Parte 4: Variabili d'Ambiente e Secrets

### Step 4.1: Creare Secrets su Github

1. Andate su **Settings** → **Secrets and variables** → **Actions**
2. Create un nuovo **Repository Secret** chiamato `VITE_API_URL` con valore `https://api.example.com`
3. Create un altro secret chiamato `VITE_APP_TITLE` con valore `My React App`

### Step 4.2: Usare le Variabili d'Ambiente nella Build

Modificate lo step di build per includere le variabili d'ambiente:

```yaml
      - name: Build application
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
          VITE_APP_TITLE: ${{ secrets.VITE_APP_TITLE }}
        run: pnpm run build
```

### Step 4.3: Creare un file .env di esempio

Nel vostro progetto, create un file `.env.example`:

```
VITE_API_URL=https://api.example.com
VITE_APP_TITLE=My React App
```

**🤔 Domande di riflessione:**
- Qual è la differenza tra Secrets e Variables?
- Perché le variabili Vite devono iniziare con `VITE_`?
- Come accedete a queste variabili nel codice React?

**💡 Esercizio pratico:**
- Modificate `App.tsx` per mostrare il valore di `VITE_APP_TITLE`
- Verificate che nella build la variabile venga correttamente sostituita

---

## 📝 Parte 5: Upload degli Artifacts

### Step 5.1: Salvare gli Artifacts della Build

Aggiungete questo step dopo la build:

```yaml
      # Salvare gli artifacts per uso futuro
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: dist/
          retention-days: 7
```

**🤔 Domande di riflessione:**
- A cosa servono gli artifacts?
- Dove potete scaricare gli artifacts di una run?
- Perché abbiamo impostato `retention-days` a 7?

---

## 📝 Parte 6: Matrice di Build (Opzionale - Avanzato)

### Step 6.1: Testare su Multiple Versioni di Node

Modificate il vostro workflow per testare su più versioni di Node:

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18, 20, 22]
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      
      # ... resto degli step
```

**🤔 Domande di riflessione:**
- Quante volte viene eseguito il workflow con questa configurazione?
- Come potete vedere i risultati di ogni versione?

---

## 📝 Parte 7: Deploy su Vercel (Bonus)

### Step 7.1: Creare un Job Separato per il Deploy

Aggiungete un nuovo job **dopo** il job di build:

```yaml
  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

### Step 7.2: Ottenere i Token Vercel

1. Andate su [Vercel](https://vercel.com)
2. Create un nuovo progetto collegato al vostro repository
3. Ottenete il **Vercel Token** da Settings → Tokens
4. Ottenete **Org ID** e **Project ID** dal file `.vercel/project.json` (dopo aver fatto il primo deploy manuale)
5. Aggiungete questi valori come Secrets su Github

**🔔 Nota Importante:**
Il deploy è solo un bonus! L'importante è aver capito come funzionano le Actions, le variabili d'ambiente, e l'esecuzione di comandi.

---

## 📝 Parte 8: Comandi Personalizzati e Script

### Step 8.1: Eseguire Comandi Shell nella Pipeline

Aggiungete uno step personalizzato per copiare file o eseguire operazioni:

```yaml
      - name: Prepare deployment files
        run: |
          echo "Creating deployment directory..."
          mkdir -p deployment
          cp -r dist/* deployment/
          echo "Build completed at $(date)" > deployment/build-info.txt
          ls -la deployment/
      
      - name: Display build information
        run: |
          echo "=== Build Information ==="
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Author: ${{ github.actor }}"
          cat deployment/build-info.txt
```

### Step 8.2: Usare Script npm Personalizzati

Aggiungete al vostro `package.json`:

```json
"scripts": {
  "prebuild": "echo 'Starting build process...'",
  "postbuild": "echo 'Build completed! Checking output...' && ls -la dist",
  "build": "tsc -b && vite build",
  "build:stats": "vite build --mode production && du -sh dist/"
}
```

Poi nella Action:

```yaml
      - name: Build with statistics
        run: pnpm run build:stats
```

**🤔 Domande di riflessione:**
- Qual è la differenza tra eseguire comandi shell e script npm?
- Quando è meglio usare uno script npm vs un comando diretto?

---

## 🎯 Workflow Completo - Esempio Finale

Ecco un esempio di workflow completo che mette insieme tutti i pezzi:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: '20'

jobs:
  lint-and-build:
    name: Lint and Build
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 9
      
      - name: Get pnpm store directory
        id: pnpm-cache
        shell: bash
        run: |
          echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT
      
      - name: Setup pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-store-
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Run linter
        run: pnpm run lint
      
      - name: Build application
        env:
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
          VITE_APP_TITLE: ${{ secrets.VITE_APP_TITLE }}
        run: pnpm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: production-files
          path: dist/
          retention-days: 7

  deploy:
    name: Deploy to Vercel
    needs: lint-and-build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: production-files
          path: dist/
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

---

## ✅ Checklist Completamento

- [ ] Ho creato la cartella `.github/workflows/`
- [ ] Ho creato un workflow base con checkout e setup
- [ ] Ho aggiunto il caching delle dipendenze
- [ ] Ho configurato il linting nella pipeline
- [ ] Ho configurato la build nella pipeline
- [ ] Ho creato e usato dei Secrets per le variabili d'ambiente
- [ ] Ho aggiunto l'upload degli artifacts
- [ ] Ho eseguito comandi shell personalizzati nella pipeline
- [ ] Ho integrato il deploy (opzionale)
- [ ] Ho testato il workflow con un push su Github

---

## 🧪 Esercizi di Verifica

### Esercizio 1: Badge di Status
Aggiungete un badge nel `README.md` che mostri lo stato del workflow:

```markdown
![CI Pipeline](https://github.com/USERNAME/REPO/workflows/CI%20Pipeline/badge.svg)
```

### Esercizio 2: Fallimento Intenzionale
- Introducete un errore di linting nel codice
- Fate push e osservate come il workflow fallisce
- Correggete l'errore e verificate che il workflow diventi verde

### Esercizio 3: Variabili Multiple
- Aggiungete 3 nuove variabili d'ambiente: `VITE_VERSION`, `VITE_ENV`, `VITE_DEBUG`
- Usatele nella build
- Mostratele nell'interfaccia dell'app

### Esercizio 4: Script Personalizzato
- Create uno script npm che genera un file `build-manifest.json` con informazioni sulla build
- Eseguitelo nella pipeline
- Salvate il file negli artifacts

---

## 📖 Risorse Utili

- [Documentazione Github Actions](https://docs.github.com/en/actions)
- [Documentazione Vite - Env Variables](https://vitejs.dev/guide/env-and-mode.html)
- [Github Actions Marketplace](https://github.com/marketplace?type=actions)
- [Vercel CLI Documentation](https://vercel.com/docs/cli)

---

## 💡 Suggerimenti Finali

1. **Iniziate semplice**: Create prima un workflow base, poi aggiungete complessità
2. **Testate localmente**: Molti comandi possono essere testati in locale prima di metterli nella pipeline
3. **Leggete i log**: I log delle Actions sono molto dettagliati e aiutano a capire cosa sta succedendo
4. **Usate le variabili**: Evitate di hardcodare valori, usate secrets e variabili d'ambiente
5. **Commentate il codice YAML**: Rendete chiaro cosa fa ogni step

---

## 🎓 Cosa Avete Imparato

Al termine di questa esercitazione dovreste essere in grado di:
- ✅ Creare e configurare un workflow Github Actions
- ✅ Comprendere le fasi di una pipeline CI/CD
- ✅ Gestire secrets e variabili d'ambiente
- ✅ Eseguire comandi npm e shell nelle Actions
- ✅ Usare actions del marketplace
- ✅ Debuggare problemi nella pipeline
- ✅ Ottimizzare le performance con la cache
- ✅ Integrare servizi esterni (come Vercel)

**Ricordate**: Il deploy è solo la ciliegina sulla torta. Il vero valore è nell'automazione dei controlli di qualità (lint, test) e nella build automatica!

---

## 📚 Appendice: Risposte alle Domande di Riflessione

### Parte 1: Creare il Primo Workflow

**Q: Cosa fa l'action `actions/checkout@v4`?**

L'action `actions/checkout@v4` è una delle actions più importanti e fondamentali in ogni workflow. Si occupa di:
- Clonare il repository nel runner (la macchina virtuale che esegue il workflow)
- Fare checkout del branch o del commit specifico che ha triggerato il workflow
- Configurare git in modo che altri step possano usarlo se necessario
- Scaricare l'intera cronologia del repository (o solo il commit corrente, in base alla configurazione)

Senza questa action, il runner sarebbe un ambiente vuoto e non avrebbe accesso al vostro codice!

**Q: Perché è importante specificare la versione di Node.js?**

Specificare la versione di Node.js è cruciale per diversi motivi:
1. **Consistenza**: Garantisce che la build venga eseguita sempre con la stessa versione di Node, evitando comportamenti diversi tra esecuzioni
2. **Compatibilità**: Alcune dipendenze potrebbero non funzionare con versioni più vecchie o più nuove di Node
3. **Riproducibilità**: Permette agli altri sviluppatori e agli ambienti di produzione di usare la stessa versione
4. **Debugging**: Se qualcosa va storto, sapete esattamente quale versione di Node era in uso

È una best practice allineare la versione specificata qui con quella usata in sviluppo locale e in produzione.

**Q: Cosa succede se fate push su un branch diverso da `main`?**

Con la configurazione mostrata:
```yaml
on:
  push:
    branches: [ main ]
```

Se fate push su un branch diverso da `main` (ad esempio `develop` o `feature/nuova-funzionalità`), il workflow **NON verrà eseguito**. 

Per far sì che il workflow si esegua anche su altri branch, dovreste:
- Aggiungere altri branch all'array: `branches: [ main, develop, staging ]`
- Oppure usare pattern: `branches: [ main, 'feature/**' ]`
- Oppure rimuovere del tutto il filtro per eseguire su tutti i branch

---

### Parte 2: Gestione delle Dipendenze e Cache

**Q: Perché è utile la cache delle dipendenze?**

La cache delle dipendenze è fondamentale per:
1. **Velocità**: Invece di scaricare tutte le dipendenze ad ogni esecuzione (può richiedere 1-3 minuti), le dipendenze vengono recuperate dalla cache in pochi secondi
2. **Costi**: Meno tempo di esecuzione = meno minuti di GitHub Actions consumati
3. **Affidabilità**: Se npm registry o altri servizi hanno problemi, la cache può salvare la situazione
4. **Efficienza di rete**: Riduce il carico sui server dei package manager

La differenza può essere drammatica: da 2-3 minuti a 10-15 secondi per l'installazione!

**Q: Cosa significa `--frozen-lockfile`?**

Il flag `--frozen-lockfile` dice a pnpm di:
- **NON modificare** il file `pnpm-lock.yaml` durante l'installazione
- **Fallire** se le dipendenze nel `package.json` non corrispondono a quelle nel lockfile
- **Garantire** che vengano installate esattamente le stesse versioni specificate nel lockfile

Questo è importante nelle CI/CD perché:
- Garantisce riproducibilità: stessa build, stesse dipendenze
- Previene sorprese: nessuna installazione di versioni diverse da quelle testate
- Rileva problemi: se qualcuno ha modificato `package.json` senza aggiornare il lockfile, il workflow fallisce

In npm è equivalente a `npm ci`, in yarn è `yarn install --frozen-lockfile`.

**Q: Come funziona il meccanismo della cache key?**

La cache key funziona così:
```yaml
key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
```

1. **Composizione della key**:
   - `runner.os`: Sistema operativo (es. `Linux`)
   - `pnpm-store`: Identificatore della cache
   - `hashFiles('**/pnpm-lock.yaml')`: Hash SHA-256 del contenuto del lockfile

2. **Esempio di key**: `Linux-pnpm-store-a3f5d8c9b2e1...`

3. **Come funziona**:
   - Se la key esiste → la cache viene restaurata (hit)
   - Se la key NON esiste → vengono provate le `restore-keys` (partial hit)
   - Se nessuna match → no cache, installazione completa (miss)

4. **Quando la cache viene invalidata**:
   - Quando modificate `pnpm-lock.yaml` (aggiungendo/aggiornando dipendenze)
   - L'hash cambia → nuova key → la cache viene rigenerata

È un sistema brillante perché la cache si aggiorna automaticamente solo quando necessario!

---

### Parte 3: Lint e Build

**Q: Cosa succede se il lint fallisce?**

Se il comando `pnpm run lint` esce con un codice di errore diverso da 0 (cioè trova errori di linting):
1. **Lo step fallisce** e viene marcato con una ❌ rossa
2. **Gli step successivi NON vengono eseguiti** (inclusa la build e il deploy)
3. **Il workflow viene marcato come fallito**
4. **GitHub invia una notifica** (se configurata)
5. **Le pull request mostrano il fallimento** e possono essere bloccate (se avete branch protection rules)

Questo è il comportamento desiderato! Il lint è un quality gate: se il codice non rispetta gli standard, non dovrebbe andare oltre.

**Q: Dove vengono salvati i file di build?**

Per un progetto Vite, i file di build vengono salvati nella cartella `dist/` (configurabile nel `vite.config.ts`).

Importante notare:
- La cartella `dist/` viene creata **nel filesystem del runner**, non nel vostro repository
- Alla fine del workflow, il runner viene distrutto e tutto viene cancellato
- Se volete conservare i file di build, dovete usare gli **artifacts** (come mostrato nella Parte 5)
- I file di build NON vengono committati automaticamente nel repository

**Q: Come possiamo verificare che la build sia andata a buon fine?**

Ci sono diversi modi:

1. **Controllo del codice di uscita**: Se `vite build` completa con successo (exit code 0), la build è ok
2. **Verificare che esistano i file**: Aggiungere uno step:
   ```yaml
   - name: Verify build output
     run: |
       ls -la dist/
       test -f dist/index.html || exit 1
   ```
3. **Controllare la dimensione**: 
   ```yaml
   - name: Check build size
     run: du -sh dist/
   ```
4. **Analizzare i log**: Vite mostra statistiche sulla build (file generati, dimensioni, ecc.)
5. **Download artifacts**: Scaricare l'artifact e ispezionare manualmente

---

### Parte 4: Variabili d'Ambiente e Secrets

**Q: Qual è la differenza tra Secrets e Variables?**

| Aspetto | Secrets | Variables |
|---------|---------|-----------|
| **Visibilità** | Sempre oscurati nei log (`***`) | Visibili nei log |
| **Uso** | Dati sensibili (token, password, chiavi API) | Configurazioni non sensibili |
| **Accesso** | `${{ secrets.NOME }}` | `${{ vars.NOME }}` |
| **Crittografia** | Crittografati a riposo | Non crittografati |
| **Esempi** | API keys, certificati, password database | URL pubblici, flags di configurazione |

**Regola d'oro**: Se non volete che appaia nei log o se è sensibile, usate un Secret. Altrimenti, una Variable.

**Q: Perché le variabili Vite devono iniziare con `VITE_`?**

Vite ha questa restrizione per motivi di **sicurezza**:
1. **Prevenire leak accidentali**: Non tutte le variabili d'ambiente dovrebbero essere esposte al client
2. **Opt-in esplicito**: Dovete deliberatamente prefissare con `VITE_` le variabili che volete esporre
3. **Protezione**: Variabili come `DATABASE_PASSWORD` o `SECRET_KEY` non verranno mai iniettate nel bundle

Durante la build, Vite:
- Scansiona il codice per riferimenti a `import.meta.env.VITE_*`
- Sostituisce questi riferimenti con i valori effettivi
- Ignora completamente le altre variabili d'ambiente

**Q: Come accedete a queste variabili nel codice React?**

Nel codice React/TypeScript, accedete alle variabili così:

```typescript
// ✅ Corretto
const apiUrl = import.meta.env.VITE_API_URL;
const appTitle = import.meta.env.VITE_APP_TITLE;

// ❌ Sbagliato (non funziona con Vite)
const apiUrl = process.env.VITE_API_URL; // Questo è Node.js style

// Esempio in un componente
const App = () => {
  return (
    <div>
      <h1>{import.meta.env.VITE_APP_TITLE}</h1>
      <p>API: {import.meta.env.VITE_API_URL}</p>
    </div>
  );
};
```

Per il TypeScript, potreste voler aggiungere i tipi in `vite-env.d.ts`:
```typescript
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}
```

---

### Parte 5: Upload degli Artifacts

**Q: A cosa servono gli artifacts?**

Gli artifacts servono a:
1. **Preservare output**: Salvare file generati durante il workflow (build, report, log)
2. **Condividere tra job**: Passare file da un job a un altro (es. build → deploy)
3. **Download manuale**: Permettere agli sviluppatori di scaricare i file dalla UI di GitHub
4. **Debugging**: Ispezionare l'output di una build per capire problemi
5. **Distribuzione**: In alcuni casi, distribuire gli artifacts come release

Esempi comuni:
- File di build (`dist/`)
- Report di test
- Coverage report
- Log files
- Screenshot di test E2E

**Q: Dove potete scaricare gli artifacts di una run?**

Gli artifacts sono scaricabili in diversi modi:

1. **GitHub UI**:
   - Andate su Actions → selezionate il workflow run
   - Nella pagina del run, scrollate fino alla sezione "Artifacts"
   - Cliccate sul nome dell'artifact per scaricarlo (viene scaricato come ZIP)

2. **GitHub CLI**:
   ```bash
   gh run download <run-id>
   ```

3. **API di GitHub**:
   ```bash
   curl -H "Authorization: token $GITHUB_TOKEN" \
     https://api.github.com/repos/OWNER/REPO/actions/runs/RUN_ID/artifacts
   ```

4. **In altri job** (usando `actions/download-artifact`):
   ```yaml
   - uses: actions/download-artifact@v4
     with:
       name: dist-files
   ```

**Q: Perché abbiamo impostato `retention-days` a 7?**

Il parametro `retention-days: 7` specifica per quanti giorni GitHub conserverà gli artifacts prima di cancellarli automaticamente.

Motivi per limitare la retention:
1. **Costi**: Gli artifacts consumano storage, che ha un costo (anche con piani free)
2. **Limiti**: GitHub ha limiti di storage (es. 500MB per repository nei piani free)
3. **Pulizia**: Artifacts vecchi raramente servono
4. **Best practice**: Per produzione, usate proper release mechanisms, non artifacts

Linee guida:
- **1-3 giorni**: Per branch feature/PR temporanei
- **7 giorni**: Buon compromesso per la maggior parte dei casi
- **30 giorni**: Per build importanti o release candidate
- **90 giorni**: Massimo consentito da GitHub

---

### Parte 6: Matrice di Build

**Q: Quante volte viene eseguito il workflow con questa configurazione?**

Con la matrice `node-version: [18, 20, 22]`, il workflow viene eseguito **3 volte**:
- Una volta con Node.js 18
- Una volta con Node.js 20
- Una volta con Node.js 22

Ogni esecuzione è **parallela** e **indipendente**. Se aveste anche una matrice per OS:
```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
    os: [ubuntu-latest, windows-latest, macos-latest]
```

Il workflow verrebbe eseguito **9 volte** (3 versioni × 3 OS = 9 combinazioni)!

**Q: Come potete vedere i risultati di ogni versione?**

Nella UI di GitHub Actions:
1. Andate alla pagina del workflow run
2. Vedrete una sezione con il job name seguito da una matrice
3. Ogni combinazione ha:
   - Un indicatore di status (✅, ❌, 🟡)
   - Il nome della combinazione (es. "Node 18", "Node 20")
   - Un link per vedere i log specifici

Potete:
- Cliccare su ogni variante per vedere i log dettagliati
- Vedere immediatamente quali combinazioni falliscono
- Ri-eseguire solo le combinazioni fallite

Esempio di come appare:
```
build-and-test (18) ✅
build-and-test (20) ✅
build-and-test (22) ❌
```

---

### Parte 8: Comandi Personalizzati e Script

**Q: Qual è la differenza tra eseguire comandi shell e script npm?**

| Aspetto | Comandi Shell | Script npm |
|---------|---------------|------------|
| **Dove sono definiti** | Direttamente nel workflow YAML | Nel `package.json` |
| **Portabilità** | Dipendono dalla shell (bash, sh) | Funzionano ovunque ci sia npm |
| **Riutilizzo** | Solo nel workflow | Usabili anche in locale |
| **Complessità** | Semplici comandi one-liner | Possono orchestrare più comandi |
| **Manutenibilità** | Logica nel workflow | Logica centralizzata nel progetto |

**Esempi**:

Comandi shell:
```yaml
- run: ls -la dist/
- run: echo "Hello World"
- run: cp -r src/ backup/
```

Script npm:
```yaml
- run: pnpm run build
- run: pnpm run test
- run: pnpm run deploy
```

**Q: Quando è meglio usare uno script npm vs un comando diretto?**

**Usate script npm quando**:
1. Il comando è **complesso** e coinvolge più step
2. Volete **riutilizzare** la stessa logica in locale
3. La logica è **specifica del progetto** (build, test, deploy)
4. Volete **astrarre** i dettagli implementativi
5. Il comando deve funzionare su **diversi OS** (npm gestisce le differenze)

Esempio: `pnpm run build` invece di `tsc -b && vite build`

**Usate comandi shell diretti quando**:
1. Sono operazioni **semplici** e generiche (ls, cat, echo)
2. Sono specifiche della **pipeline CI/CD** (non servono in locale)
3. Manipolate **file/directory** temporanei
4. Volete **debugging** rapido senza modificare `package.json`
5. L'operazione è **una tantum** per quel workflow

Esempio: `echo "Build started at $(date)" > build.log`

**Best Practice**:
```yaml
# ✅ Buono - usa script npm per logica di business
- run: pnpm run build
- run: pnpm run test
- run: pnpm run lint

# ✅ Buono - usa shell per operazioni CI/CD
- run: mkdir -p artifacts
- run: cp -r dist/ artifacts/
- run: ls -la artifacts/

# ❌ Evitare - comando complesso che dovrebbe essere uno script npm
- run: |
    tsc -b && 
    vite build --mode production && 
    cp dist/index.html dist/200.html &&
    gzip -9 dist/*.js
```

---

Buon lavoro! 🚀
