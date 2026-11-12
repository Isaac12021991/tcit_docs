# REST API Contracts & Endpoint Specification

This document defines the HTTP/JSON communication contracts exposed by the **Golang Backend** (`http://localhost:8080`) and the internal transactional REST contracts consumed by the **Symfony Orchestrator** against the **Google Cloud Spanner Emulator REST API** (`http://localhost:9020`).

---

## 1. General Conventions of the Public API (`/backend`)

- **Base URL (Local):** `http://localhost:8080`
- **Data Exchange Format:** `application/json; charset=utf-8`
- **JSON Naming Convention:** Strict `camelCase` (`id`, `nombre`, `descripcion`, `adjunto`, `datosExtraidosIA`, `estadoProcesamiento`).
- **CORS Policy (Cross-Origin Resource Sharing):**
  - `Access-Control-Allow-Origin: *`
  - `Access-Control-Allow-Methods: GET, POST, DELETE, OPTIONS`
  - `Access-Control-Allow-Headers: Content-Type, Authorization`
  - Preflight `OPTIONS` requests respond immediately with `204 No Content`.

---

## 2. Canonical JSON Model (`Post`)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "nombre": "Q3 E-Commerce Growth Campaign",
  "descripcion": "Digital marketing campaign for Q3.\n\n--- BRIEF ---\nFocus on conversion rate optimization and cart recovery.",
  "adjunto": "data:application/pdf;base64,JVBERi0xLjQK...",
  "datosExtraidosIA": "Objetivo: Impulsar ventas del e-commerce en Q3 | Audiencia: Compradores compulsivos de tecnología",
  "estadoProcesamiento": "COMPLETADO"
}
```

---

## 3. Backend Endpoint Catalog (`http://localhost:8080`)

### 3.1. Create a New Campaign / Post
Persists a new Post in Cloud Spanner with an auto-generated `UUIDv4` identifier and initializes its status to `'PENDIENTE'` for asynchronous background processing by the Symfony worker.

- **Method & Path:** `POST /posts`
- **Primary Consumer:** Angular Frontend SPA (`/posts/new`)

#### Validation Rules (Frontend + Backend)
| JSON Field | Required | Frontend Constraint (Angular) | Backend Constraint (Go) |
| :--- | :--- | :--- | :--- |
| `nombre` | **Yes** | `Validators.required`, `Validators.maxLength(300)` | Non-empty (`strings.TrimSpace(nombre) != ""`) |
| `descripcion` | **Yes** (in UI) | `Validators.required`, `Validators.maxLength(300)` (on `descripcion` and `brief`) | Accepts long text (`STRING(MAX)`) |
| `adjunto` | Optional | Client-side Base64 encoding via `FileReader.readAsDataURL()` | Accepts Base64 string or empty (`STRING(MAX)`) |

#### Sample Request Body
```json
{
  "nombre": "Cloud Enterprise Launch 2026",
  "descripcion": "Regional launch across LATAM.\n\n--- BRIEF ---\nTarget CTOs and IT Directors looking for hybrid cloud modernization.",
  "adjunto": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA..."
}
```

#### HTTP Responses
- **`201 Created`**: Record successfully inserted into Cloud Spanner.
  ```json
  {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "nombre": "Cloud Enterprise Launch 2026",
    "descripcion": "Regional launch across LATAM.\n\n--- BRIEF ---\nTarget CTOs and IT Directors looking for hybrid cloud modernization.",
    "adjunto": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA...",
    "datosExtraidosIA": "",
    "estadoProcesamiento": "PENDIENTE"
  }
  ```
- **`400 Bad Request`**: Malformed JSON payload or empty `nombre` field.
  ```json
  {
    "error": "el campo 'nombre' es obligatorio"
  }
  ```
- **`500 Internal Server Error`**: Cloud Spanner connectivity or persistence error.
  ```json
  {
    "error": "error al guardar el post en Cloud Spanner"
  }
  ```

---

### 3.2. List All Campaigns / Posts
Returns the full collection of records stored in the `Posts` table. If no records exist, it returns an empty JSON array `[]` (never `null`).

- **Method & Path:** `GET /posts`
- **Primary Consumer:** Angular Frontend SPA (`PostListComponent.ngOnInit()`)

#### HTTP Responses
- **`200 OK`**:
  ```json
  [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "nombre": "Cloud Enterprise Launch 2026",
      "descripcion": "Regional launch across LATAM...",
      "adjunto": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA...",
      "datosExtraidosIA": "Objetivo: Generar leads cualificados B2B | Audiencia: Directores de tecnología (CTOs) en LATAM",
      "estadoProcesamiento": "COMPLETADO"
    }
  ]
  ```
- **`500 Internal Server Error`**:
  ```json
  {
    "error": "error al obtener los posts desde Cloud Spanner"
  }
  ```

---

### 3.3. List Pending Campaigns for Background Processing
Queries Cloud Spanner exclusively for records where `EstadoProcesamiento = 'PENDIENTE'`. This endpoint feeds the polling loop of the Symfony Orchestrator.

- **Method & Path:** `GET /posts/pending`
- **Primary Consumer:** Symfony Orchestrator Worker (`php bin/console app:process-posts`)

#### HTTP Responses
- **`200 OK`**:
  ```json
  [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "nombre": "Cloud Enterprise Launch 2026",
      "descripcion": "Regional launch across LATAM...",
      "adjunto": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA...",
      "datosExtraidosIA": "",
      "estadoProcesamiento": "PENDIENTE"
    }
  ]
  ```
- **`500 Internal Server Error`**:
  ```json
  {
    "error": "error al obtener los posts pendientes desde Cloud Spanner"
  }
  ```

---

### 3.4. Delete a Campaign / Post by ID
Verifies record existence inside a Cloud Spanner `ReadWriteTransaction` and executes the primary-key deletion mutation.

- **Method & Path:** `DELETE /posts/{id}`
- **Path Parameters:**
  - `id` (`string`, required): `UUIDv4` of the Post to delete.
- **Primary Consumer:** Angular Frontend SPA (`PostListComponent`)

#### HTTP Responses
- **`200 OK`**:
  ```json
  {
    "mensaje": "post eliminado correctamente",
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
  }
  ```
- **`404 Not Found`**: Returned when the specified `id` does not exist in the `Posts` table.
  ```json
  {
    "error": "el post solicitado no existe"
  }
  ```
- **`500 Internal Server Error`**:
  ```json
  {
    "error": "error al eliminar el post en Cloud Spanner"
  }
  ```

---

## 4. Internal Contract: Symfony Orchestrator -> Cloud Spanner REST API (`:9020`)

Because local PHP environments on Windows may run without the native `ext-grpc` extension, Symfony's `ProcessPostAiBriefHandler` communicates directly with the Cloud Spanner v1 REST API (`http://localhost:9020/v1/projects/mi-proyecto-local/instances/mi-instancia-local/databases/posts_db`) in four atomic steps:

1. **Create Session (`POST /sessions`):** Acquires a temporary session identifier (`session.name`).
2. **Begin Transaction (`POST /{session}:beginTransaction`):** Requests a read-write transaction `{ "options": { "readWrite": {} } }`.
3. **Execute Parameterized DML (`POST /{session}:executeSql`):**
   ```json
   {
     "transaction": { "id": "<transaction_id>" },
     "sql": "UPDATE Posts SET DatosExtraidosIA = @datosExtraidosIA, EstadoProcesamiento = 'COMPLETADO' WHERE Id = @postId",
     "params": {
       "datosExtraidosIA": "Objetivo: Lanzamiento de nuevo producto | Audiencia: Jóvenes 18-24 años segmento gamer",
       "postId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
     },
     "paramTypes": {
       "datosExtraidosIA": { "code": "STRING" },
       "postId": { "code": "STRING" }
     }
   }
   ```
4. **Commit Transaction (`POST /{session}:commit`):** Permanently commits the state transition and releases the session (`DELETE /{session}`).
