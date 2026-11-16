# Library Manager

A full-stack app for a public library: a catalog anyone can browse, with live availability of the library's copies, and a REST API to manage books, genres, loans, users and roles.

Built as a course project at [EPICODE Institute of Technology](https://epicode.com): Spring Boot REST API + React SPA. The UI and the codebase are in Italian (`Libro` = book, `Genere` = genre, `Prestito` = loan).

<p align="center">
  <img src="docs/screenshots/home.png" alt="Home page with search box and book showcase" width="100%">
</p>

<table>
  <tr>
    <td><img src="docs/screenshots/catalog.png" alt="Catalog grid with availability badges"></td>
    <td><img src="docs/screenshots/book-detail.png" alt="Book detail with copies available in the library"></td>
  </tr>
  <tr>
    <td align="center">Catalog with availability badges</td>
    <td align="center">Book detail: 3 of 4 copies available</td>
  </tr>
  <tr>
    <td><img src="docs/screenshots/catalog-filtered.png" alt="Catalog filtered by genre in dark theme"></td>
    <td><img src="docs/screenshots/how-it-works.png" alt="Home section describing the lending rules"></td>
  </tr>
  <tr>
    <td align="center">Genre filter (dark theme)</td>
    <td align="center">Lending rules on the home page</td>
  </tr>
</table>

<sub>Screenshots use invented titles, authors and ISBNs with generated placeholder covers.</sub>

## Features

The project is split in two stages: the **backend API is complete**, while the **frontend currently covers the public side** (home and catalog). Login, loans and admin screens are not in the UI yet: today they are used through the API, for example with the Postman collection (see [Roadmap](#roadmap)).

### Frontend (available in the UI)

**Catalog**
- The catalog comes from the [Open Library](https://openlibrary.org) Search API: free-text search (title, author, ISBN...), genre and period filters, "only with cover" toggle and sorting (most read, top rated, relevance, newest, oldest, title A-Z).
- Each page of results is matched by ISBN against the library's books (`POST /api/book/byIsbn`): titles the library owns get an **Available** or **On loan** badge (`Disponibile` / `In prestito`).
- Detail dialog with description and subjects from Open Library, plus the library's copies: available/total, publisher, year, edition, binding and ISBN of each owned edition.
- Infinite scroll on a virtualized grid, loaded in blocks of 30.
- Filters, search and sorting live in the URL, so every search can be bookmarked and shared.
- If the backend is unreachable the catalog keeps working, just without availability.

**Home and layout**
- Hero with a search box that opens the catalog, and a "showcase" shelf of popular titles.
- "How it works" section (`Come funziona`) that explains the lending rules implemented in the backend.
- Light and dark theme: a toggle in the header saves the choice.

**What the home page describes, and where it works today**

| Home card | Status |
|---|---|
| Search the catalog, check availability, browse by genre | **UI**: catalog page, with the genre filter |
| Choose the loan length (short, medium, long) | **API**: the librarian opens the loan with `durata` = `BREVE`, `MEDIA`, `LUNGA` |
| Extend a loan once | **API**: the librarian extends it with `PATCH /api/prestiti/ExtendPrestito` |
| Follow your loans | **API**: `GET /api/prestiti/UserPrestiti`; the UI page is on the roadmap |

The menu entries **Generi**, **I miei prestiti** (my loans), **Accedi** (log in) and **Registrati** (sign up) currently open a "page under construction" placeholder.

### Backend API (available through the API)

**Authentication**
- Registration and stateless JWT login (HS256); passwords hashed with BCrypt. Wrong email and wrong password return the same error.
- Every issued token is stored server-side as a SHA-256 hash: logout and refresh revoke it, and unknown or revoked tokens are rejected even if their signature is valid.
- `GET /api/user/me` returns the profile and roles.

**Roles**
- Three roles: `SuperUser`, `Admin` (librarian) and `User` (reader), carried in the JWT and checked with `@PreAuthorize` on every endpoint.
- A SuperUser is created on first startup from environment variables. Only the SuperUser can create roles and grant or revoke `Admin`; doing so revokes the user's active tokens, so the new roles apply at the next login.

**Books and genres**
- Public: paginated list, search with combinable filters (text, title, author, publisher, genre, year range, price range, binding, only available) and sorting on a whitelist of fields, lookup by up to 10,000 ISBNs.
- Logged-in users: list of genres.
- Admin: create a book (if the ISBN already exists, the copies are added to it), add copies, create genres.

**Loans**
- Admin: open a loan for a user (short, medium or long), extend it once, close it. Closing returns the copy and computes the late fee.
- Copies are taken with a conditional `UPDATE`, so a book can never be lent beyond its available copies.
- Status is computed on read: `APERTO` (open), `IN_RITARDO` (overdue), `CHIUSO` (closed).
- Search with filters (user, book, status, extended, opening date, due date) for admins; readers see only their own loans.

**Configurable rules**
- Loan lengths (7 / 15 / 23 days) and the late fee (0.50 per day, capped at 20.00) live in a `costanti` (constants) table that the SuperUser can edit. Values are validated, and a missing or invalid row falls back to the default in the code.

## Tech stack

| Area | Technology | Version |
|---|---|---|
| Backend | Java | 25 |
| | Spring Boot (Web MVC, Security, Data JPA, Validation, Actuator) | 4.1.1 |
| | Spring Security OAuth2 Resource Server (JWT) | from Spring Boot |
| | PostgreSQL | 16 (used in development) |
| | Lombok | from Spring Boot |
| Frontend | React | 19.2 |
| | TypeScript | 6.0 |
| | Vite | 8.2 |
| | React Router | 8.4 |
| | Radix Themes + Radix Icons | 3.3 / 1.3 |
| | Motion | 13.4 |
| | TanStack Virtual | 3.14 |
| External services | Open Library Search and Covers API | |
| Deployment | Docker (backend image), Render Blueprint | |

## Architecture

```
be/   Spring Boot REST API
  controllers/   /api/user, /api/book, /api/generi, /api/prestiti, /api/role, /api/costanti
  services/      business logic, JWT issuing, JPA Specifications for searches
  repositories/  Spring Data JPA repositories, atomic copy updates
  entities/      Libro, Genere, Prestito, User, Ruolo, RuoloUtente, TokenJwt, Costante
  dto/           request and response records, with Bean Validation
  security/      SecurityConfig, token hashing and revocation validator
  config/        DataInitializer (roles, SuperUser, constants), DATABASE_URL parsing
  postman/       collection and environment to try the whole API

fe/   React + TypeScript SPA
  src/lib/          API client, Open Library client, ISBN normalization
  src/pages/home/   hero, showcase, lending rules
  src/pages/catalogo/  catalog, filters, virtualized grid, detail dialog
  src/components/   layout (header, footer), book cover
  src/theme/        light/dark theme provider
```

Some technical choices:

- **Open Library as the catalog, the library as availability.** Instead of a catalog limited to the books in the database, the frontend searches Open Library and asks the backend which of those ISBNs the library owns. ISBN-10s are converted to ISBN-13 on the client, and the backend stores the ISBN as a 13-digit number. Large lookups are split into `IN` queries of 1,000 values.
- **No overbooking.** Taking a copy is a single `UPDATE ... WHERE copieDisponibili > 0`: if no row is updated the loan is refused with `409`, so concurrent loans cannot go below zero.
- **Revocable JWTs.** A stateless JWT cannot be "logged out" by itself. Here a custom `OAuth2TokenValidator` checks every token against the hashes stored at issue time, which makes logout, refresh and role changes take effect immediately.
- **JPA Specifications for searches.** Books and loans are searched by building one `Specification` from the optional filters. Sort fields are mapped through a whitelist, and unknown fields return `400`.
- **Light catalog.** Results are loaded in blocks of 30 and rendered with a window virtualizer. Requests in flight are aborted when the filters change, and duplicates returned by Open Library across pages are skipped.
- **Render-ready configuration.** Render provides `DATABASE_URL` as `postgresql://user:pass@host/db`, which Spring does not read: `DatabaseUrl` converts it to JDBC before startup. The tables live in a dedicated schema (`DB_SCHEMA`), so the free database can be shared with another project.

## Getting started

### Prerequisites

- JDK 25
- Node.js 20.19 or later
- PostgreSQL
- Internet access: the catalog and the covers come from Open Library

### 1. Clone

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
cp .env.example .env     # then fill in the values (see below)
./mvnw spring-boot:run   # Windows: mvnw.cmd spring-boot:run
```

The API starts on `http://localhost:8080`. The schema is created automatically, and roles, constants and the SuperUser are created on first startup.

### 4. Frontend

```bash
cd fe
cp .env.example .env     # VITE_API_URL can stay empty in development
npm install
npm run dev
```

The app opens on `http://localhost:5173`. In development Vite proxies `/api` to the backend, so no CORS setup is needed.

On Windows, `avvia.cmd` in the project root checks that PostgreSQL is listening, installs the frontend dependencies if needed and starts both servers.

### 5. Try the API with Postman

The library starts empty: books, genres, users and loans are created through the API. The quickest way is the Postman collection in `be/postman/`:

1. Import `Biblioteca.postman_collection.json` and `Biblioteca.postman_environment.json`.
2. Select the **Biblioteca - Local** environment and set `superuserPassword` to the value of `SUPERUSER_PASSWORD`.
3. Run the requests in order, or the whole collection with the Runner. The folders cover login, users, genres, books, loans, roles, constants and logout: 45 requests, each with its own tests. The login requests save the JWT for the following ones.

To see a book as available in the catalog, add it with the ISBN of a real edition that Open Library knows.

### Docker (backend only)

`be/Dockerfile` is a multi-stage build (JDK 25 to compile, JRE 25 to run) used by Render. It can also run the backend locally:

```bash
docker build -t library-manager-be ./be
docker run -p 8080:8080 --env-file be/.env library-manager-be
```

Inside the container `localhost` is the container itself: point `DB_URL` to the host, for example `jdbc:postgresql://host.docker.internal:5432/biblioteca`.

## Running tests

```bash
cd be
./mvnw test
```

There is one smoke test (`DemoApplicationTests`) that checks that the Spring context starts. It connects to the PostgreSQL database configured in `DB_URL`, so the database must be running.

## Deployment

The project is ready for [Render](https://render.com) but is **not deployed online**: to try it, run it locally by following [Getting started](#getting-started).

`render.yaml` is a Blueprint with two services:

- **Backend**: a Docker web service built from `be/Dockerfile`, with `/actuator/health` as the health check. The JVM is limited with `-XX:MaxRAMPercentage=75` to fit the free plan's 512 MB.
- **Frontend**: a static site built with `npm ci && npm run build`, with a rewrite to `index.html` for client-side routing.

The Blueprint does not create a database: it uses an existing Render PostgreSQL through `DATABASE_URL`, with the tables in the `biblioteca` schema. `JWT_SECRET` is generated by Render; the other variables are entered in the dashboard. `VITE_API_URL` is read at build time, so changing it requires a new deploy of the frontend.

## Environment variables

Templates: `be/.env.example` and `fe/.env.example`. The `.env` files are git-ignored. When a variable is missing, the backend falls back to development-only defaults in `application.properties`; set real values outside your machine.

**Backend (`be/.env` locally, Render dashboard in production)**

| Variable | Purpose |
|---|---|
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL connection (local) |
| `DATABASE_URL` | Render connection string; when set, it replaces the three variables above |
| `DB_SCHEMA` | Schema for the tables (default `public`, `biblioteca` on Render) |
| `JWT_SECRET` | Key used to sign JWTs (at least 32 characters) |
| `JWT_EXPIRATION_MINUTES` | Token lifetime in minutes |
| `SUPERUSER_EMAIL`, `SUPERUSER_PASSWORD` | SuperUser created on first startup |
| `ALLOWED_ORIGIN` or `CORS_ALLOWED_ORIGINS` | Frontend origins allowed by CORS, comma-separated |
| `PORT` | HTTP port (set by Render, default 8080) |

**Frontend (`fe/.env`)**

| Variable | Purpose |
|---|---|
| `VITE_API_URL` | Backend base URL in production; empty in development, where Vite proxies `/api` |

## Roadmap

The backend already supports these features; the next step is their UI:

- Login and registration pages, with the JWT kept in the client.
- **My loans** (`I miei prestiti`): open, overdue and closed loans with due dates, from `GET /api/prestiti/UserPrestiti`.
- Genres page.
- Admin area for librarians: add books and copies, open, extend and close loans, and for the SuperUser manage constants and admin roles.

## Author

Claudio Giasi: [GitHub](https://github.com/TheRedKardinal) · [LinkedIn](https://www.linkedin.com/in/claudio-giasi-8836b5243/)
