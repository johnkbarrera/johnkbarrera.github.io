# Prompt: recrear `layout_example` (FastAPI) como proyecto Spring Boot desde cero

Copia todo el contenido de este archivo y pégalo como prompt inicial a la IA que generará el proyecto.

---

## Contexto

Tengo un microservicio backend escrito en **Python 3.12 + FastAPI** (proyecto `layout_example` dentro de una plataforma multi-proyecto). Quiero que generes un proyecto **Spring Boot equivalente, completo y funcional desde cero**, replicando toda la funcionalidad descrita abajo. No es necesario mantener compatibilidad binaria con Python — solo paridad funcional de endpoints, contratos JSON, reglas de negocio y modelo de datos.

## Stack requerido

- Java 21
- Spring Boot 3.3.x, Maven
- Spring Web (MVC, servlet, no reactivo)
- Spring Data JPA + PostgreSQL (driver `org.postgresql`)
- Spring Data MongoDB
- Spring for GraphQL
- `io.jsonwebtoken:jjwt` (JWT)
- AWS SDK v2 (`software.amazon.awssdk:s3`) con `S3Presigner` para storage compatible con S3/Cloudflare R2 (usar `endpointOverride`)
- Lombok
- Bean Validation (`jakarta.validation`)
- Sin Spring Security completo: basta un `OncePerRequestFilter` ligero que valide el JWT Bearer y exponga `userId`/`email` en el request (igual de simple que el original).

## Proyecto / artefacto Maven

- `groupId`: `com.illarek`
- `artifactId` / nombre del proyecto: `platform-api`
- paquete raíz: `com.illarek.platformapi`

## Arquitectura / paquetes

Refleja la separación por capas del original (`domain` / `infra` / `api`):

```
com.illarek.platformapi
├── api/            # controllers REST, DTOs (records), GraphQL resolvers
├── domain/         # servicios de dominio, modelos, excepciones, interfaces de repos
├── infra/
│   ├── config/     # configuración (settings, beans S3, JWT, etc.)
│   ├── persistence/postgres/   # entidades JPA + repos
│   ├── persistence/mongo/      # documentos Mongo + repos
│   └── clients/     # GoogleOAuthClient, S3 client
```

---

## 1. Configuración (`application.yml` + variables de entorno)

Variables de entorno requeridas (mapear 1:1, sin lógica extra):

```
APP_NAME, APP_ENV, DEBUG, LOG_LEVEL

POSTGRES_HOST, POSTGRES_PORT, POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
MONGO_URI, MONGO_DATABASE
REDIS_URL

OBJECT_STORAGE_PROVIDER=r2
OBJECT_STORAGE_BUCKET, OBJECT_STORAGE_ENDPOINT
OBJECT_STORAGE_ACCESS_KEY, OBJECT_STORAGE_SECRET_KEY, OBJECT_STORAGE_REGION

SECRET_KEY                 # firma JWT, HS256
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=120
REFRESH_TOKEN_EXPIRE_DAYS=30

GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, GOOGLE_REDIRECT_URI
```

Genera también un `.env.example` y un `application.yml` que lea estas variables con `${VAR}`.

---

## 2. Módulo Auth

### Modelos de datos (MongoDB)

**Colección `mobile_users`**
```
_id: ObjectId
email: string (único)
password: string | null      # hash PBKDF2, ver abajo
auth_provider: "local" | "google"
google_id: string | null
```

**Colección `mobile_refresh_tokens`**
```
_id: ObjectId
token: string (único, índice)
user_id: string
email: string
expires_at: datetime
device_id: string | null
created_at: datetime
```

### Hash de password (PBKDF2)

Implementar `PasswordHasher` con:
- Algoritmo `PBKDF2WithHmacSHA256`
- 600 000 iteraciones
- salt aleatorio de 16 bytes
- formato de almacenamiento: `"{iteraciones}:{saltHex}:{hashHex}"`
- `verify(password, stored)`: parsear el formato, recalcular y comparar en tiempo constante (`MessageDigest.isEqual` o equivalente)

### JWT

- Algoritmo HS256, secreto = `SECRET_KEY`
- Claims: `sub` (user id), `email`, `exp`
- Expiración access token = `ACCESS_TOKEN_EXPIRE_MINUTES`
- Refresh token = string aleatorio URL-safe de 32 bytes (`SecureRandom` + base64url), expiración = `REFRESH_TOKEN_EXPIRE_DAYS` días desde ahora

### Endpoints (prefijo `/auth`)

| Método | Path | Body | Respuesta éxito | Errores |
|---|---|---|---|---|
| POST | `/auth/register` | `{email, password}` | `200 {user_id}` | `400 {detail:"User already exists"}` si el email ya existe |
| POST | `/auth/login` | `{email, password, device_id?}` | `200 TokenResponse` | `401 {detail:"Invalid credentials"}` |
| POST | `/auth/google` | `{token, device_id?}` | `200 TokenResponse` | `401 {detail:"Invalid Google token"}` |
| POST | `/auth/refresh-token` | `{refresh_token, device_id?}` | `200 TokenResponse` (nuevo par) | `401 {detail:"Invalid or expired refresh token"}` |
| POST | `/auth/logout` | `{refresh_token}` | `204` (sin body) | — (idempotente, no falla si el token no existe) |
| GET | `/auth/me` | — (header `Authorization: Bearer <jwt>`) | `200 {"user": {"user_id":..., "email":...}}` | `401 {detail:"Invalid token"}` |

`TokenResponse`:
```json
{ "access_token": "...", "refresh_token": "...", "token_type": "bearer" }
```

### Reglas de negocio

- **Register**: si el email ya existe → error. Si no, crear documento con `password` hasheado y `auth_provider="local"`.
- **Login**: buscar por email; si no existe o no tiene `password` → credenciales inválidas. Verificar password con PBKDF2. Devolver `(user_id=_id, email)`.
- **Google login**:
  1. Llamar `GET https://oauth2.googleapis.com/tokeninfo?id_token=<token>`.
  2. Si status != 200 → `InvalidGoogleTokenError` → 401.
  3. De la respuesta JSON tomar `email` y `sub` (google_id). **El `id_token` ya contiene el email; no se recibe por separado.**
  4. Si no existe usuario con ese email, crearlo con `auth_provider="google"`, `google_id=sub`, sin `password`.
  5. Devolver `(user_id=google_id /* sub */, email)` — nota: para usuarios Google, el `sub` del JWT propio es el `google_id`, no el `_id` de Mongo.
- **Refresh token rotation**:
  1. Buscar el refresh token en Mongo. Si no existe → 401.
  2. Si `expires_at` < ahora → borrarlo y devolver 401.
  3. Si el documento tiene `device_id` guardado y no coincide con el `device_id` recibido → 401 (binding por dispositivo).
  4. Borrar el token viejo, generar un nuevo par `(refresh_token, expires_at)` y guardarlo con el mismo `user_id`, `email`, `device_id`.
  5. Devolver nuevo `access_token` (JWT) + nuevo `refresh_token`.
- **Logout**: borrar el documento por `refresh_token`. No falla si no existe (idempotente). El `access_token` JWT existente sigue siendo válido hasta su expiración natural (no hay blacklist).
- **/me**: validar el JWT Bearer con `SECRET_KEY`/`ALGORITHM`; cualquier error de validación → `401 {detail:"Invalid token"}`. Devolver `sub` como `user_id` y `email` del payload.

---

## 3. Módulo Geo Events

### Tabla Postgres `geo_events`

```sql
CREATE TABLE geo_events (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID NULL,
    latitude DOUBLE PRECISION NOT NULL,
    longitude DOUBLE PRECISION NOT NULL,
    altitude DOUBLE PRECISION NULL,
    accuracy DOUBLE PRECISION NULL,
    speed DOUBLE PRECISION NULL,
    heading DOUBLE PRECISION NULL,
    event_type TEXT NOT NULL DEFAULT 'gps_ping',
    device_id TEXT NULL,
    platform TEXT NULL,
    app_version TEXT NULL,
    device_model TEXT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### DTO `GeoEventRequest` (entrada, para POST y para GraphQL input)

```json
{
  "user_id": "uuid | null",
  "latitude": 0.0,
  "longitude": 0.0,
  "altitude": 0.0,
  "accuracy": 0.0,
  "speed": 0.0,
  "heading": 0.0,
  "event_type": "gps_ping",
  "device_id": "string | null",
  "platform": "string | null",
  "app_version": "string | null",
  "device_model": "string | null",
  "recorded_at": "ISO-8601 datetime | null"
}
```

`GeoEventResponse` = lo mismo + `id` (long) + `created_at`. Si `recorded_at` no se envía, usar `now()` (vía `COALESCE` en SQL o default en Java).

### Endpoints — implementar **dos veces** la misma API, con dos implementaciones de persistencia distintas (como en el original, que tiene una versión SQL crudo y otra JPA/ORM):

- **`/geo-events`** → implementación con `JdbcTemplate` / SQL nativo (`INSERT ... RETURNING *`, `SELECT * WHERE ...`).
- **`/geo-events-orm`** → implementación con Spring Data JPA (`@Entity GeoEventEntity`, `JpaRepository`).

Ambas con el mismo contrato:

| Método | Path | Respuesta | Notas |
|---|---|---|---|
| POST | `/{prefix}/` | `201 GeoEventResponse` | crea el evento |
| GET | `/{prefix}/{id}` | `200 GeoEventResponse` o `404` | |
| GET | `/{prefix}/?user_id=&event_type=&limit=50&offset=0` | `200 GeoEventResponse[]` | filtros opcionales, orden por `recorded_at DESC` |
| DELETE | `/{prefix}/{id}` | `204` o `404` | |

---

## 4. Módulo GraphQL

Usar Spring for GraphQL (schema-first, archivo `.graphqls`). Mismo modelo que `geo_events` (usar la implementación ORM como backing repo). Operaciones:

```graphql
type GeoEvent {
  id: ID!
  userId: ID
  latitude: Float!
  longitude: Float!
  altitude: Float
  accuracy: Float
  speed: Float
  heading: Float
  eventType: String!
  deviceId: String
  platform: String
  appVersion: String
  deviceModel: String
  recordedAt: String!
  createdAt: String!
}

input GeoEventInput {
  userId: ID
  latitude: Float!
  longitude: Float!
  altitude: Float
  accuracy: Float
  speed: Float
  heading: Float
  eventType: String = "gps_ping"
  deviceId: String
  platform: String
  appVersion: String
  deviceModel: String
  recordedAt: String
}

type Query {
  geoEvent(id: ID!): GeoEvent
  geoEvents(userId: ID, eventType: String, limit: Int = 50, offset: Int = 0): [GeoEvent!]!
}

type Mutation {
  createGeoEvent(input: GeoEventInput!): GeoEvent!
  deleteGeoEvent(id: ID!): Boolean!
}
```

Expuesto en `/graphql` (GET y POST).

---

## 5. Módulo Storage (S3 / Cloudflare R2)

### Tabla Postgres `files`

```sql
CREATE TABLE files (
    id BIGSERIAL PRIMARY KEY,
    project_slug TEXT NOT NULL,
    user_id TEXT NOT NULL,
    storage_provider TEXT NOT NULL,
    bucket TEXT NOT NULL,
    object_key TEXT NOT NULL,
    url TEXT NULL,
    file_name TEXT NOT NULL,
    content_type TEXT NULL,
    size_bytes BIGINT NULL,
    is_public BOOLEAN NOT NULL DEFAULT false,
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);
```

Soft delete vía `deleted_at` (todas las queries de lectura filtran `deleted_at IS NULL`).

### Cliente S3

`S3Presigner` configurado con `endpointOverride(OBJECT_STORAGE_ENDPOINT)`, credenciales estáticas (`OBJECT_STORAGE_ACCESS_KEY`/`SECRET_KEY`), región `OBJECT_STORAGE_REGION`. Métodos: `presignPutObject`, `presignGetObject`, `deleteObject`.

### Endpoints (prefijo `/storage`, todos requieren JWT Bearer)

| Método | Path | Body | Respuesta | Notas |
|---|---|---|---|---|
| POST | `/storage/upload-url` | `{file_name, content_type, size_bytes?, is_public?, expires_in=3600}` | `200 {upload_url, object_key, expires_in}` | `object_key = "{project_slug}/{user_id}/{uuid}{ext}"`. `project_slug` = nombre del proyecto (constante de config, ej. `"layout_example"`). |
| POST | `/storage/confirm` | `{object_key, file_name, content_type?, size_bytes?, is_public=false}` | `201 FileResponse` | Validar que `object_key` empiece con `"{project_slug}/{user_id}/"` → si no, `403 {detail:"Invalid object_key for this user"}` |
| GET | `/storage/files?limit=50&offset=0` | — | `200 FileResponse[]` | filtra por `project_slug` + `user_id` del JWT |
| DELETE | `/storage/file/{file_id}` | — | `204` | `404 {detail:"File not found"}` si no existe; `403 {detail:"Access denied"}` si no es del usuario/proyecto. Borra el objeto en S3 y hace soft-delete en DB. |

`FileResponse` = todas las columnas de `files` (sin `deleted_at`).

---

## 6. Manejo de errores

Usar `@RestControllerAdvice` para mapear excepciones de dominio a respuestas `{"detail": "..."}` con los códigos indicados en cada tabla arriba (401/400/403/404). No exponer stack traces.

## 7. Health check

`GET /health` → `200 {"project": "platform-api", "status": "ok"}`, donde el nombre viene de una propiedad de configuración (`spring.application.name` o equivalente), no hardcodeado en el controller.

---

## Entregables esperados

1. Proyecto Maven completo y compilable (`mvn clean verify`).
2. `application.yml` + `.env.example` con todas las variables listadas.
3. Migraciones SQL (Flyway o `schema.sql`) para `geo_events` y `files`.
4. README con instrucciones de arranque local (Postgres, Mongo, Redis vía docker-compose si es posible).
5. Tests unitarios mínimos para: hashing de password, rotación de refresh token, y verificación de Google token (mockeando el cliente HTTP).
