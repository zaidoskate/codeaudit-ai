# CodeAudit AI — API Contract

**Versión:** 1.0 · **Base URL (prod):** `https://api.codeaudit.dev/v1` · **Formato:** JSON (`application/json; charset=utf-8`) · **Fechas:** ISO-8601 UTC

---

## 1. Convenciones generales

- Todo el API vive bajo el prefijo `/v1`. Un cambio incompatible (breaking change) sube la versión a `/v2`; nunca se rompe un contrato dentro de la misma versión.
- IDs de recursos son UUID v4, expuestos como string.
- Toda respuesta exitosa usa el envelope `{ "data": ... }`; toda respuesta de error usa el envelope de la sección 3.
- Paginación (en endpoints de listado): query params `?page=1&page_size=20` (máx. `page_size=100`), respuesta incluye `meta: { page, page_size, total_items, total_pages }`.
- El backend es *stateless* entre requests — el estado de una auditoría vive únicamente en PostgreSQL (Neon), nunca en memoria del proceso, para que múltiples instancias (o un reinicio por cold start de Render) no pierdan trabajo en curso.

## 2. Autenticación y Headers comunes

Existen **dos esquemas de autenticación** distintos, para dos consumidores distintos del API:

| Esquema | Quién lo usa | Cómo se envía | Vigencia |
|---|---|---|---|
| **Bearer Token (API Token)** | CLI | Header `Authorization: Bearer cak_live_<token>` | No expira hasta que se revoca manualmente desde el dashboard |
| **Sesión (Cookie JWT)** | Dashboard Web (Next.js) | Cookie `HttpOnly`, `Secure`, `SameSite=Lax`, emitida tras login con GitHub OAuth | Expira a las 7 días, renovable |

### Headers requeridos en toda request

| Header | Obligatorio | Descripción |
|---|---|---|
| `Authorization` | Sí (excepto `/health` y `/auth/*`) | `Bearer <token>` o cookie de sesión según el consumidor |
| `Content-Type` | Sí en requests con body | `application/json` |
| `X-Request-Id` | No (recomendado) | UUID generado por el cliente para trazabilidad end-to-end en logs |
| `X-CLI-Version` | Sí (solo CLI) | Versión semántica del binario, para detectar clientes desactualizados |

### Headers presentes en toda response

| Header | Descripción |
|---|---|
| `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` | Cuota del **token del usuario** contra el propio backend (no confundir con la cuota de Groq/Gemini) |
| `X-Request-Id` | Eco del enviado por el cliente, o uno generado por el servidor si no vino |

## 3. Modelo estándar de errores

Toda respuesta de error (4xx/5xx) sigue esta forma exacta:

```
{
  "error": {
    "code": "STRING_CODE_ESTABLE",
    "message": "Descripción legible para humanos",
    "details": { "campo_opcional": "contexto adicional" },
    "request_id": "uuid-eco-del-request"
  }
}
```

| HTTP Status | `code` | Cuándo ocurre |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Body mal formado, campo faltante, snippet excede el tamaño máximo |
| 401 | `UNAUTHENTICATED` | Token ausente, inválido o revocado |
| 403 | `FORBIDDEN` | Token válido pero sin permiso sobre el recurso (ej. auditoría de otro usuario) |
| 404 | `NOT_FOUND` | El recurso solicitado no existe |
| 409 | `CONFLICT` | Ej. intentar reintentar una auditoría que ya está `processing` |
| 422 | `UNPROCESSABLE_LANGUAGE` | El lenguaje del snippet no está soportado por el motor de análisis (ver SRS FR-AUD-03) |
| 429 | `RATE_LIMITED` | El usuario superó su propia cuota interna del backend (independiente de la cuota de Groq/Gemini) |
| 502 | `LLM_PROVIDER_ERROR` | Groq y Gemini fallaron ambos en el mismo intento |
| 503 | `SERVICE_WAKING_UP` | Reservado para el estado de cold start (ver nota abajo) |
| 500 | `INTERNAL_ERROR` | Error no controlado |

> **Nota sobre cold starts (Render):** el propio proceso HTTP no puede responder mientras está dormido — esto lo gestiona el CLI, no el servidor (ver sección 5). El código `SERVICE_WAKING_UP` está reservado para cuando en el futuro se añada un *health check proxy* ligero.

---

## 4. Endpoints

### 4.1 `POST /v1/auth/tokens`

Genera un nuevo token de API para el CLI. Requiere sesión web activa (cookie), no un Bearer token — es la única forma de "arrancar la cadena de confianza".

- **Auth:** Cookie de sesión (dashboard)
- **Request body:**
  ```
  { "name": "laptop-personal" }
  ```
- **Response `201 Created`:**
  ```
  {
    "data": {
      "id": "uuid",
      "token": "cak_live_xxxxxxxxxxxxxxxx",
      "name": "laptop-personal",
      "created_at": "2026-07-01T10:00:00Z"
    }
  }
  ```
  El campo `token` **solo se muestra una vez**, en esta respuesta. El backend solo persiste su hash (ver DB_model.md).
- **Errores:** `401 UNAUTHENTICATED` (sesión expirada), `400 VALIDATION_ERROR` (nombre vacío o duplicado)
- **Lógica interna:**
  1. Valida sesión y extrae `user_id`.
  2. Genera token aleatorio criptográficamente seguro (32 bytes).
  3. Calcula y guarda `sha256(token)` en la tabla `api_tokens` — nunca el token en texto plano.
  4. Devuelve el token en claro solo en esta respuesta.

### 4.2 `GET /v1/auth/tokens`

Lista los tokens del usuario autenticado (sin exponer el valor del token, solo metadata).

- **Auth:** Cookie de sesión
- **Response `200 OK`:** `{ "data": [ { "id", "name", "created_at", "last_used_at" } ] }`

### 4.3 `DELETE /v1/auth/tokens/{token_id}`

Revoca un token (ej. el usuario perdió su laptop).

- **Auth:** Cookie de sesión
- **Response:** `204 No Content`
- **Errores:** `404 NOT_FOUND`, `403 FORBIDDEN` (el token pertenece a otro usuario)

### 4.4 `GET /v1/me`

- **Auth:** Cookie de sesión o Bearer token
- **Response `200 OK`:** `{ "data": { "id", "github_username", "email", "created_at" } }`

### 4.5 `POST /v1/audits` — Crear una auditoría

El endpoint central del sistema.

- **Auth:** Bearer token (CLI)
- **Request body:**
  ```
  {
    "source": "cli",
    "language": "go",
    "context": {
      "repo": "zaid/mi-proyecto",
      "branch": "feature/refactor-x",
      "commit_sha": "abc123"
    },
    "diff": "-- unified diff o contenido del snippet --"
  }
  ```
  - `diff`: tamaño máximo 200 KB (ver NFR-PERF-02 en SRS). Excederlo devuelve `400 VALIDATION_ERROR`.
  - `language`: enum controlado (`go`, `javascript`, `typescript` en v1 — ver alcance en Architecture.md).
- **Response `202 Accepted`:**
  ```
  {
    "data": {
      "id": "uuid",
      "status": "pending",
      "created_at": "2026-07-01T10:00:00Z"
    }
  }
  ```
- **Errores:** `400 VALIDATION_ERROR`, `422 UNPROCESSABLE_LANGUAGE`, `429 RATE_LIMITED`
- **Lógica interna:**
  1. Autentica el token y resuelve `user_id`.
  2. Valida tamaño y lenguaje del payload.
  3. Ejecuta el **motor de análisis estático de forma síncrona** (es determinista y rápido — no bloquea significativamente el request).
  4. Inserta el registro en `audits` con `status = 'pending'` y los hallazgos estáticos en `findings` (`source = 'static'`).
  5. Encola un job (tabla `job_queue` o Upstash Redis, ver Architecture.md) con el `audit_id` y los fragmentos de código relevantes para el LLM.
  6. Responde `202` de inmediato — el trabajo del LLM ocurre en segundo plano vía el worker asíncrono.

### 4.6 `GET /v1/audits/{id}`

Consulta el estado/resultado de una auditoría. Es el endpoint que el CLI hace *polling* corto sobre él.

- **Auth:** Bearer token o cookie de sesión (dueño del recurso)
- **Response `200 OK` (en progreso):**
  ```
  { "data": { "id", "status": "processing", "created_at" } }
  ```
- **Response `200 OK` (completada):**
  ```
  {
    "data": {
      "id", "status": "completed", "created_at", "completed_at",
      "llm_provider_used": "groq",
      "findings": [
        {
          "id", "source": "static" | "llm",
          "severity": "blocker" | "critical" | "major" | "minor" | "info",
          "category": "high-cyclomatic-complexity" | "...",
          "line_start", "line_end",
          "explanation": "...",
          "suggested_refactor": "..."
        }
      ]
    }
  }
  ```
- **Errores:** `404 NOT_FOUND`, `403 FORBIDDEN`
- **Lógica interna:** lectura simple de `audits` + `findings` por `audit_id`, sin lógica de negocio adicional — toda la complejidad ya ocurrió en el worker.

### 4.7 `GET /v1/audits` — Historial (para el dashboard)

- **Auth:** Cookie de sesión
- **Query params:** `?page=1&page_size=20&status=completed&repo=zaid/mi-proyecto`
- **Response `200 OK`:** lista paginada de auditorías (sin el detalle completo de `findings`, solo un resumen: conteo por severidad).

### 4.8 `DELETE /v1/audits/{id}`

Purga una auditoría (cumple la estrategia de retención de datos de la sección 4.4 del Architecture.md, dado el límite de almacenamiento de Neon).

- **Auth:** Cookie de sesión (dueño del recurso)
- **Response:** `204 No Content`

### 4.9 `GET /v1/health`

- **Auth:** ninguna
- **Response `200 OK`:** `{ "status": "ok", "db": "ok", "version": "1.0.0" }`
- **Propósito:** monitoreo externo (uptime checks) y diagnóstico rápido — **no** debe usarse para mantener el servicio despierto artificialmente (ver Architecture.md, riesgo de cold start).

---

## 5. Plan del CLI

### 5.1 Comandos

| Comando | Propósito |
|---|---|
| `codeaudit login` | Abre el navegador hacia el dashboard para autenticar con GitHub y pegar el token generado (flujo manual tipo `gh auth login`, sin necesidad de un servidor local de callback) |
| `codeaudit review [--path .] [--diff-only]` | Ejecuta una auditoría sobre el diff actual (contra `main`) o sobre archivos locales |
| `codeaudit status <audit_id>` | Consulta el resultado de una auditoría específica |
| `codeaudit config` | Muestra/edita configuración local (endpoint del API, token activo) |

### 5.2 Configuración local

Archivo `~/.codeaudit/config.yaml` con permisos `0600` (solo lectura/escritura del dueño — ver tabla de seguridad en Architecture.md):

```
api_url: https://api.codeaudit.dev/v1
token: cak_live_xxxxxxxxxxxxxxxx
```

### 5.3 Flujo de `codeaudit review`

1. Detecta el diff local (vía `git diff` contra la rama base) o lee el archivo indicado.
2. Valida tamaño localmente antes de enviar (evita un roundtrip innecesario si ya se sabe que excede el límite).
3. `POST /v1/audits` → recibe `job_id`.
4. Muestra un spinner con mensaje explícito: *"Analizando código..."* y, si detecta latencia anómala (>10s en el primer request), cambia el mensaje a *"El servicio estaba inactivo, despertando..."* — así se convierte el cold start en información útil, no en silencio.
5. Hace polling a `GET /v1/audits/{id}` cada 2 segundos, con backoff exponencial hasta un máximo de 60 segundos de espera total.
6. Imprime el reporte en terminal, coloreado por severidad, con un link directo al dashboard web para ver el detalle completo.

## 6. Plan del Backend (organización conceptual, sin código)

División de responsabilidades en el binario Go (que corre como un único proceso en Render, incluyendo el worker):

- **Capa de transporte (`api`):** define rutas, decodifica/valida requests, traduce respuestas al envelope estándar. No contiene lógica de negocio.
- **Capa de dominio (`audit`, `staticanalysis`, `llm`):** contiene las reglas de negocio — qué es un hallazgo, cómo se calcula severidad, cómo se arma el prompt. No sabe nada de HTTP.
- **Capa de persistencia (`store`):** único punto de acceso a PostgreSQL. Expone operaciones de negocio (`CreateAudit`, `MarkCompleted`), nunca SQL crudo fuera de esta capa.
- **Capa de integración (`providers`):** clientes de Groq y Gemini, con la lógica de failover encapsulada aquí — el resto del sistema solo conoce una interfaz `LLMProvider`, no sabe cuál de los dos respondió.
- **Worker (`worker`):** goroutine de larga duración que consume la cola y orquesta `staticanalysis` (ya ejecutado) + `llm` + `store`.

Esta separación es la que permite, por ejemplo, cambiar Groq por otro proveedor sin tocar el motor de análisis estático, o migrar de Render a otro host sin tocar una sola línea de lógica de negocio — la superficie de cambio queda contenida en `providers` e `infra`, respectivamente.
