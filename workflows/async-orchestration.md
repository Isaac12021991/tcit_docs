# Asynchronous Workflows & State Machine Specification

This document details the End-to-End (*E2E*) execution flows, the asynchronous processing state machine, and the Artificial Intelligence integration rules (**Gemini API Category 1 Privacy** and **Dynamic Random Mock**).

---

## 1. Post Lifecycle State Machine (`EstadoProcesamiento`)

Every record in the `Posts` table follows a deterministic lifecycle governed by the `EstadoProcesamiento` column:

```mermaid
stateDiagram-v2
    [*] --> PENDIENTE: POST /posts (Golang API)<br/>Generates UUIDv4 & inserts into Spanner
    PENDIENTE --> PROCESSING: GET /posts/pending (Symfony Worker)<br/>Dispatches ProcessPostAiBrief
    PROCESSING --> COMPLETADO: DML UPDATE in Spanner<br/>Persists DatosExtraidosIA
    PROCESSING --> ERROR: Unhandled exception after exhausting<br/>3 retries with Exponential Backoff
    COMPLETADO --> [*]: DELETE /posts/{id}
    PENDIENTE --> [*]: DELETE /posts/{id}
```

---

## 2. End-to-End Sequence Diagram

The following sequence diagram illustrates the chronological interactions across the User, the Angular SPA (`:4200`), the Golang REST API (`:8080`), the Cloud Spanner Emulator (`:9010` / `:9020`), and the Symfony CLI Worker (`app:process-posts`):

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Angular as Frontend SPA<br/>(Angular :4200)
    participant GoAPI as Backend REST API<br/>(Golang :8080)
    participant Spanner as Google Cloud Spanner<br/>(:9010 gRPC / :9020 REST)
    participant Symfony as Orchestrator Worker<br/>(Symfony CLI)
    participant Gemini as Gemini AI /<br/>Dynamic Mock

    rect rgb(235, 245, 255)
        Note over User,Spanner: Phase 1: Low-Latency Synchronous Creation
        User->>Angular: Completes form (/posts/new)<br/>Name, Description, Brief (<=300 chars) + Attachment
        Angular->>Angular: Validates Reactive Form & encodes file<br/>to Base64 (FileReader.readAsDataURL)
        Angular->>GoAPI: POST /posts (JSON camelCase)
        GoAPI->>GoAPI: Generates UUIDv4 & sets<br/>EstadoProcesamiento = 'PENDIENTE'
        GoAPI->>Spanner: ReadWriteTransaction: Insert(Posts) via gRPC (:9010)
        Spanner-->>GoAPI: Commit OK
        GoAPI-->>Angular: HTTP 201 Created (Post in PENDIENTE state)
        Angular-->>User: Redirects to /posts (Displays PENDIENTE badge)
    end

    rect rgb(240, 255, 240)
        Note over GoAPI,Gemini: Phase 2: Asynchronous Background Orchestration (OneStack Workflow)
        loop Polls every 5 seconds (app:process-posts)
            Symfony->>GoAPI: GET /posts/pending
            GoAPI->>Spanner: SELECT * FROM Posts WHERE EstadoProcesamiento = 'PENDIENTE'
            Spanner-->>GoAPI: Pending rows
            GoAPI-->>Symfony: JSON Array of Pending Posts
            opt Pending Posts exist
                Symfony->>Symfony: Dispatches ProcessPostAiBrief message to Bus
                Symfony->>Gemini: Analyzes Brief (Category 1 Privacy or array_rand Mock)
                Gemini-->>Symfony: "Objetivo: <obj> | Audiencia: <aud>"
                Symfony->>Spanner: POST :beginTransaction + :executeSql (UPDATE Posts) + :commit (:9020)
                Spanner-->>Symfony: Commit OK (EstadoProcesamiento = 'COMPLETADO')
            end
        end
    end

    rect rgb(255, 248, 235)
        Note over User,Spanner: Phase 3: Retrieval & In-Memory Local Filtering
        User->>Angular: Navigates to /posts
        Angular->>GoAPI: GET /posts (Single call during ngOnInit)
        GoAPI->>Spanner: SELECT * FROM Posts
        Spanner-->>GoAPI: All rows (COMPLETADO / PENDIENTE)
        GoAPI-->>Angular: Full JSON Array
        User->>Angular: Types in search box (Filter by Campaign Name)
        Angular->>Angular: Filters local array in browser memory (0 extra HTTP calls)
    end
```

---

## 3. AI Extraction Engine Specification in Symfony

The `ProcessPostAiBriefHandler` (`orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php`) operates in two modes depending on the environment configuration:

### 3.1. Production Mode (`GEMINI_API_KEY` configured)
Sends an HTTP `POST` request to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent` injecting mandatory enterprise data-governance headers (**Category 1 Privacy Setup**):

| HTTP Header | Value | Purpose |
| :--- | :--- | :--- |
| `X-Goog-Vertex-Data-Governance` | `category-1-no-train` | Prohibits using the payload for foundation model training or fine-tuning. |
| `X-Goog-Privacy-Disable-Training` | `true` | Disables prompt ingestion into product improvement pipelines. |
| `X-Goog-Privacy-Zero-Retention` | `true` | Enforces zero cache retention after inference completes. |

### 3.2. Local Development Mode (Dynamic Mock via `array_rand`)
When running locally with simulated AI integration (or without an API key), the worker randomly selects a realistic campaign objective and target audience on every execution using `array_rand()`:

- **Campaign Objectives Catalog (5 options):**
  1. `"Aumentar el brand awareness en un 20%"` *(Increase brand awareness by 20%)*
  2. `"Generar leads cualificados B2B"` *(Generate qualified B2B leads)*
  3. `"Impulsar ventas del e-commerce en Q3"` *(Boost Q3 e-commerce sales)*
  4. `"Lanzamiento de nuevo producto"` *(New product launch)*
  5. `"Retención y fidelización de usuarios activos"` *(Active user retention and loyalty)*
- **Target Audiences Catalog (5 options):**
  1. `"Jóvenes 18-24 años segmento gamer"` *(Youth aged 18–24, gaming segment)*
  2. `"Directores de tecnología (CTOs) en LATAM"` *(Chief Technology Officers in LATAM)*
  3. `"Público general urbano 25-45 años"` *(General urban audience aged 25–45)*
  4. `"Pequeñas y medianas empresas (PyMEs)"` *(Small and medium-sized enterprises - SMEs)*
  5. `"Compradores compulsivos de tecnología"` *(Tech early adopters and frequent buyers)*

**Output format persisted in `DatosExtraidosIA`:**
```text
Objetivo: <Random Objective> | Audiencia: <Random Audience>
```

---

## 4. Frontend Validation & Base64 Encoding Flow (`/frontend`)

The `PostFormComponent` enforces data integrity prior to invoking the Backend API:

1. **Reactive Form Validations (`Validators`):**
   - `nombre` (*Campaign Name*): `[Validators.required, Validators.maxLength(300)]`
   - `descripcion` (*Description*): `[Validators.required, Validators.maxLength(300)]`
   - `brief` (*Campaign Brief*): `[Validators.required, Validators.maxLength(300)]`
2. **Visual Feedback (`Angular Material`):**
   - Live character counters (`{{ control.value?.length || 0 }}/300`) rendered via `<mat-hint align="end">`.
   - Dedicated `<mat-error>` messages for both `required` and `maxlength` violations.
   - Submit button dynamically disabled via `[disabled]="form.invalid || enviando"`.
3. **Client-Side Base64 Packaging:**
   - Upon selecting a file in the browser, `FileReader.readAsDataURL(file)` encodes the binary into a Base64 Data URL and embeds it directly into the single `POST /posts` JSON request, avoiding intermediate upload endpoints.
