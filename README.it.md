🇮🇹 Italiano · 🇬🇧 [English](README.md)

# Library Manager

Web app full stack per una biblioteca pubblica: un catalogo consultabile da chiunque, con la disponibilità in tempo reale delle copie della biblioteca, e una REST API per gestire libri, generi, prestiti, utenti e ruoli.

Progetto realizzato durante il corso di [EPICODE Institute of Technology](https://epicode.com): REST API in Spring Boot + SPA in React. L'interfaccia e il codice sono in italiano.

<p align="center">
  <img src="docs/screenshots/home.png" alt="Home con casella di ricerca e vetrina dei libri" width="100%">
</p>

<table>
  <tr>
    <td><img src="docs/screenshots/catalog.png" alt="Griglia del catalogo con i badge di disponibilità"></td>
    <td><img src="docs/screenshots/book-detail.png" alt="Dettaglio di un libro con le copie disponibili in biblioteca"></td>
  </tr>
  <tr>
    <td align="center">Catalogo con i badge di disponibilità</td>
    <td align="center">Dettaglio libro: 3 copie disponibili su 4</td>
  </tr>
  <tr>
    <td><img src="docs/screenshots/catalog-filtered.png" alt="Catalogo filtrato per genere in tema scuro"></td>
    <td><img src="docs/screenshots/how-it-works.png" alt="Sezione della home che descrive le regole dei prestiti"></td>
  </tr>
  <tr>
    <td align="center">Filtro per genere (tema scuro)</td>
    <td align="center">Le regole dei prestiti nella home</td>
  </tr>
</table>

<sub>Gli screenshot usano titoli, autori e ISBN inventati, con copertine segnaposto generate.</sub>

## Funzionalità

Il progetto è diviso in due fasi: la **REST API del backend è completa**, mentre il **frontend copre per ora la parte pubblica** (home e catalogo). Login, prestiti e area admin non sono ancora nell'interfaccia: oggi si usano tramite API, per esempio con la collection Postman (vedi [Roadmap](#roadmap)).

### Frontend (disponibile nell'interfaccia)

**Catalogo**
- Il catalogo arriva dalla Search API di [Open Library](https://openlibrary.org): ricerca libera (titolo, autore, ISBN...), filtri per genere e periodo, interruttore "solo con copertina" e ordinamento (più letti, più apprezzati, pertinenza, più recenti, meno recenti, titolo A-Z).
- Ogni pagina di risultati viene incrociata per ISBN con i libri della biblioteca (`POST /api/book/byIsbn`): i titoli posseduti mostrano il badge **Disponibile** o **In prestito**.
- Finestra di dettaglio con descrizione e soggetti da Open Library, più le copie della biblioteca: disponibili/totali, editore, anno, edizione, rilegatura e ISBN di ogni edizione posseduta.
- Scorrimento infinito su una griglia virtualizzata, caricata a blocchi da 30.
- Filtri, ricerca e ordinamento stanno nell'URL: ogni ricerca si può salvare nei preferiti e condividere.
- Se il backend non risponde il catalogo continua a funzionare, solo senza la disponibilità.

**Home e layout**
- Hero con una casella di ricerca che apre il catalogo, e una "vetrina" di titoli popolari.
- Sezione "Come funziona" che spiega le regole dei prestiti implementate nel backend.
- Tema chiaro e scuro: un interruttore nell'header salva la scelta.

**Cosa descrive la home e dove funziona oggi**

| Card della home | Stato |
|---|---|
| Cerca nel catalogo, controlla la disponibilità, sfoglia per genere | **Interfaccia**: pagina del catalogo, con il filtro per genere |
| Scegli la durata (breve, media, lunga) | **API**: il bibliotecario apre il prestito con `durata` = `BREVE`, `MEDIA`, `LUNGA` |
| Proroga il prestito una volta | **API**: il bibliotecario lo proroga con `PATCH /api/prestiti/ExtendPrestito` |
| Segui i tuoi prestiti | **API**: `GET /api/prestiti/UserPrestiti`; la pagina è nella roadmap |

Le voci di menu **Generi**, **I miei prestiti**, **Accedi** e **Registrati** aprono per ora il segnaposto "Pagina in costruzione".

### Backend (disponibile tramite API)

**Autenticazione**
- Registrazione e login JWT stateless (HS256); password cifrate con BCrypt. Email inesistente e password errata restituiscono lo stesso errore.
- Ogni token emesso viene salvato sul server come hash SHA-256: logout e refresh lo revocano, e i token sconosciuti o revocati vengono rifiutati anche se la firma è valida.
- `GET /api/user/me` restituisce profilo e ruoli.

**Ruoli**
- Tre ruoli: `SuperUser`, `Admin` (bibliotecario) e `User` (lettore), contenuti nel JWT e verificati con `@PreAuthorize` su ogni endpoint.
- Un SuperUser viene creato al primo avvio dalle variabili d'ambiente. Solo il SuperUser può creare ruoli e assegnare o revocare `Admin`; farlo revoca i token attivi dell'utente, così i nuovi ruoli valgono dal login successivo.

**Libri e generi**
- Pubblico: elenco paginato, ricerca con filtri combinabili (testo, titolo, autore, casa editrice, genere, intervallo di anni, intervallo di prezzo, rilegatura, solo disponibili) e ordinamento su una whitelist di campi, ricerca per un massimo di 10.000 ISBN.
- Utenti autenticati: elenco dei generi.
- Admin: inserimento di un libro (se l'ISBN esiste già, le copie si sommano), aggiunta di copie, creazione di generi.

**Prestiti**
- Admin: apertura di un prestito per un utente (breve, medio o lungo), proroga una sola volta, chiusura. La chiusura restituisce la copia e calcola la penale per il ritardo.
- Le copie si prendono con un `UPDATE` condizionale: un libro non può mai essere prestato oltre le copie disponibili.
- Lo stato si calcola in lettura: `APERTO`, `IN_RITARDO`, `CHIUSO`.
- Ricerca con filtri (utente, libro, stato, prorogato, data di apertura, data di scadenza) per gli admin; i lettori vedono solo i propri prestiti.

**Regole configurabili**
- Le durate dei prestiti (7 / 15 / 23 giorni) e la penale (0,50 al giorno, fino a un massimo di 20,00) stanno nella tabella `costanti`, modificabile dal SuperUser. I valori vengono validati, e una riga mancante o non valida torna al default definito nel codice.

## Stack tecnologico

| Area | Tecnologia | Versione |
|---|---|---|
| Backend | Java | 25 |
| | Spring Boot (Web MVC, Security, Data JPA, Validation, Actuator) | 4.1.1 |
| | Spring Security OAuth2 Resource Server (JWT) | da Spring Boot |
| | PostgreSQL | 16 (usato in sviluppo) |
| | Lombok | da Spring Boot |
| Frontend | React | 19.2 |
| | TypeScript | 6.0 |
| | Vite | 8.2 |
| | React Router | 8.4 |
| | Radix Themes + Radix Icons | 3.3 / 1.3 |
| | Motion | 13.4 |
| | TanStack Virtual | 3.14 |
| Servizi esterni | Open Library Search e Covers API | |
| Deploy | Docker (immagine del backend), Render Blueprint | |

## Architettura

```
be/   REST API in Spring Boot
  controllers/   /api/user, /api/book, /api/generi, /api/prestiti, /api/role, /api/costanti
  services/      logica di business, emissione dei JWT, JPA Specifications per le ricerche
  repositories/  repository Spring Data JPA, aggiornamenti atomici delle copie
  entities/      Libro, Genere, Prestito, User, Ruolo, RuoloUtente, TokenJwt, Costante
  dto/           record di richiesta e risposta, con Bean Validation
  security/      SecurityConfig, hash dei token e validatore delle revoche
  config/        DataInitializer (ruoli, SuperUser, costanti), lettura di DATABASE_URL
  postman/       collection ed environment per provare tutta l'API

fe/   SPA in React + TypeScript
  src/lib/          client dell'API, client di Open Library, normalizzazione ISBN
  src/pages/home/   hero, vetrina, regole dei prestiti
  src/pages/catalogo/  catalogo, filtri, griglia virtualizzata, finestra di dettaglio
  src/components/   layout (header, footer), copertina del libro
  src/theme/        provider del tema chiaro/scuro
```

Alcune scelte tecniche:

- **Open Library come catalogo, la biblioteca come disponibilità.** Invece di un catalogo limitato ai libri nel database, il frontend cerca su Open Library e chiede al backend quali di quegli ISBN la biblioteca possiede. Gli ISBN-10 vengono convertiti in ISBN-13 sul client, e il backend salva l'ISBN come numero a 13 cifre. Le ricerche grandi vengono divise in query `IN` da 1.000 valori.
- **Nessun prestito oltre le copie.** Prendere una copia è un singolo `UPDATE ... WHERE copieDisponibili > 0`: se nessuna riga viene aggiornata il prestito viene rifiutato con `409`, quindi prestiti contemporanei non possono scendere sotto zero.
- **JWT revocabili.** Un JWT stateless da solo non si può "disconnettere". Qui un `OAuth2TokenValidator` personalizzato confronta ogni token con gli hash salvati all'emissione: logout, refresh e cambi di ruolo hanno effetto subito.
- **JPA Specifications per le ricerche.** Libri e prestiti si cercano costruendo una sola `Specification` dai filtri facoltativi. I campi di ordinamento passano da una whitelist, e quelli sconosciuti restituiscono `400`.
- **Catalogo leggero.** I risultati si caricano a blocchi da 30 e si disegnano con un virtualizzatore sulla finestra. Le richieste in corso vengono annullate quando cambiano i filtri, e i duplicati che Open Library ripete tra le pagine vengono saltati.
- **Configurazione pronta per Render.** Render fornisce `DATABASE_URL` nel formato `postgresql://utente:password@host/db`, che Spring non legge: `DatabaseUrl` la converte in JDBC prima dell'avvio. Le tabelle stanno in uno schema dedicato (`DB_SCHEMA`), così il database gratuito può essere condiviso con un altro progetto.

## Avvio in locale

### Prerequisiti

- JDK 25
- Node.js 20.19 o successivo
- PostgreSQL
- Connessione a internet: catalogo e copertine arrivano da Open Library

### 1. Clona la repo

```bash
git clone https://github.com/TheRedKardinal/library-manager.git
cd library-manager
```

### 2. Database

```bash
createdb -U postgres biblioteca
```

### 3. Backend

```bash
cd be
cp .env.example .env     # poi compila i valori (vedi sotto)
./mvnw spring-boot:run   # Windows: mvnw.cmd spring-boot:run
```

L'API parte su `http://localhost:8080`. Lo schema viene creato in automatico, e ruoli, costanti e SuperUser vengono creati al primo avvio.

### 4. Frontend

```bash
cd fe
cp .env.example .env     # VITE_API_URL può restare vuota in sviluppo
npm install
npm run dev
```

L'app si apre su `http://localhost:5173`. In sviluppo Vite inoltra `/api` al backend, quindi non serve configurare il CORS.

Su Windows, `avvia.cmd` nella radice del progetto controlla che PostgreSQL sia in ascolto, installa le dipendenze del frontend se mancano e avvia entrambi i server.

### 5. Prova l'API con Postman

La biblioteca parte vuota: libri, generi, utenti e prestiti si creano tramite API. Il modo più rapido è la collection Postman in `be/postman/`:

1. Importa `Biblioteca.postman_collection.json` e `Biblioteca.postman_environment.json`.
2. Seleziona l'environment **Biblioteca - Local** e imposta `superuserPassword` con il valore di `SUPERUSER_PASSWORD`.
3. Esegui le richieste in ordine, oppure l'intera collection con il Runner. Le cartelle coprono login, utenti, generi, libri, prestiti, ruoli, costanti e logout: 45 richieste, ognuna con i suoi test. Le richieste di login salvano il JWT per quelle successive.

Per vedere un libro come disponibile nel catalogo, inseriscilo con l'ISBN di un'edizione reale presente su Open Library.

### Docker (solo backend)

`be/Dockerfile` è una build multi-stage (JDK 25 per compilare, JRE 25 per eseguire) usata da Render. Si può usare anche per avviare il backend in locale:

```bash
docker build -t library-manager-be ./be
docker run -p 8080:8080 --env-file be/.env library-manager-be
```

Dentro il container `localhost` è il container stesso: punta `DB_URL` verso l'host, per esempio `jdbc:postgresql://host.docker.internal:5432/biblioteca`.

## Eseguire i test

```bash
cd be
./mvnw test
```

C'è un solo smoke test (`DemoApplicationTests`) che verifica l'avvio del contesto Spring. Si collega al database PostgreSQL configurato in `DB_URL`, quindi il database deve essere attivo.

## Deploy

Il progetto è pronto per [Render](https://render.com) ma **non è online**: per provarlo, avvialo in locale seguendo [Avvio in locale](#avvio-in-locale).

`render.yaml` è un Blueprint con due servizi:

- **Backend**: web service Docker costruito da `be/Dockerfile`, con `/actuator/health` come health check. La JVM è limitata con `-XX:MaxRAMPercentage=75` per stare nei 512 MB del piano gratuito.
- **Frontend**: sito statico costruito con `npm ci && npm run build`, con un rewrite verso `index.html` per il routing lato client.

Il Blueprint non crea un database: usa un PostgreSQL di Render già esistente tramite `DATABASE_URL`, con le tabelle nello schema `biblioteca`. `JWT_SECRET` viene generata da Render; le altre variabili si inseriscono nella dashboard. `VITE_API_URL` viene letta durante la build, quindi cambiarla richiede un nuovo deploy del frontend.

## Variabili d'ambiente

Modelli: `be/.env.example` e `fe/.env.example`. I file `.env` sono ignorati da git. Se una variabile manca, il backend usa i default di `application.properties`, validi solo per lo sviluppo: fuori dal tuo computer imposta valori reali.

**Backend (`be/.env` in locale, dashboard di Render in produzione)**

| Variabile | Scopo |
|---|---|
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | Connessione a PostgreSQL (locale) |
| `DATABASE_URL` | Connection string di Render; se presente, sostituisce le tre variabili sopra |
| `DB_SCHEMA` | Schema delle tabelle (default `public`, `biblioteca` su Render) |
| `JWT_SECRET` | Chiave per firmare i JWT (almeno 32 caratteri) |
| `JWT_EXPIRATION_MINUTES` | Durata del token in minuti |
| `SUPERUSER_EMAIL`, `SUPERUSER_PASSWORD` | SuperUser creato al primo avvio |
| `ALLOWED_ORIGIN` o `CORS_ALLOWED_ORIGINS` | Origini del frontend ammesse dal CORS, separate da virgola |
| `PORT` | Porta HTTP (impostata da Render, default 8080) |

**Frontend (`fe/.env`)**

| Variabile | Scopo |
|---|---|
| `VITE_API_URL` | Indirizzo base del backend in produzione; vuota in sviluppo, dove Vite inoltra `/api` |

## Roadmap

Il backend supporta già queste funzionalità; il prossimo passo è la loro interfaccia:

- Pagine di login e registrazione, con il JWT conservato nel client.
- **I miei prestiti**: prestiti aperti, in ritardo e chiusi con le date di riconsegna, da `GET /api/prestiti/UserPrestiti`.
- Pagina dei generi.
- Area admin per i bibliotecari: inserimento di libri e copie, apertura, proroga e chiusura dei prestiti, e per il SuperUser gestione delle costanti e dei ruoli admin.

## Autore

Claudio Giasi: [GitHub](https://github.com/TheRedKardinal) · [LinkedIn](https://www.linkedin.com/in/claudio-giasi-8836b5243/)
