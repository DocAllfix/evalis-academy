# Evalis Academy — piattaforma LMS

Piattaforma **LMS multi-tenant** per l'erogazione di corsi di formazione professionale
certificata (schemi ISO, auditor, mestieri specialistici). Costruiamo l'**erogazione**: accesso,
tracciamento della fruizione a prova di aggiramento, quiz, certificati, pagamenti. I corsi
arrivano già prodotti da monte (avatar video + slide + banca domande).

> 📐 **La fonte di verità architetturale è [`ARCHITETTURA.md`](./ARCHITETTURA.md)**: stack, 12
> moduli, schema del database, criteri di successo.
> La postura di sicurezza e' in [`SECURITY.md`](./SECURITY.md), le scelte di design del
> prodotto in [`PRODUCT.md`](./PRODUCT.md) e [`DESIGN.md`](./DESIGN.md).

---

## Perché esiste, in una riga

Un corso di formazione vale come titolo solo se si può **dimostrare** che è stato seguito. Tutto
ciò che segue — il tracciamento sul server, il registro append-only, il quiz corretto lato
server — serve a rendere quella dimostrazione difendibile davanti a un'ispezione.

## Vincoli non negoziabili

Non sono requisiti di prodotto: sono le condizioni che rendono la piattaforma vendibile a
clienti professionali italiani. Ognuno ha un posto preciso nel codice.

| Vincolo | Dove vive |
|---|---|
| Tracciabilità completa e **append-only** di ogni attività | `src/features/audit/log.ts` — catena hash per organizzazione |
| **Tempo minimo** di fruizione verificato sul server, non aggirabile via API | `src/features/tracking/progress.ts` — `creditableSeconds` sull'orologio del server |
| Quiz a estrazione casuale, soglia configurabile, bloccanti | `src/features/quiz/engine.ts` — correzione server-side |
| **Sessione singola attiva** per utente | `src/features/auth/` |
| **Residenza dati UE** per database, storage e autenticazione | Supabase `eu-central-1`, Cloudflare Stream |
| Certificato emesso solo a requisiti soddisfatti, **inviato dopo revisione umana** | `src/features/certificates/lifecycle.ts` |

## Stack

**Next.js 16.2.9** (App Router, TypeScript) · **Drizzle ORM** · **Supabase EU**
(Postgres + Storage, RLS attiva) · **better-auth** (organizzazioni, inviti, ruoli) ·
**Stripe** (+ Stripe Tax) · **Cloudflare Stream** (video firmati) · `<video>` + hls.js
(player antifrode) · **scorm-again** · **Resend** (email) · **Sentry** · **Vitest** +
**Playwright**.

## Struttura

```
src/app/            rotte: (marketing) (auth) (app) (admin) api verify
src/features/       domini: auth, billing, courses, player, tracking, quiz,
                    certificates, audit, catalog, blog, consenso, support
src/lib/            infrastruttura: db (schema per cluster + migrazioni), auth,
                    supabase, cloudflare, stripe, email, env
src/proxy.ts        multi-tenancy per sottodominio + 410 sugli URL rimossi
e2e/                Playwright
scripts/produzione/ script di verifica: blog, consenso, listino, scatti
```

## Avvio

```bash
npm ci                        # dipendenze esatte dal lockfile
cp .env.example .env          # e riempire (l'avvio fallisce se manca una variabile richiesta)
npm run db:migrate            # applica le migrazioni Drizzle
npm run dev
```

Le variabili d'ambiente sono **validate all'avvio** da `src/lib/env.ts`: se ne manca una
obbligatoria il processo si ferma con un messaggio esplicito, invece di degradare in silenzio.

## Verifiche

```bash
npm run build                 # compilazione
npm test                      # suite Vitest
npx playwright test           # end-to-end

# guardrail isolamento multi-tenant: esegue i test COME il ruolo ristretto di produzione
RLS_FORCE_ROLE=app_rls npx vitest run src/__tests__/rls-routing.db.test.ts

# controlli sul sito pubblicato
npx tsx scripts/produzione/_verifica-blog.ts --sito https://… --cms https://…
node scripts/produzione/_verifica-consenso.mjs https://…
```

⚠️ I test con suffisso `.db.test.ts` **toccano un database vero**: vanno eseguiti contro
un'istanza di sviluppo, mai contro la produzione.

## Convenzioni

- **Regola inderogabile sui commit:** i commit sono firmati **solo come `DocAllfix`** e non
  devono mai contenere firme o co-author di terzi o di assistenti. Un hook `commit-msg`
  (`.githooks/`) lo verifica automaticamente.
- Modifiche chirurgiche: si tocca solo ciò che serve, si pulisce solo il proprio disordine.
- **Lo schema `compliance` (registro append-only) non si tocca «di striscio»**: quei dati hanno
  valore probatorio e la catena hash si rompe se qualcuno cancella una riga.
- Ogni modifica al database passa da una **migrazione versionata**, mai da un comando estemporaneo.

## Perimetro di questo repository

Qui vive **solo la piattaforma**. La produzione dei contenuti — copioni, sintesi vocale, avatar,
slide, pacchetti dei corsi — sta in un repository separato: è il 99% del peso e non serve a chi
lavora sull'applicazione. Il `.gitignore` la esclude per costruzione.
