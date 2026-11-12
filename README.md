# System Documentation: Campaigns & Briefs Management Platform (TCIT)

Welcome to the central technical documentation repository for the **Campaigns & Briefs Management** ecosystem (developed for the **TCIT** technical assessment by **Isaac Betancourt**). This directory centralizes the software architecture, Google Cloud Spanner DDL schema and data dictionary, REST API contracts, asynchronous workflow specifications, production readiness alerts, and system diagrams.

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
>
> For local development reproducibility, external cloud dependencies and paid LLM APIs are simulated out-of-the-box:
> - **Google Cloud Spanner:** Runs inside the local Docker Emulator (`localhost:9010` gRPC / `localhost:9020` REST) without IAM credentials.
> - **Gemini AI Extraction:** Uses a dynamic `array_rand()` fallback mock in Symfony (`5 Objectives x 5 Target Audiences`) until a live `GEMINI_API_KEY` is injected into `orchestrator/.env`.
> - **File Attachments (`Adjunto`):** Stored as client-encoded Base64 Data URLs in Cloud Spanner rather than uploading to a live Google Drive bucket.
>
> 👉 **See the full inventory and step-by-step production remediation guide in [`docs/MOCKED_DATA_ALERTS.md`](./MOCKED_DATA_ALERTS.md).**

---

## `/docs` Directory Structure & Index

```text
docs/
├── README.md                             # Master index, component map, and quick DDL reference
├── MOCKED_DATA_ALERTS.md                 # Inventory of mocked data, simulated connections & remediation
├── architecture/
│   └── system-design.md                  # System architecture, SoC principles, and layer breakdown
├── database/
│   └── data-dictionary.md                # Cloud Spanner DDL schema, ER diagram, and Data Dictionary
├── api/
│   └── api-contracts.md                  # REST API contracts (Go :8080 and Spanner REST :9020)
├── workflows/
│   └── async-orchestration.md            # State machine, E2E sequence diagram, and AI engine specs
└── diagrams/
    └── flowchart.mmd                     # Mermaid.js flowchart of the complete Post lifecycle
```

### Direct Links by Domain
1. **[Mocked Data, Simulated Connections & Production Readiness Guide](./MOCKED_DATA_ALERTS.md)**
   - Complete inventory of local mocks (Cloud Spanner Emulator, Gemini AI `array_rand` mock, Base64 attachments, development tokens) and actionable steps to transition to production.
2. **[System Architecture & Separation of Concerns](./architecture/system-design.md)**
   - Rationale behind the Clean Architecture pattern and asynchronous decoupling (*Strict Stack*).
   - Responsibilities of the Golang REST API (`:8080`), Symfony Orchestrator (`app:process-posts`), Angular SPA Frontend (`:4200`), and Cloud Spanner Emulator (`:9010` / `:9020`).
3. **[Database Schema (DDL) & Data Dictionary](./database/data-dictionary.md)**
   - Official DDL statements (`CREATE TABLE Posts`, `ALTER TABLE` migrations), Entity-Relationship diagram (`erDiagram`), data types, nullability constraints, `UUIDv4` anti-hotspotting strategy, and SQL/DML operation matrix.
4. **[REST API Contracts & Endpoint Specification](./api/api-contracts.md)**
   - Detailed specification for `POST /posts`, `GET /posts`, `GET /posts/pending`, and `DELETE /posts/{id}`, `camelCase` JSON schemas, HTTP status codes, and Cloud Spanner REST transactions.
5. **[Asynchronous Workflows & State Machine Specification](./workflows/async-orchestration.md)**
   - Lifecycle state machine (`PENDIENTE` -> `COMPLETADO` / `ERROR`), End-to-End sequence diagram, Gemini **Category 1 Privacy Setup** headers, dynamic random mock catalog (`array_rand`), and Angular reactive validations.
6. **[Post Lifecycle Flowchart (Mermaid.js)](./diagrams/flowchart.mmd)**
   - Mermaid source file illustrating the end-to-end interaction graph across all four services.

---

## High-Level Overview of the 4 Core Components

| Component | Directory | Core Technology Stack | Port / Interface | Primary Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **1. Frontend SPA** | `/frontend` | Angular 18 + TypeScript + Angular Material | `http://localhost:4200` | English user interface with routing (`/`, `/posts`, `/posts/new`), reactive form validations (`maxLength(300)`), client-side Base64 file encoding, and in-memory filtering. |
| **2. Backend REST API** | `/backend` | Golang 1.23 + Cloud Spanner SDK | `http://localhost:8080` | Low-latency REST API (`POST /posts`, `GET /posts`, `GET /posts/pending`, `DELETE /posts/{id}`). Immediate persistence without blocking on AI inference. |
| **3. Orchestrator Worker** | `/orchestrator` | PHP 8.2 + Symfony 7.1 (Messenger / CLI Worker) | CLI (`app:process-posts`) | Asynchronous background processing: polls `PENDIENTE` posts, executes AI brief extraction (Category 1 Privacy / dynamic mock), and updates state to `COMPLETADO`. |
| **4. Database** | `/database` | Google Cloud Spanner Emulator (Docker) | `:9010` (gRPC) / `:9020` (REST) | Distributed relational storage for the `Posts` table using `UUIDv4` primary keys. |

---

## How the Components Interact (End-to-End Summary)

1. **Client Creation (`Angular -> Golang`):**
   The user fills out the campaign form at `/posts/new` (Campaign Name, Description, Campaign Brief, and an optional attachment encoded in Base64) and submits a single `POST http://localhost:8080/posts` request.
2. **Immediate Persistence (`Golang -> Cloud Spanner`):**
   The Go backend generates a `UUIDv4`, inserts the record into the `Posts` table with `EstadoProcesamiento = 'PENDIENTE'`, and responds with `201 Created` in milliseconds, immediately freeing the client UI.
3. **Discovery & Polling (`Symfony -> Golang / Cloud Spanner`):**
   The Symfony CLI worker (`php bin/console app:process-posts`) periodically polls `GET http://localhost:8080/posts/pending` to fetch newly created records awaiting processing.
4. **Asynchronous Enrichment & Consolidation (`Symfony -> Cloud Spanner`):**
   For each pending post, the Symfony worker extracts the campaign objective and target audience via Gemini API (under *Category 1 Privacy* headers) or the dynamic random mock in local development. Finally, it updates `DatosExtraidosIA` and transitions `EstadoProcesamiento = 'COMPLETADO'` in Cloud Spanner.

---

## Quick Reference: DDL & Data Dictionary (`Posts`)

> For the Entity-Relationship diagram, detailed column business rules, and full transactional matrix, see **[`docs/database/data-dictionary.md`](./database/data-dictionary.md)**.

```sql
CREATE TABLE Posts (
    Id STRING(36) NOT NULL,
    Nombre STRING(255) NOT NULL,
    Descripcion STRING(MAX),
    Adjunto STRING(MAX),
    DatosExtraidosIA STRING(MAX),
    EstadoProcesamiento STRING(50)
) PRIMARY KEY (Id);
```

| Column | Spanner Type | Constraint | JSON Property | Description |
| :--- | :--- | :--- | :--- | :--- |
| `Id` | `STRING(36)` | `NOT NULL` (`PK`) | `id` | Auto-generated `UUIDv4` unique identifier created in Go to prevent write hot-spotting. |
| `Nombre` | `STRING(255)` | `NOT NULL` | `nombre` | Campaign name (required, maximum 300 characters enforced in UI). |
| `Descripcion` | `STRING(MAX)` | `NULL` | `descripcion` | General campaign description concatenated with the Campaign Brief. |
| `Adjunto` | `STRING(MAX)` | `NULL` | `adjunto` | Base64-encoded file attachment (`data:<mime>;base64,...`) or Google Drive URL. |
| `DatosExtraidosIA` | `STRING(MAX)` | `NULL` | `datosExtraidosIA` | Objective and target audience extracted by the Symfony worker (`Objetivo: ... \| Audiencia: ...`). |
| `EstadoProcesamiento` | `STRING(50)` | `NULL` | `estadoProcesamiento` | Asynchronous workflow state (`'PENDIENTE'`, `'COMPLETADO'`, `'ERROR'`). |

---

## Recommended Startup Order (Local Environment)

```powershell
# 1. Database (Cloud Spanner Emulator on ports 9010 and 9020)
cd database
docker compose up -d

# 2. Golang Backend REST API (Port 8080)
cd ..\backend
go run .

# 3. Symfony Orchestrator Worker (Background CLI Worker)
cd ..\orchestrator
php bin/console app:process-posts

# 4. Angular Frontend SPA (Port 4200)
cd ..\frontend
npx ng serve --port 4200
```
