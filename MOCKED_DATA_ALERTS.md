# Mocked Data, Simulated Connections & Production Readiness Guide

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
>
> Several external integrations, cloud infrastructure connections, and credential-dependent workflows in this repository are intentionally **mocked or emulated** to guarantee a self-contained, zero-cost, and reproducible local development environment for the TCIT technical assessment.
>
> **Do not deploy this system to a live production environment without completing the remediation steps documented below.**

---

## 1. Executive Summary Matrix of Simulated Elements

| # | Component / Integration | Current Local State (Simulated / Mocked) | Source File(s) Involved | Production Requirement |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Google Cloud Spanner** | Local Docker Emulator (`gcr.io/cloud-spanner-emulator/emulator:latest`) over unauthenticated gRPC (`:9010`) and REST (`:9020`). Data is ephemeral if the container volume is removed. | `database/docker-compose.yml`<br/>`backend/.env`<br/>`orchestrator/.env` | Live GCP Cloud Spanner instance, IAM Service Account (`GOOGLE_APPLICATION_CREDENTIALS`), and removal of `SPANNER_EMULATOR_HOST`. |
| **2** | **Google Gemini AI (LLM Extraction)** | Dynamic `array_rand()` fallback mock combining 5 static campaign objectives and 5 target audiences when `GEMINI_API_KEY` is empty. | `orchestrator/.env`<br/>`orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php` | Valid Google AI Studio / Vertex AI credential injected into `GEMINI_API_KEY` with **Category 1 Privacy** headers enabled. |
| **3** | **Google Drive / Binary Attachment Storage** | Client-side Base64 Data URL encoding (`FileReader.readAsDataURL`) persisted directly inside the `Adjunto STRING(MAX)` column in Cloud Spanner. | `frontend/src/app/components/post-form/post-form.component.ts`<br/>`orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php` | Google Drive API v3 (or Google Cloud Storage) integration via GCP Service Account to upload binary files and store only the canonical Drive URL in `Adjunto`. |
| **4** | **Symfony Messenger Transport & Secrets** | Synchronous/in-memory `sync://` message bus triggered by a CLI polling loop (`app:process-posts`) and a development dummy `APP_SECRET`. | `orchestrator/.env`<br/>`orchestrator/config/packages/messenger.yaml`<br/>`orchestrator/src/Command/ProcessPendingPostsCommand.php` | Managed daemon supervisor (systemd / Kubernetes CronJob / Cloud Run Jobs), cryptographically secure `APP_SECRET`, and OAuth2 Bearer token for Spanner REST calls. |
| **5** | **Frontend API Base URL, CORS & Edit Flow** | Hardcoded `http://localhost:8080/posts` in Angular, permissive `Access-Control-Allow-Origin: *` in Go, and client-side re-submission (`POST`) when editing a campaign. | `frontend/src/app/services/post.service.ts`<br/>`backend/internal/handler/post_handler.go` | Angular `environment.prod.ts` configuration, strict origin allowlist in Go CORS middleware, and a dedicated `PUT /posts/{id}` endpoint if in-place updates are required. |

---

## 2. Detailed Inventory & Actionable Remediation Plan

### 2.1. Google Cloud Spanner Connection (Local Emulator vs. Managed GCP Instance)

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
> Both the Golang Backend (`:8080`) and the Symfony Orchestrator currently target the local **Cloud Spanner Emulator** (`projects/mi-proyecto-local/instances/mi-instancia-local/databases/posts_db`) without TLS encryption or IAM authentication.

#### Current Simulated Behavior
- **Golang Backend (`backend/.env`):** Defines `SPANNER_EMULATOR_HOST=localhost:9010`. When the Google Cloud Go SDK (`cloud.google.com/go/spanner`) detects this environment variable, it bypasses OAuth2/IAM authentication and connects over plaintext gRPC to the local Docker container.
- **Symfony Orchestrator (`orchestrator/.env`):** Defines `SPANNER_REST_BASE_URL=http://localhost:9020/v1/projects/mi-proyecto-local/instances/mi-instancia-local/databases/posts_db` and executes unauthenticated REST calls (`:beginTransaction`, `:executeSql`, `:commit`).

#### Actionable Remediation for Production
1. **Provision Cloud Spanner in GCP:**
   Create a production instance and database in Google Cloud Platform and apply the DDL in `database/schema.sql`.
2. **Configure IAM Service Account:**
   Create a GCP Service Account with the `roles/spanner.databaseUser` role and mount its JSON key file (or use Workload Identity Federation in GKE / Cloud Run).
3. **Update `backend/.env` (Golang):**
   - **Remove** `SPANNER_EMULATOR_HOST`.
   - Set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json`.
   - Update `SPANNER_DSN` to the live resource path:
     ```dotenv
     PORT=8080
     SPANNER_DSN=projects/<PROD_GCP_PROJECT>/instances/<PROD_INSTANCE>/databases/posts_db
     GOOGLE_APPLICATION_CREDENTIALS=/etc/secrets/gcp-service-account.json
     ```
4. **Update `orchestrator/.env` and `ProcessPostAiBriefHandler.php` (Symfony):**
   - Point `SPANNER_REST_BASE_URL` to the production Google API endpoint:
     ```dotenv
     SPANNER_REST_BASE_URL=https://spanner.googleapis.com/v1/projects/<PROD_GCP_PROJECT>/instances/<PROD_INSTANCE>/databases/posts_db
     ```
   - Inject an OAuth2 Bearer token (obtained via `google/auth` Service Account credentials) into the HTTP headers of `actualizarPostEnSpanner()` inside `orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php`:
     ```php
     'headers' => [
         'Authorization' => 'Bearer ' . $accessToken,
         'Content-Type'  => 'application/json',
     ]
     ```

---

### 2.2. Artificial Intelligence Extraction (Gemini API vs. Dynamic Random Mock)

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
> In local development, `GEMINI_API_KEY` is intentionally left blank (`GEMINI_API_KEY=""`). Consequently, `ProcessPostAiBriefHandler` falls back to a **dynamic random mock** (`array_rand`) that selects from 5 static objectives and 5 static target audiences.

#### Current Simulated Behavior
- **Location:** `orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php` (`invocarGeminiCategory1Privacy()` and `generarExtraccionMock()`).
- **Mock Data Structures:**
  - **5 Simulated Objectives:**
    1. `"Aumentar el brand awareness en un 20%"`
    2. `"Generar leads cualificados B2B"`
    3. `"Impulsar ventas del e-commerce en Q3"`
    4. `"Lanzamiento de nuevo producto"`
    5. `"Retención y fidelización de usuarios activos"`
  - **5 Simulated Target Audiences:**
    1. `"Jóvenes 18-24 años segmento gamer"`
    2. `"Directores de tecnología (CTOs) en LATAM"`
    3. `"Público general urbano 25-45 años"`
    4. `"Pequeñas y medianas empresas (PyMEs)"`
    5. `"Compradores compulsivos de tecnología"`
- When `GEMINI_API_KEY` is empty (or if the external HTTP call fails), the handler returns `"Objetivo: <random_objective> | Audiencia: <random_audience>"`.

#### Actionable Remediation for Production
1. **Inject a Active Gemini API Key:**
   Obtain a production API key from Google AI Studio or Vertex AI and set it in `orchestrator/.env` (or your secret manager):
   ```dotenv
   GEMINI_API_KEY="AIzaSyYourProductionGeminiApiKeyHere"
   GEMINI_ENDPOINT="https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent"
   ```
2. **Disable Silent Fallback to Mock in Production:**
   In `orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php`, replace the fallback call to `$this->generarExtraccionMock()` inside the `if ($this->geminiApiKey === '')` and `catch (\Throwable $e)` blocks with an explicit domain exception so Symfony Messenger can trigger its retry policy and mark unrecoverable jobs as `'ERROR'` instead of storing simulated data.
3. **Verify Category 1 Privacy Headers:**
   Keep the already-implemented governance headers (`X-Goog-Vertex-Data-Governance: category-1-no-train`, `X-Goog-Privacy-Disable-Training: true`, `X-Goog-Privacy-Zero-Retention: true`) active to ensure enterprise data compliance.

---

### 2.3. File Attachment Storage (Base64 Data URL vs. Google Drive API)

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
> File attachments uploaded in the Angular form are currently encoded as **Base64 Data URLs** (`data:<mime>;base64,...`) in the browser and persisted directly in the `Adjunto STRING(MAX)` column of Cloud Spanner rather than being uploaded to an external Google Drive folder.

#### Current Simulated Behavior
- **Frontend (`frontend/src/app/components/post-form/post-form.component.ts`):** Uses `FileReader.readAsDataURL(file)` to convert the selected file into a Base64 string and sends it in the `adjunto` field of `POST /posts`.
- **Database (`Posts.Adjunto`):** Stores the raw Base64 payload (up to `STRING(MAX)`, which in Cloud Spanner is limited to 2,621,440 characters / ~2.5 MB).

#### Actionable Remediation for Production
1. **Enable Google Drive API v3 (or Google Cloud Storage):**
   Configure a GCP Service Account with Drive API scopes (`https://www.googleapis.com/auth/drive.file`) and share the target corporate Drive folder with the service account email.
2. **Delegate Binary Upload in the Symfony Worker:**
   In `orchestrator/src/MessageHandler/ProcessPostAiBriefHandler.php`:
   - Read the Base64 string from the pending Post's `adjunto` field.
   - Decode the binary payload and upload it via `POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart`.
   - Replace the raw Base64 string in Cloud Spanner with the returned `webViewLink` (`https://drive.google.com/file/d/<FILE_ID>/view`) during the `UPDATE Posts` transaction:
     ```sql
     UPDATE Posts
     SET Adjunto = @driveUrl,
         DatosExtraidosIA = @datosExtraidosIA,
         EstadoProcesamiento = 'COMPLETADO'
     WHERE Id = @postId
     ```

---

### 2.4. Environment Secrets, CORS Policy & Frontend Endpoint Configuration

> **⚠️ PRODUCTION ALERT: MOCKED DATA & PENDING INTEGRATIONS**
> Development defaults such as static secrets, wildcard CORS headers, and `localhost` URLs must be hardened prior to production release.

#### Actionable Remediation Checklist
1. **Symfony Application Secret (`orchestrator/.env`):**
   - **Current:** `APP_SECRET=c9f8d7e6a5b4c3d2e1f0a9b8c7d6e5f4` (static development string) and `APP_ENV=dev`.
   - **Remediation:** Set `APP_ENV=prod`, `APP_DEBUG=0`, and inject a cryptographically random 32-byte hex secret via environment variables or Kubernetes Secrets.
2. **Golang CORS Middleware (`backend/internal/handler/post_handler.go`):**
   - **Current:** `w.Header().Set("Access-Control-Allow-Origin", "*")`.
   - **Remediation:** Read an `ALLOWED_ORIGIN` environment variable (e.g., `https://campaigns.tcit.cl`) and restrict `Access-Control-Allow-Origin` to trusted frontend origins.
3. **Angular API Base URL (`frontend/src/app/services/post.service.ts`):**
   - **Current:** Hardcoded `private readonly apiUrl = 'http://localhost:8080/posts';`.
   - **Remediation:** Extract `apiUrl` into `src/environments/environment.ts` and `src/environments/environment.prod.ts` so production builds point to the production API Gateway over HTTPS.

