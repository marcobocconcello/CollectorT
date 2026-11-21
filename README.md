# CollectorT

App gratuita (web + mobile) per tracciare la propria collezione di carte Pokémon e l'andamento del suo valore nel tempo con prezzi di riferimento Cardmarket.

> **Stato attuale:** fase di progettazione architetturale completata. Nessun codice ancora scritto.

## Documentazione

- [docs/architettura.md](./docs/architettura.md) — Specifica architetturale completa (stack, modello dati, flussi, hosting k3s, strategia di test e fasi).
- [docs/chat.md](./docs/chat.md) — Cronologia della discussione che ha portato alle decisioni architetturali.

## Stack scelto

- **Backend:** .NET 10 (LTS), ASP.NET Core, EF Core, PostgreSQL 17+ (`pg_trgm`, `pgvector`)
- **Autenticazione:** ASP.NET Core Identity + OpenIddict (solo email/password nella v1)
- **Web:** Angular (senza ricerca/caricamento foto)
- **Mobile:** Flutter (target iniziale Android, con OCR on-device via ML Kit e foto utente)
- **Fonti dati:** TCGdex (catalogo internazionale + giapponese e prezzi Cardmarket `trend`)
- **Hosting:** Oracle Cloud Always Free + k3s (`arm64`)

---

## Come importare questa chat e il contesto in locale (Antigravity / assistente AI)

Le chat di Antigravity sono salvate localmente per singola macchina e non esiste un pulsante di importazione diretta da GitHub. Per riprendere esattamente da dove abbiamo lasciato con il tuo assistente in locale:

1. **Clona il repository:**
   ```bash
   git clone <URL-DEL-REPOSITORY>
   cd CollectorT
   ```

2. **Apri la cartella `CollectorT` come workspace** in Antigravity (o nel tuo IDE con assistente AI).

3. **Avvia una nuova chat** e invia un prompt iniziale che carica il contesto:
   > Leggi `docs/architettura.md` e `docs/chat.md` per allinearti su tutte le decisioni prese finora per CollectorT. Non modificare nulla finché non ti dico da quale fase partire.

4. A questo punto il tuo assistente avrà lo stesso contesto della progettazione originale e potrete continuare a raffinare l'architettura o iniziare lo sviluppo (Fase 0: raccolta dati e backend).
