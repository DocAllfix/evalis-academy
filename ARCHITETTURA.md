# ARCHITETTURA — FormazioneEvalis (LMS B2B+B2C)

> **Documento di riferimento unico per lo sviluppo.** Fissa stack, moduli, schema dati, fonti open source e criteri di successo. Deriva da: CLAUDE.md, `docsricerca/`, EduVault, e l'analisi di 13 repo clonate in `references/` (report in `references/_REPORT/`).
> **Aggiornato:** 2026-06-21. **Stato decisioni:** 21/21 confermate (`_REPORT/90-decisioni.md`) + decisioni post-cliente su formato corso, avatar e standard componibili (vedi §6).

---

## 0. Cosa stiamo costruendo

Piattaforma LMS multi-tenant per erogare corsi di formazione professionale (sicurezza D.Lgs. 81/08 + Accordo Stato-Regioni 2025, ISO, auditor), 4–8 ore, fruiti in autonomia. I corsi arrivano **già pronti** (video con avatar + slide + quiz) da monte (incl. EduVault per i contenuti). Qui costruiamo solo l'**erogazione**: accesso, tracciamento antifrode, quiz, certificati, pagamenti.

**Due tipi di cliente, un solo modello dati:**
- **B2B** — azienda (organization) con subscription seat-based; l'admin assegna i seat ai dipendenti.
- **B2C** — utente singolo, signup self-service aperto, acquista corsi one-off. Modellato come **organizzazione personale** (1 membro owner).

**Vincoli normativi non negoziabili** (CLAUDE.md): tracciabilità append-only di ogni attività; tempo minimo di fruizione verificato lato server; quiz a estrazione casuale con soglia; sessione singola attiva; data residency UE per DB/auth/dati personali; certificato emesso solo dopo revisione umana.

---

## 1. Stack definitivo

| Livello | Tecnologia | Categoria |
|---|---|---|
| App (frontend+backend) | **Next.js 15** App Router, TypeScript, Server Actions + Route Handlers | monolite (C-01) |
| UI | shadcn/ui + TailwindCSS | — |
| ORM | **Drizzle** | DEP (C-19) |
| DB + Storage | **Supabase EU** (Postgres `eu-central-1` + Storage + RLS) | DEP managed (C-02) |
| Auth + multi-tenancy | **better-auth** + organization plugin (v1.6.20) | DEP (C-03) |
| Pagamenti | **Stripe** (subscription seat + one-off) | DEP (C-05) |
| Video hosting | **Cloudflare Stream** (signed URL RS256 JWT) | DEP (C-06) |
| Player streaming | **hls.js** dentro `<video>` nativo | DEP (C-07) |
| SCORM (secondario) | **scorm-again v3.0.6** | DEP (C-09) |
| Certificati | **pdf-lib/pdfkit + qrcode** | DEP (C-15) |
| Email | **Brevo** (EU) | DEP (C-16) |
| Monitoring | **Sentry** | DEP (C-17) |
| Job async | **Trigger.dev** — solo quando serve | DEP (C-18) |
| LRS xAPI | **lrsql** — differito post-MVP | FUTURO (C-12) |

**Policy OSS (C-21):** *DEPENDENCY* (MIT/Apache/ISC, si wrappa con notice) · *CLONE-REF* (si studia e reimplementa, mai lift — vale per AGPL/GPL e per i 3 repo senza licenza) · *BUILD* (codice nostro). Idee/schemi/pattern sono liberi; il codice letterale no.

**Frontend:** vive in questo codebase (113 componenti in `src/components/`). L'impostazione visiva della landing pubblica nasce da un generatore esterno e viene poi integrata e mantenuta qui; le pagine di prodotto — dashboard, player, console azienda e staff — sono costruite direttamente. Confine sul player: la *logica* antifrode e' nostra e sta sul server, il livello visivo e' React.

---

## 2. I moduli

Per ogni modulo: scopo, categoria, fonti, cosa prendiamo / cosa costruiamo, dati, criterio di successo.

### Modulo 1 — Multi-tenancy, Auth, Org & Seat  `[DEP + BUILD]`
- **Fonte:** better-auth org plugin (`_REPORT/02`). Reference minore: saas-starter, ixartz.
- **Ci dà:** `organization`, `member`, `invitation`, `team`, `organizationRole` (RBAC), `session` (con `activeOrganizationId`); signup/login/verify-email/reset nativi.
- **Costruiamo:** hook `onSignup` che crea l'org personale B2C (`metadata.type`); vincolo `seats = subscription.quantity`.
- **Criterio:** test → due login stesso utente ⇒ prima sessione invalidata; B2C signup ⇒ org personale creata con 1 owner.

### Modulo 2 — Billing Stripe (B2B seat + B2C one-off)  `[DEP + BUILD]`
- **Fonte:** saas-starter (`_REPORT/01`), shounoop `Purchase`, courselit membership (`_REPORT/06`).
- **Ci dà:** wiring checkout/portal/webhook; modello acquisto one-off `Purchase(user,course)`.
- **Costruiamo:** due prezzi Stripe (subscription seat `quantity=posti` / one-off per corso); webhook → `checkout.session.completed` (one-off) crea `enrollment`; `customer.subscription.deleted` revoca enrollment dei seat. **Nessuna assunzione su proration/rimborsi senza conferma** (CLAUDE.md §1).
- **Criterio:** acquisto one-off ⇒ enrollment per quel corso; subscription cancellata ⇒ accesso revocato.

### Modulo 3 — Catalogo e dominio corsi  `[CLONE-REF + BUILD]`
- **Fonte:** learnhouse Course/Chapter/Activity (`_REPORT/03`), sonnysangha gating (`_REPORT/06`).
- **Ci dà (pattern):** gerarchia `course → module → lesson`, `activity_type` (video/scorm/document), ordinamento, gating.
- **Costruiamo:** schema Drizzle con `min_required_seconds` per lezione; scoping `organization_id`.
- **Criterio:** `GET next lesson` ⇒ 403 se la precedente non è `completed`.

### Modulo 4 — Player video antifrode ⭐  `[DEP + BUILD]`
- **Fonte:** hls.js (DEP); react-player v3 solo pattern (`_REPORT/05`).
- **Ci dà:** riproduzione HLS dello stream firmato Cloudflare.
- **Costruiamo (cuore):** hook `useSecureVideo` su `<video>` nativo — blocco seek oltre `maxValidatedSeconds` (`seeking`/`seeked`), pausa su `visibilitychange`/`blur`, heartbeat ogni N s, AFK/presenza attiva, overlay quiz a timestamp.
- **Criterio:** seek in avanti oltre il validato ⇒ riportato indietro; tab nascosta ⇒ conteggio sospeso + evento loggato.

### Modulo 5 — Tracciamento server-authoritative  `[BUILD]`
- **Fonte:** ADL_LRS solo per i verbi xAPI (`_REPORT/05`). Nessun OSS lo copre.
- **Costruiamo:** endpoint heartbeat che incrementa `effective_watch_seconds` lato server validando (playing+visibile+no salto); `completed_at` calcolato dal server; gating sequenziale non aggirabile via API.
- **Criterio (CLAUDE.md §4):** marcare lezione completata con `< min_required_seconds` ⇒ 403; con tempo raggiunto ⇒ 200.

### Modulo 6 — Audit log append-only  `[BUILD]`
- **Fonte:** pattern EduVault D-13 (ruolo SQL write-only). Verbi da ADL_LRS.
- **Costruiamo:** `activity_log` append-only con `prev_hash`/`hash` (hash-chaining); scrittura via **ruolo Postgres con solo INSERT** (REVOKE UPDATE/DELETE) + trigger anti-modifica su Supabase.
- **Criterio:** UPDATE/DELETE sull'`activity_log` dal ruolo app ⇒ negato; catena hash verificabile.

### Modulo 7 — Quiz engine compliance  `[BUILD]`
- **Fonte:** learnhouse (idea attività quiz). Nessun OSS copre i requisiti normativi.
- **Costruiamo:** banca domande + **estrazione casuale**, soglia configurabile bloccante, limite tentativi + `locked_until`; tutto storicizzato in `quiz_attempt`.
- **Criterio:** generazione certificato negata senza quiz superato; domande estratte casualmente dalla banca.

### Modulo 8 — SCORM (canale secondario)  `[DEP + BUILD]`
- **Fonte:** scorm-again v3.0.6 (`_REPORT/04`).
- **Ci dà:** runtime SCORM 1.2/2004, CrossFrame (iframe sandbox), `lmsCommitUrl`.
- **Costruiamo:** wrapper `<ScormPlayer>` che punta `lmsCommitUrl` al nostro endpoint e mappa i CMI → audit/progress. **Non alteriamo il sorgente.**
- **Criterio:** un pacchetto SCORM di test produce eventi commit registrati nel nostro audit log.

### Modulo 9 — xAPI / LRS  `[FUTURO]`
- **Fonte:** lrsql (Apache, Docker). **Nessuna azione ora**: oggi basta `activity_log`. Se un cliente esige un LRS certificato, aggiungiamo lrsql come sink e mappiamo eventi→statement (verbi già allineati ad ADL_LRS).

### Modulo 10 — Certificati PDF verificabili  `[DEP + BUILD]`
- **Fonte:** pdf-cert (pattern), qr-certificate-generator (workflow verify), qrcode/pdf-lib (DEP) (`_REPORT/06`).
- **Ci dà:** generazione PDF con dati dinamici + firma/timbro; QR; idea pagina di verifica.
- **Costruiamo:** generazione **solo** se tempo+quiz+audit ok; stato `ready_for_review → approved → issued` (**mai invio automatico**, CLAUDE.md); pagina pubblica `/verify/:uuid`.
- **Criterio:** certificato resta `ready_for_review` finché un umano non approva; `/verify/:uuid` mostra stato valido/revocato.

### Modulo 11 — Sessione singola attiva  `[BUILD]`
- **Fonte:** tabella `session` di better-auth.
- **Costruiamo:** al nuovo login invalidiamo le sessioni precedenti dell'utente.
- **Criterio:** login da secondo device ⇒ prima sessione terminata.

### Modulo 12 — Trasversali (managed)  `[DEP]`
Supabase EU (DB/Storage/RLS) · Cloudflare Stream (video, signed URL) · Sentry (errori) · Brevo (email EU) · Trigger.dev (job, quando serve). Zero codice nostro oltre config + wrapper minimi.

---

## 3. Schema DB (3 cluster — ogni record di dominio porta `organization_id`)

**Cluster A — Auth/Tenant (better-auth):** `user`, `session` (+`active_organization_id`, +supporto single-session), `account`, `organization` (`metadata.type = personal|company`, campi Stripe: `stripe_customer_id`, `stripe_subscription_id`, `plan`, `subscription_status`, `seats`), `member`, `invitation`, `organization_role`.

**Cluster B — Catalogo & Fruizione (BUILD):**
- `course` (`org_id?` null=catalogo globale, titolo, descrizione, durata, stato)
- `module` (`course_id`, ordine)
- `lesson` (`module_id`, `activity_type` video|scorm|document, `video_uid`, **`min_required_seconds`**, ordine)
- `enrollment` (`organization_id`, `user_id`, `course_id`, `source` b2b_seat|b2c_purchase, `expires_at?`, `status`)
- `lesson_progress` (`enrollment_id`, `lesson_id`, **`effective_watch_seconds`**, `max_validated_seconds`, `completed_at?`)
- `quiz_question` (`course_id`, testo, `options` JSON, corretta) · `quiz_attempt` (`enrollment_id`, `score`, `passed`, `locked_until?`, `detail` JSON)

**Cluster C — Compliance (BUILD, append-only):**
- `activity_log` (`organization_id`, `user_id`, `verb`, `object`, `payload` JSON, `prev_hash`, `hash`, `created_at`) — **ruolo SQL solo-INSERT**
- `heartbeat` (`enrollment_id`, `lesson_id`, `position`, `focus`, `ts`)
- `certificate` (`enrollment_id`, `status` ready_for_review|approved|issued, `verify_uuid` unique, `pdf_path?`, `approved_by?`, ts)

> Nota: `enrollment` resta unico anche per il B2C (org personale). `expires_at` opzionale copre la scadenza accesso B2C (default: nessuna scadenza finché non confermato diversamente).

---

## 4. Ordine di implementazione (vertical slices verificabili)

1. **Fondamenta:** Supabase EU + Drizzle + migrazione (Cluster A,B,C) + ruolo audit write-only.
2. **Auth/tenant:** better-auth + org plugin → signup B2C (org personale) + invito B2B verificabili.
3. **Billing:** Stripe test → subscription seat (B2B) + one-off (B2C) + webhook.
4. **Vertical slice cuore:** una lezione video Cloudflare → `<video>`+hls.js antifrode → heartbeat → `lesson_progress` server-side → **criterio §4 (403/200)**.
5. **Quiz + gating** → **Certificato con revisione umana** → **Sessione singola**.
6. SCORM secondario (scorm-again) quando arriva il primo pacchetto reale.

---

## 5. Cosa NON facciamo (vincoli da CLAUDE.md)
- Niente microservizi prematuri (`services/lrs`, `services/certificates`) finché il monolite non mostra un collo di bottiglia reale.
- Niente astrazione provider (storage/email/pagamenti) finché non c'è un secondo provider reale.
- Niente indebolimento dei controlli tempo/AFK per "semplificare".
- Niente invio automatico certificati senza revisione umana.
- Niente modifica del codice vendorizzato (scorm-again, ecc.): wrappa, non alterare.
- L'`activity_log` non si tocca "di striscio": se una modifica lo sfiora, fermarsi e segnalarlo.

---

## 6. Formato corso HTML-first, avatar, standard & moduli componibili

> Decisioni post-cliente. Report di dettaglio: `_REPORT/95` (struttura corso HTML), `96` (avatar), `97` (build vs borrow), `98` (standard & moduli componibili). Questa sezione **estende** lo stack di §1 e precisa i Moduli 4–5.

### 6.1 Formato corso = HTML-first (non un unico video)
Un corso è una **sequenza di slide HTML interattive**, ognuna con: contenuto, **clip avatar "testa parlante" per-slide**, eventuali checkpoint/quiz. NON un video renderizzato unico. Più leggero, editabile, multilingua, mobile, e con antifrode più forte (partecipazione attiva). Si aggiunge l'`activity_type = html` accanto a `video`/`scorm`.
- **Layout (vedi `mockups/`):** A = avatar nell'angolo su slide HTML; B = avatar in scena/ambiente; C = immersivo. Mescolabili. Generati **una volta per corso**, ri-generabili.
- **Convenzione di authoring (da artifact cliente, `_REPORT/95`):** ogni slide porta `data-speaker-notes` = **lo script di narrazione** → è il punto d'innesto dell'avatar (testo→clip). La durata della clip ≈ tempo minimo naturale della slide; avanzamento al **completamento audio**.
- **Conseguenza sul Modulo 4:** la logica antifrode (`useSecureVideo`) si applica anche al **player a slide** (tempo minimo per-slide + completamento clip/audio + presenza), non solo al video lungo. Il video resta un activity_type secondario.

### 6.2 Avatar (produzione) — `_REPORT/96`
- **Vincolo di costo:** corsi lunghi ⇒ il prezzo SaaS *al minuto* esplode. Volume iniziale: **4 corsi (2×4h + 2×8h) ≈ 1.440 min**.
- **Per partire:** **Colossyan** (EU, AWS Germania/UK, minuti illimitati ~$80/mese, scena B nativa) o **HeyGen** (miglior API, A+B, avatar anche **fittizi da prompt**). Test gratuito Colossyan: 14 gg / 3 min, no carta. Da verificare qualità italiano + fair-use API.
- **Self-host (fase 2, costi/sovranità):** **OpenVoice** (voce, MIT) + **LivePortrait/SadTalker/Ditto** (talking-head) → alpha vero via matting (RVM/MODNet). "Talking Slide Avatars".
- **Scartati per il bulk:** D-ID ($5.90/min), Synthesia (enterprise $$$).
- **Anti-lock-in:** conservare sempre script + audio + immagine sorgente + voice model come asset indipendenti.

### 6.3 Build vs Borrow — `_REPORT/97`
- **BORROW (dependency MIT):** `reveal.js`+audio-slideshow (motore slide), `hls.js` (video), `scorm-again` (import), `OpenVoice` (voce, fase 2).
- **CLONE-REF (studia & reimplementa):** Adapt (struttura corso JSON), Moodle/TCExam (modello quiz random-bank), xAPI (schema eventi). H5P/OpenMAIC solo per idee di interazione.
- **BUILD (il valore, nessun OSS lo regala):** antifrode server-authoritative (tempo minimo, completamento audio/clip, presenza, sessione singola), audit append-only hash-chain, **correzione esame server-side**, gate certificato con revisione umana.

### 6.4 Standard di riferimento (conformiamo la *forma dei dati*) — `_REPORT/98`
| Standard | Uso | Quando |
|---|---|---|
| **xAPI** (IEEE 9274.1.1) + **cmi5** | forma `actor–verb–object` dell'`activity_log`; semantica launch/completion/score/duration | schema NOW; emissione vera = al bisogno (lib `@xapi/xapi`, `@xapi/cmi5`) |
| **QTI 3.0** | modello portabile banca quiz (import/export) | schema NOW; import/export poi |
| **Open Badges 3.0 / W3C VC** | certificato firmato e verificabile in autonomia | fase 2: si parte PDF+QR+UUID, si aggiorna a VC |
| **LTI 1.3** | embed tool esterni / SSO forum-ticketing | quando matura l'integrazione |
| **OIDC/OAuth2** | better-auth come provider SSO | con i tool esterni |
| **Page Visibility API · Media events · WebVTT · WCAG 2.2** | presenza/anti-AFK, completamento audio, sottotitoli, accessibilità | NOW (nativi, no librerie) |

**Moduli a singola responsabilità (comporre):** reveal.js · hls.js · scorm-again · @xapi/xapi+cmi5 · pdf-lib · qrcode · zod · better-auth · drizzle · Node `crypto` (hash-chain) · BullMQ/Trigger.dev (code job). **Trappola da evitare:** adottare ORA cmi5/QTI/LTI/Open-Badges come runtime completi = over-engineering (CLAUDE.md §2). Si conforma la *forma* subito (~zero costo), si tira dentro il peso solo al bisogno reale.

---

## Appendice — Indice analisi
Report completi in `references/_REPORT/`: `00-inventory`, `01-saas-starter`, `02-better-auth`, `03-learnhouse`, `04-scorm-again`, `05-ixartz-reactplayer-adllrs`, `06-courselit-sonnysangha-cert-shounoop`, `90-decisioni`, `91-schema-mapping`, `95-html-course-references`, `96-avatar-solution-analysis`, `97-oss-references-build-vs-borrow`, `98-standard-e-moduli-componibili`.
