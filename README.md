<div align="center">

# LoRent

**Piattaforma di gestione noleggi e flotte veicoli — moderna, veloce, multi-dispositivo.**

[![Flutter](https://img.shields.io/badge/Flutter-3.9%2B-02569B?logo=flutter&logoColor=white)](https://flutter.dev)
[![Node.js](https://img.shields.io/badge/Node.js-Express-339933?logo=node.js&logoColor=white)](https://nodejs.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Neon-4169E1?logo=postgresql&logoColor=white)](https://neon.tech)
[![Next.js](https://img.shields.io/badge/Next.js-15-000000?logo=next.js&logoColor=white)](https://nextjs.org)

</div>

---

## Indice

- [Panoramica](#panoramica)
- [Architettura](#architettura)
- [Funzionalità principali](#funzionalità-principali)
- [Stack tecnologico](#stack-tecnologico)
- [Modello dati (sintesi)](#modello-dati-sintesi)
- [Scelte progettuali](#scelte-progettuali)
- [Autore](#autore)

---

## Panoramica

**LoRent** è una piattaforma full-stack pensata per aziende di noleggio veicoli e gestione flotte aziendali. Consente di gestire veicoli, dipendenti, clienti (privati e aziendali), prenotazioni e scadenze documentali da un'unica applicazione moderna. Attualmente in sviluppo per **iOS e Android**, con estensione prevista a **web e desktop**.

Il progetto comprende tre prodotti complementari:

1. **App client** (Flutter) — l'interfaccia usata quotidianamente da gestori, dipendenti e clienti. Mobile-first, con codebase pronto all'estensione multipiattaforma.
2. **API backend** (Node.js / Express + PostgreSQL) — cuore logico, autenticazione JWT, validazione, business rules.
3. **Sito vetrina** (Next.js) — landing page pubblica con presentazione del prodotto, features e roadmap.

> LoRent è un progetto personale sviluppato end-to-end: design, architettura, backend, frontend, DevOps e product management.

---

## Architettura

```
┌─────────────────────┐      ┌─────────────────────┐
│  Flutter App        │      │  Express API        │      ┌──────────────────┐
│  (iOS · Android)    │◄────►│  REST + JWT         │◄────►│  PostgreSQL      │
│                     │ HTTPS│  Rate limiting      │  pg  └──────────────────┘
└─────────────────────┘      │  Nodemailer         │
                             └──────────┬──────────┘
                                        │
                                        ▼
                                  ┌──────────┐
                                  │  Sentry  │   (monitoring · replay · perf)
                                  └──────────┘

┌─────────────────────┐
│  Next.js website    │   (landing, features, roadmap, form contatti)
└─────────────────────┘
```

- **Separation of concerns** netta fra layer di presentazione (Flutter), dominio (Express), persistenza (PostgreSQL).
- **Autenticazione stateless** con JWT e secure storage lato client.
- **Modalità manutenzione** lato server (`MAINTENANCE_MODE`) con risposta 503 intercettata dall'`ApiClient` Flutter per mostrare UI dedicata senza crash.
- **Osservabilità** tramite Sentry (error tracking, session replay, performance).

---

## Funzionalità principali

### Gestione veicoli
- Anagrafica completa (targa, marca, modello, tipologia, foto).
- Stati dinamici (disponibile, in noleggio, in manutenzione, fermo).
- Scadenze documentali: assicurazione, bollo, revisione.
- Storico noleggi per veicolo.

### Prenotazioni & Calendario
- Calendario interattivo con visualizzazione per veicolo / dipendente / periodo.
- Validazione automatica sovrapposizioni e conflitti.
- Permessi granulari (chi può prenotare cosa).
- Dettaglio prenotazione con cliente, veicolo, dipendente assegnato, note.

### Clienti
- Doppio binario **privati** e **aziende**, con campi e validazioni dedicate (P.IVA, CF, patente).
- Archivio documenti, storico interventi, ricerca veloce.

### Azienda & Dipendenti
- Pannello di gestione azienda (dati fiscali, logo, preferenze).
- Anagrafica dipendenti con ruoli e permessi.
- Onboarding guidato alla prima attivazione.

### Autenticazione
- Registrazione, login, logout.
- Reset password via email (Nodemailer).
- Rate limiting su endpoint sensibili.
- Cambio password in-app.

### UX
- Tema coerente con palette dedicata (navy / petrol / beige).
- Shimmer loaders, animazioni fluide, feedback immediato.
- Cache locale con invalidazione controllata.
- Gestione offline / riconnessione (`connectivity_plus`).

---

## Stack tecnologico

### Client — `lib/`
| Area | Tecnologia |
|---|---|
| Framework | Flutter 3.9+ (Dart) |
| Networking | `http` + `ApiClient` custom con gestione errori tipizzati |
| Storage | `flutter_secure_storage` (token) · `shared_preferences` (preferenze) |
| Stato | State management leggero basato su `StatefulWidget` + repository pattern |
| UX | `shimmer`, `animations`, `intl` (locale `it_IT`), `url_launcher` |
| Observability | `sentry_flutter` (replay + performance) |

### Backend — `SERVER/`
| Area | Tecnologia |
|---|---|
| Runtime | Node.js (ESM) |
| Framework | Express 4 |
| Database | PostgreSQL (driver `pg`) su **Neon** (serverless) |
| Auth | `jsonwebtoken` + `bcrypt` |
| Security | `cors`, `express-rate-limit`, validazione input in `utils/validate.js` |
| Mail | `nodemailer` (reset password, notifiche) |
| Dev tools | `nodemon`, `dotenv` |

### Website — `lorent-website/`
| Area | Tecnologia |
|---|---|
| Framework | Next.js 15 (App Router) |
| Linguaggio | TypeScript |
| Styling | Tailwind CSS |
| Deploy | Vercel |

---

## Modello dati (sintesi)

Entità principali e relazioni:

```
azienda ─┬─< dipendenti
         ├─< veicoli ─< scadenze
         ├─< clienti_privati
         ├─< clienti_aziende
         └─< prenotazioni >─ veicoli
                           >─ clienti (privati|aziende)
                           >─ dipendenti (assegnatario)
```

- Ogni **azienda** è un tenant: tutti i dati sono isolati per `azienda_id`.
- **Permessi** applicati a livello middleware (`SERVER/middleware/permissions.js`) prima di ogni route sensibile.
- Le **prenotazioni** validano sovrapposizioni sia lato client (feedback immediato) sia lato server (fonte di verità).

---

## Scelte progettuali

- **Monorepo** per mantenere allineati contratti API e client senza frammentazione. Backend, app e sito evolvono insieme.
- **JWT stateless** al posto di sessioni: backend scalabile orizzontalmente senza sticky session.
- **Neon serverless** come database: zero manutenzione, cold start accettabile per il workload, costi proporzionali all'uso.
- **Mobile-first, multipiattaforma pronta**: sviluppo iniziale su iOS e Android con Flutter, scelto proprio per poter estendere a web e desktop senza riscrivere il codice.
- **Maintenance mode nativo** nell'API: permette deploy "con preavviso" senza rompere l'UX dell'app.
- **Sentry + session replay**: debug di bug in produzione senza dover riprodurre manualmente.
- **Italiano come lingua primaria**: il prodotto è pensato per il mercato italiano, compreso il formato date (`utils/italianDate.js`) e la validazione documenti (CF, P.IVA, patente).

---

## Autore

**Lorenzo Rimi**
Progetto full-stack sviluppato in autonomia: product design, architettura, implementazione e deploy.

- Website: [lorent.app](https://lorent.app) *(in pubblicazione)*
