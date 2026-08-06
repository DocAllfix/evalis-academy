# Sicurezza

Questo documento descrive **cosa è implementato e dove**, e cosa **non** lo è. Ogni voce è
verificabile leggendo il file indicato: non ci sono affermazioni senza un posto nel codice.

## Segnalare una vulnerabilità

Scrivere a **evaliscert@libero.it** con oggetto `[security]`. Chiediamo di non divulgare
pubblicamente prima che sia stata corretta. Rispondiamo entro 5 giorni lavorativi.

---

## Modello di minaccia

Chi attacca questa piattaforma, e cosa vuole:

| Attaccante | Obiettivo | Perché è il rischio principale |
|---|---|---|
| **Un discente** | Ottenere l'attestato senza seguire il corso | È il rischio **commerciale** più grave: un attestato non guadagnato rende la piattaforma invendibile e mette a rischio l'accreditamento |
| **Un'azienda cliente** | Vedere i dati di un'altra azienda | Multi-tenancy: dati di formazione di dipendenti terzi |
| **Un esterno** | Accedere ad aree amministrative o ai contenuti a pagamento | Superficie classica |
| **Chi ha accesso all'editor del blog** | Iniettare script sul dominio dove gli utenti sono autenticati | Il CMS è di terze parti (WordPress) |

---

## Controlli implementati

### Autenticazione e sessione
- **better-auth** con organizzazioni, inviti, ruoli (`owner`, `admin`, `member`) e ruolo di
  piattaforma separato (`user.platformRole`).
- **Verifica email obbligatoria** (`requireEmailVerification: true`, `src/lib/auth/index.ts`).
- **Sessione singola attiva**: un secondo accesso invalida il precedente.
- Reset password via Resend; i token non vengono mai registrati nei log in produzione.

### Isolamento multi-tenant — Row Level Security
Attiva **in produzione**:
- l'applicazione si connette come ruolo **`app_rls`** con `NOBYPASSRLS`;
- `FORCE ROW LEVEL SECURITY` + policy sulle tabelle sensibili (migrazione `0007`, valvole
  `0012`, `0014`, `0015`);
- ogni query alle tabelle tenant passa da **`withTenant(ctx, fn)`** (`src/lib/db/tenant.ts`),
  che imposta le GUC dentro una transazione;
- guardrail eseguibile: `RLS_FORCE_ROLE=app_rls npx vitest run src/__tests__/rls-routing.db.test.ts`
  — verifica che il **cross-tenant sia negato** eseguendo davvero come `app_rls`.

**Difesa in profondità, non barriera unica:** i controlli applicativi (`requireRole`,
`requireActiveOrg`, verifiche di appartenenza) restano al loro posto. La RLS è la rete sotto.

### Tracciamento a prova di aggiramento
- Il tempo si accredita sull'**orologio del server** (`creditableSeconds`,
  `src/features/tracking/progress.ts`): un client che sposta il proprio orologio o falsifica i
  battiti non ottiene credito, perché l'accredito è limitato dal tempo reale trascorso.
- Limite di frequenza sull'endpoint dei battiti (120/minuto per iscrizione).
- Proprietà dell'iscrizione verificata a ogni battito (`assertEnrollmentOwnedBy`).

### Quiz
- Al browser arrivano **solo** `id`, `text`, `options`: la risposta corretta
  (`correctOptionId`) **non lascia mai il server** (`src/features/quiz/engine.ts`).
- Correzione server-side; domande estratte casualmente dalla banca.
- Blocco temporale dopo i tentativi falliti (`lockedUntil`).

### Registro append-only
- `activity_log` con **catena hash per organizzazione**: ogni record firma il precedente
  (`prevHash` → `hash`), con indice unico su `(organizationId, hash)`.
- Cancellare o alterare una riga rompe la catena, ed è rilevabile.
- Nessuna operazione applicativa aggiorna o cancella righe: solo inserimento.

### Pagamenti
- Firma dei webhook Stripe **verificata** su ogni richiesta; senza firma valida → 400.
- Idempotenza sugli eventi (claim rilasciabile per il ritentativo).
- Il provisioning avviene **solo** dal webhook, mai dal ritorno del browser.

### Contenuti video
- Cloudflare Stream con **token firmati RS256** a scadenza: gli URL dei video non sono
  condivisibili in modo permanente.

### HTML di terze parti (blog)
- Il contenuto che arriva da WordPress passa da una **lista bianca stretta**
  (`src/features/blog/sanitize.ts`): script, gestori di eventi inline e schemi pericolosi
  vengono rimossi prima di raggiungere la pagina.
- Il CMS è su un sottodominio chiuso, `noindex`, con la dashboard protetta da password HTTP.

### Intestazioni HTTP
`next.config.ts`: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
`Permissions-Policy`, `Strict-Transport-Security`.

La **Content-Security-Policy** e' in vigore come `Content-Security-Policy-Report-Only`: il
browser segnala le violazioni e le raccogliamo per stringere le direttive sui casi reali prima
di renderla bloccante. Una CSP scritta a tavolino e messa subito in blocco spegne il player
video — cioe' il prodotto — su un caso che nessuno aveva previsto.

### Segreti e rotte amministrative
- I confronti fra segreti passano da `src/lib/segreti.ts`: **hash SHA-256 + `timingSafeEqual`**.
  Un `===` fra stringhe si ferma al primo carattere diverso e lascia trapelare il segreto un
  carattere alla volta a chi misura i tempi di risposta. La funzione **fallisce chiusa** se la
  variabile d'ambiente manca: una env dimenticata non deve rendere valido qualsiasi valore.
- La rotta di manutenzione Stripe (`/api/staff/stripe-golive`) ha tre serrature: **spenta di
  default** (senza `STRIPE_ADMIN_ENABLED=1` risponde 404 a chiunque), **token dedicato**
  (`STRIPE_ADMIN_TOKEN`, non condiviso con altre funzioni), confronto a tempo costante.
- Le rotte a token rispondono **404 e non 401**: a chi bussa non si conferma che esistano.

### Privacy
- Consenso ai cookie con **Consent Mode v2**: i quattro parametri partono **negati** e Google
  Analytics **non viene caricato** finché non c'è un consenso esplicito. Verificabile con
  `node scripts/produzione/_verifica-consenso.mjs <url>`.
- Nessun avatar da terze parti: le immagini degli autori si accettano solo se ospitate da noi.

---

## Gestione dei segreti

- **Nessun segreto è mai stato committato.** Verificato sull'intera cronologia cercando chiavi
  Stripe, segreti webhook, URL di database con credenziali, JWT Supabase, chiavi Resend,
  token Cloudflare, Azure e Sentry: ogni corrispondenza è un confronto nel codice, un
  segnaposto di `.env.example` o uno stub di test.
- I segreti vivono nelle variabili d'ambiente della piattaforma di hosting, mai nel codice.
  Le operazioni che richiedono la chiave di pagamento girano lato server: quella chiave non e'
  leggibile da un ambiente di sviluppo.
- I segreti usati in sviluppo si considerano bruciati e vengono ruotati prima della messa in
  esercizio.
- `.env.example` documenta ogni variabile senza valori.
