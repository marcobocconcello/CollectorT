# Cronologia della discussione architetturale — CollectorT

> Esportazione della chat di progettazione architetturale del 21/11/2026.  
> Per il documento finale di sintesi vedi [architettura.md](./architettura.md).

---

## 1. Idea iniziale e fonti dei prezzi

**Utente:**
> Ciao, vorrei creare un'app per controllare lo stato ed il valore riguardo al collezionismo.
> So che in questo modo la scelta è un po' ampia, nel senso che se si collezionano carte i valori di riferimento dovrebbero essere presi da Cardmarket, se si collezionano Vespe il valore di riferimento sarà su un'altra piattaforma che non conosco ecc.
> Vorrei quindi il tuo parere, per esempio: al caricamento dell'oggetto di collezione particolare dovrei andare a cercare le info?

**Assistente:**
- Separare **oggetto di catalogo** (astratto, con ID esterno), **oggetto posseduto** (copia specifica con condizione, prezzo e data d'acquisto) e **rilevazione di prezzo** (valore nel tempo, con fonte e data).
- Flusso: ricerca e collegamento all'ID esterno in fase di aggiunta + primo prezzo; aggiornamento periodico in background; lettura solo dal DB locale/server in consultazione.
- Pattern a **fornitori di prezzo** (`PriceProvider`) per categoria, più valore manuale sempre disponibile. Evitare lo scraping (vietato dai ToS e fragile).

---

## 2. Mobile e necessità del backend

**Utente:**
> Più mobile a dire il vero. Ma servirà anche una parte server side altrimenti il db delle carte dove lo tiene?

**Assistente:**
- Sì, serve un backend per: proteggere chiavi API, eseguire job periodici di aggiornamento prezzi, condividere catalogo/prezzi tra utenti (evitando limiti API) e sincronizzare/salvare le collezioni.
- Sul telefono: UI + cache locale SQLite per velocità e uso offline.

---

## 3. Pubblicazione e categoria iniziale (Pokémon)

**Utente:**
> Vorrei pubblicare l'app. Inoltre la categoria dalla quale vorrei partire è Pokémon.

**Assistente:**
- Verifica fonti Pokémon: la vecchia API gratuita `pokemontcg.io` è stata dismessa e migrata su **Scrydex** (a pagamento, ~\$29–\$399/mese). **TCGdex** (`tcgdex.dev`) è gratuita, multilingua (incluso italiano) e include prezzi Cardmarket (€) e TCGplayer ($).
- Aspetti legati alla pubblicazione: copyright e disclaimer "app non affiliata", GDPR (server UE, export/cancellazione account), regole store (Sign in with Apple se presenti altri login social), fonte e data sui prezzi.

---

## 4. Requisiti funzionali dettagliati

**Utente:**
> No aspettiamo a creare qualsiasi cosa o ad installare. Prima pensiamo alla parte architetturale e poi valutiamo cosa fare.
> L'applicazione deve esporre sostanzialmente le seguenti funzionalità:
> - Ricerca per nome della carta/numero
> - Ricerca per foto della carta
> - Visualizzazione della collezione in formato griglia con la foto della carta: se caricata da utente si usa quella, se cercata per nome/numero si usa un'immagine di copertina (quindi in DB bisogna avere tutte le immagini di tutte le carte)
> - Dati della collezione, quindi grafico totale del valore rispetto al prezzo di acquisto un po' come Scalable
> - Se si clicca sulla singola carta, il grafico del valore della singola carta con il saldo rispetto al prezzo di acquisto
> - Fonte dei dati deve essere Cardmarket
> - Possibilità di fare login con gestione delle credenziali ecc.
> - Non è ufficiale quindi con solo le info pubbliche non soggette a copy

**Assistente:**
Creata la prima bozza di `architettura.md` evidenziando tre vincoli:
1. **Immagini e copyright:** le immagini delle carte sono protette da copyright. Proposte 3 opzioni: **A** (salvare solo URL esterno + cache locale), **B** (hosting nostro), **C** (solo foto utente + segnaposto).
2. **Accesso a Cardmarket:** API chiusa ai nuovi sviluppatori e scraping vietato; si usano i prezzi Cardmarket tramite **TCGdex** (verificando i termini d'uso commerciale).
3. **Storico prezzi:** TCGdex dà solo il prezzo attuale e medie 1/7/30g; lo storico va costruito con un job giornaliero avviato per primo.
4. **Ricerca per foto ibrida:** OCR on-device (ML Kit) in prima versione, riconoscimento visivo con vettori immagine (`pgvector`) in fase successiva.

---

## 5. Prima tornata di decisioni

**Commenti utente su `architettura.md`:**
- Immagini: *"Mi pare di capire che l'unica legale per un'app pubblicata su store sia la A"*
- Prezzo di riferimento: *"prezzo di riferimento quello istantaneo, quindi il trend. Sì vorrei anche il coefficiente di condizione."*
- Lingua: *"Vorrei distinzione per lingua. Il prezzo cambia molto a seconda della lingua"*
- Modello di business: *"Gratuita"*
- Backend: *"Backend personalizzato. L'idea è .NET 8, Postgres, e rilasciato su k8s"*

**Assistente:**
- **Opzione A** riduce il rischio ma non lo azzera: servono disclaimer, procedura di takedown e niente loghi/"Pokémon" nel nome o nell'icona.
- **`trend` Cardmarket** è una media smussata delle vendite recenti (stabile e ideale per grafici di portafoglio). Coefficienti di condizione sulla scala Cardmarket (MT..PO).
- **Prezzi per lingua:** Cardmarket/TCGdex non forniscono prezzi separati per le lingue occidentali (un solo `trend` per carta); le carte giapponesi invece sono prodotti separati. Strategia: catalogo separato per JP, coefficienti di lingua configurabili per le lingue occidentali, schema DB già predisposto con colonna `language` per eventuali prezzi reali futuri.
- **.NET 10 (LTS)** consigliato al posto di .NET 8 (uscito dal supporto il 10/11/2026). Architettura a monolite modulare + CronJob.

---

## 6. Seconda tornata di decisioni

**Commenti utente su `architettura.md`:**
- Mobile: *"Va bene Flutter"*
- Auth: *"OpenIddict"*
- Hosting: *"Non ci sono cluster gratuiti k8s?"*
- Coefficienti: *"Fissiamo i coefficienti"*
- Carte giapponesi: *"Subito presenti"*

**Assistente:**
- **OpenIddict + ASP.NET Core Identity** confermato (servizio `CollectorT.Auth`, molta meno RAM di Keycloak). Provider SMTP con piano gratuito (es. Brevo/Resend) per conferma email e reset password.
- **Cluster k8s gratuiti:** GKE/AKS fanno pagare i nodi; l'unica opzione gratuita senza scadenza è **Oracle Cloud Always Free** (VM ARM Ampere A1, ~2 OCPU / 12 GB RAM, 200 GB disco, 10 TB egress) con **k3s** e immagini `arm64`, più backup automatici di PostgreSQL su object storage.
- **Carte giapponesi:** catalogo completo su TCGdex, ma prezzi Cardmarket spesso assenti (`null`). Previsto stato "valore non disponibile" e collegamento alla controparte internazionale per la ricerca.

---

## 7. Terza tornata di decisioni

**Commenti utente su `architettura.md`:**
- Hosting: *"Cos'è k3s?"*
- Carte giapponesi senza prezzo: *"Non conteggiarle"*
- Login social: *"Facciamo una prima versione senza Auth di terze parti? Ma in questo modo dobbiamo gestire le credenziali corretto?"*
- Nome app: *"Sì va bene"* (CollectorT)

**Assistente:**
- Spiegato **k3s** (Kubernetes leggero certificato da Rancher/SUSE, stessi YAML/Helm/`kubectl`, ideale per un singolo nodo con poca RAM).
- Carte senza prezzo escluse **sia dal valore sia dall'investito** per non falsare il saldo, con indicazione in dashboard.
- Prima versione con **solo email e password**: le credenziali sono gestite tramite ASP.NET Identity (hash PBKDF2, lockout, token monouso) + nostre misure (rate limiting, policy password, HTTPS, revoca token, no log sensibili). Rimandare Google/Apple evita anche l'obbligo di Sign in with Apple su iOS nella v1.

---

## 8. Versione web in Angular e strategia di test

**Utente:**
- Conferma su Oracle Cloud Always Free + k3s: *"Sì confermo."*
- *"Poi facciamo anche una versione web in Angular? Senza scannerizzazione della foto."*

**Assistente:**
- Aggiunta **web app Angular** (§5.1): usa le stesse API e lo stesso `CollectorT.Auth` (client OIDC separato con PKCE), servita da un pod `nginx` `arm64` su k3s. Client TypeScript e Dart generati dalla stessa specifica OpenAPI; tutta la logica nel backend.

**Commenti utente:**
- Caricamento foto da web: *"No no in quella web non ha molto senso lasciare i caricamenti delle foto. La togliamo proprio secondo me."*
- Tempistica web vs mobile: *"Forse è un po' più testabile sulla parte web che dici? Quella mobile come faccio a testarla?"*

**Assistente:**
- Rimosso il caricamento foto dal web (le foto caricate da mobile vengono comunque mostrate nella griglia).
- Invertite le fasi: **prima Backend + Web Angular (Fase 1)** per validare subito tutto via browser e Playwright, **poi App Flutter (Fase 2)**.
- Aggiunta strategia di test (§11): xUnit + Testcontainers per il backend, Playwright per il web, `flutter test` + emulatore Android / telefono via USB per il mobile.

---

## 9. Target Android e rendering immagini (Opzione A)

**Utente:**
- iOS: *"No, è solo per completezza. Ho Android. Ma possiamo fare con Android Studio e via."*
- *"Una nota: siccome ho scelto l'opzione A per le immagini, nella griglia 'La mia collezione' appare un link dove devo cliccare o viene renderizzata l'immagine del link?"*

**Assistente:**
- **Viene renderizzata direttamente l'immagine**, non un link testuale: `CachedNetworkImage` su Flutter (con cache locale su disco) e `<img [ngSrc]="..." loading="lazy">` su Angular. TCGdex offre qualità `low.webp` per la griglia e `high.webp` per il dettaglio, con fallback segnaposto in caso di errore.
- Target mobile iniziale fissato su **solo Android** (iOS valutato in futuro). Architettura completa senza domande aperte.
