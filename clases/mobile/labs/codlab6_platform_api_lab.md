# Manual: crear `platform-api` desde cero usando `layout_example` como plantilla

Este manual construye, archivo por archivo, el proyecto de referencia `layout_example`
(autenticación local + Google, geo-events con SQL crudo y ORM, GraphQL, storage S3/R2).
Al final tendrás un backend FastAPI completo y funcional que puedes **copiar y renombrar**
para crear tu propio proyecto con tus propias credenciales.

---

## 0. Requisitos previos

- Python ≥ 3.12
- [uv](https://docs.astral.sh/uv/) como gestor de paquetes
- Una instancia de **PostgreSQL** (geo-events, files)
- Una instancia de **MongoDB** (usuarios, refresh tokens) — Atlas free tier sirve
- Una instancia de **Redis** (reservado para uso futuro)
- Un bucket **S3 / Cloudflare R2** (almacenamiento de archivos)
- Credenciales OAuth de **Google** (Client ID / Secret) para "Login con Google"

---

## 1. Estructura final

```
platform-api/
├── pyproject.toml
├── credentials/
│   └── layout_example.env          # tus credenciales (NO se commitea)
└── src/
    └── app/
        ├── __init__.py
        ├── main.py
        ├── core/
        │   ├── __init__.py
        │   └── discovery.py
        └── projects/
            ├── __init__.py
            └── layout_example/
                ├── __init__.py
                ├── api/
                │   ├── __init__.py
                │   ├── router.py
                │   ├── auth.py
                │   ├── deps.py
                │   ├── schemas.py
                │   ├── geo_events.py
                │   ├── geo_events_orm.py
                │   ├── storage.py
                │   └── graphql/
                │       ├── __init__.py
                │       ├── router.py
                │       ├── types.py
                │       └── resolvers/
                │           ├── __init__.py
                │           └── geo_events.py
                ├── domain/
                │   ├── auth_service.py
                │   ├── storage_service.py
                │   ├── exceptions.py
                │   ├── ports.py
                │   ├── security.py
                │   └── models/
                │       ├── __init__.py
                │       ├── user.py
                │       ├── geo_event.py
                │       └── file.py
                └── infra/
                    ├── settings.py
                    ├── token.py
                    ├── db/
                    │   ├── __init__.py
                    │   ├── mongo.py
                    │   ├── postgres.py
                    │   └── redis.py
                    ├── orm/
                    │   ├── __init__.py
                    │   ├── base.py
                    │   ├── geo_event.py
                    │   └── file.py
                    ├── repositories/
                    │   ├── __init__.py
                    │   ├── user_repo.py
                    │   ├── refresh_token_repo.py
                    │   ├── geo_event_repo.py
                    │   ├── geo_event_orm_repo.py
                    │   └── file_orm_repo.py
                    └── clients/
                        ├── __init__.py
                        ├── google.py
                        ├── storage.py
                        └── llm.py
```

Regla de dependencias: `api → domain ← infra`. El dominio **nunca** importa de `infra`;
las implementaciones concretas se inyectan vía `Depends` (en `api/deps.py`).

---

## 2. `pyproject.toml`

```toml
[project]
name = "platform-api"
version = "0.1.0"
description = "Multi-project FastAPI platform"
readme = "README.md"
requires-python = ">=3.12"

dependencies = [
    # FastAPI
    "fastapi>=0.116.1",
    "uvicorn[standard]>=0.35.0",
    # Configuration
    "pydantic>=2.11.9",
    "pydantic-settings>=2.11.0",
    "email-validator>=2.2.0",
    # HTTP Client
    "httpx>=0.28.1",
    # PostgreSQL
    "sqlalchemy>=2.0.43",
    "asyncpg>=0.30.0",
    "alembic>=1.16.5",
    "greenlet>=3.0.0",
    # MongoDB
    "pymongo>=4.15.0",
    # Redis
    "redis[hiredis]>=6.4.0",
    # OAuth 2.1 / OIDC
    "authlib>=1.6.4",
    "pyjwt[crypto]>=2.10.1",
    # File Uploads
    "python-multipart>=0.0.20",
    # Cloudflare R2 (S3 compatible)
    "boto3>=1.40.30",
    # High-performance JSON
    "orjson>=3.11.3",
    "motor>=3.7.1",
    "strawberry-graphql[fastapi]>=0.316.0",
    "aioboto3>=15.5.0",
]

[dependency-groups]
dev = [
    "pytest>=8.4.2",
    "pytest-asyncio>=1.2.0",
    "pytest-cov>=7.0.0",
    "ruff>=0.13.2",
    "mypy>=1.18.2",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.mypy]
python_version = "3.12"
strict = true
ignore_missing_imports = true

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.uv]
package = true
```

Instala dependencias:

```bash
uv sync
```

---

## 3. Núcleo de la plataforma (`src/app/`)

### 3.1 `src/app/__init__.py`, `src/app/core/__init__.py`, `src/app/projects/__init__.py`, `src/app/projects/layout_example/__init__.py`

Todos vacíos (marcan los paquetes Python).

### 3.2 `src/app/core/discovery.py`

Escanea `app/projects/*`, importa `api.router` de cada uno y lo monta con prefijo `/<nombre_carpeta>`.
**Esto es lo que permite que copiar una carpeta = nuevo proyecto montado, sin tocar `main.py`.**

```python
import importlib
import pkgutil
from typing import Any

import app.projects as projects
from fastapi import FastAPI

registry: list[dict[str, Any]] = []


def _import_project_module(project_name: str):
    module_path = f"app.projects.{project_name}.api.router"
    try:
        return importlib.import_module(module_path)
    except ModuleNotFoundError:
        return None


def _collect_routes(router) -> list[str]:
    paths = []
    for route in getattr(router, "routes", []):
        path = getattr(route, "path", None)
        methods = getattr(route, "methods", None)
        if path and methods:
            for method in sorted(methods):
                paths.append(f"{method} {path}")
    return paths


def load_projects(app: FastAPI):
    registry.clear()

    print("🔍 Scanning projects...")
    print("PATH:", projects.__path__)

    for m in pkgutil.iter_modules(projects.__path__):
        print("FOUND MODULE:", m.name)

        try:
            module = _import_project_module(m.name)
            if module is None:
                print(f"⚠ No router.py in {m.name}")
                continue

            router = getattr(module, "router", None)

            if router:
                prefix = f"/{m.name}"
                app.include_router(router, prefix=prefix)
                registry.append({
                    "name": m.name,
                    "prefix": prefix,
                    "routes": _collect_routes(router),
                })
                print(f"✔ LOADED: {m.name}")
            else:
                print(f"⚠ No `router` object in {m.name}")

        except Exception as e:
            print(f"❌ ERROR {m.name}: {e}")
```

### 3.3 `src/app/main.py`

Crea la app FastAPI, carga los proyectos y expone una landing page simple en `/`.

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

from app.core.discovery import load_projects, registry

app = FastAPI(title="Platform API", version="1.0.0")

print("STARTING APP")
load_projects(app)


@app.get("/", response_class=HTMLResponse, include_in_schema=False)
async def landing():
    cards = ""
    for project in registry:
        route_rows = "".join(
            f'<tr><td class="method {r.split()[0]}">{r.split()[0]}</td>'
            f'<td class="path">{project["prefix"]}{r.split(" ", 1)[1]}</td></tr>'
            for r in project["routes"]
        )
        cards += f"""
        <div class="card">
            <div class="card-header">
                <h2>{project["name"]}</h2>
                <div class="links">
                    <a href="{project["prefix"]}/health">health</a>
                    <a href="/docs">docs</a>
                </div>
            </div>
            <table>{route_rows}</table>
        </div>"""

    return f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Illarek Lab — Platform API</title>
</head>
<body>
  <h1>Illarek Lab — Platform API</h1>
  <p>{len(registry)} project(s), {sum(len(p["routes"]) for p in registry)} endpoint(s)</p>
  <div class="grid">{cards}</div>
</body>
</html>"""
```

> El HTML completo (estilos, topbar, etc.) está en `src/app/main.py` del repo — aquí se
> muestra una versión reducida porque es solo cosmético y no forma parte de la plantilla
> que vas a copiar.

Con esto ya puedes arrancar (aunque sin proyectos todavía):

```bash
PYTHONPATH=src uv run uvicorn app.main:app --reload
```

---

## 4. Proyecto `layout_example` — capa `infra`

### 4.1 `infra/settings.py`

Lee variables de entorno desde `credentials/{nombre_de_la_carpeta}.env` (vía `PROJECT_NAME`,
que se calcula automáticamente del nombre de carpeta — así cada copia del proyecto usa
su propio archivo de credenciales sin tocar código).

```python
from pathlib import Path

from pydantic_settings import BaseSettings, SettingsConfigDict

PROJECT_NAME = Path(__file__).resolve().parents[1].name
BASE_DIR = Path(__file__).resolve().parents[5]  # repo root (…/platform-api)


class Settings(BaseSettings):

    # App
    APP_NAME: str
    APP_ENV: str
    DEBUG: bool = False
    LOG_LEVEL: str = "INFO"

    # PostgreSQL
    POSTGRES_HOST: str
    POSTGRES_PORT: int
    POSTGRES_DB: str
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str

    # Mongo
    MONGO_URI: str
    MONGO_DATABASE: str

    # Redis
    REDIS_URL: str

    # Storage
    OBJECT_STORAGE_PROVIDER: str = "r2"
    OBJECT_STORAGE_BUCKET: str
    OBJECT_STORAGE_ENDPOINT: str
    OBJECT_STORAGE_ACCESS_KEY: str
    OBJECT_STORAGE_SECRET_KEY: str
    OBJECT_STORAGE_REGION: str

    # Auth
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 120
    REFRESH_TOKEN_EXPIRE_DAYS: int = 30

    GOOGLE_CLIENT_ID: str
    GOOGLE_CLIENT_SECRET: str
    GOOGLE_REDIRECT_URI: str

    # AI
    LLM_URL: str

    # Seed
    SEED_ADMIN_EMAIL: str
    SEED_ADMIN_PASSWORD: str
    SEED_ADMIN_DOCUMENT_ID: str

    model_config = SettingsConfigDict(
        env_file=BASE_DIR / "credentials" / f"{PROJECT_NAME}.env",
        extra="ignore",
    )


settings = Settings()
```

### 4.2 `infra/db/mongo.py`

```python
from motor.motor_asyncio import AsyncIOMotorClient

from app.projects.layout_example.infra.settings import settings


class MongoDB:

    def __init__(self) -> None:
        self.client = AsyncIOMotorClient(settings.MONGO_URI)
        self.db = self.client[settings.MONGO_DATABASE]


mongo = MongoDB()
database = mongo.db
```

### 4.3 `infra/db/postgres.py`

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.projects.layout_example.infra.settings import settings

DATABASE_URL = (
    f"postgresql+asyncpg://"
    f"{settings.POSTGRES_USER}:{settings.POSTGRES_PASSWORD}@"
    f"{settings.POSTGRES_HOST}:{settings.POSTGRES_PORT}/{settings.POSTGRES_DB}"
)

engine = create_async_engine(DATABASE_URL, echo=False, connect_args={"statement_cache_size": 0})
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### 4.4 `infra/db/redis.py`

```python
from redis.asyncio import Redis

from app.projects.layout_example.infra.settings import settings

redis_client = Redis.from_url(settings.REDIS_URL, decode_responses=True)
```

### 4.5 `infra/db/__init__.py`, `infra/orm/__init__.py`, `infra/repositories/__init__.py`, `infra/clients/__init__.py`, `domain/models/__init__.py`

Todos vacíos.

---

## 5. Capa `domain`

### 5.1 `domain/models/user.py`

```python
from typing import Literal, Optional

from pydantic import BaseModel, EmailStr


class UserDocument(BaseModel):
    email: EmailStr
    auth_provider: Literal["local", "google"]
    password: Optional[str] = None
    google_id: Optional[str] = None

    model_config = {"extra": "ignore"}
```

### 5.2 `domain/models/geo_event.py`

```python
from datetime import datetime
from typing import Optional
from uuid import UUID

from pydantic import BaseModel


class GeoEvent(BaseModel):
    id: int
    user_id: Optional[UUID] = None
    latitude: float
    longitude: float
    altitude: Optional[float] = None
    accuracy: Optional[float] = None
    speed: Optional[float] = None
    heading: Optional[float] = None
    event_type: str
    device_id: Optional[str] = None
    platform: Optional[str] = None
    app_version: Optional[str] = None
    device_model: Optional[str] = None
    recorded_at: datetime
    created_at: datetime

    model_config = {"from_attributes": True}
```

### 5.3 `domain/models/file.py`

```python
from datetime import datetime
from typing import Optional

from pydantic import BaseModel


class File(BaseModel):
    id: int
    project_slug: str
    user_id: str
    storage_provider: str
    bucket: str
    object_key: str
    url: Optional[str] = None
    file_name: str
    content_type: Optional[str] = None
    size_bytes: Optional[int] = None
    is_public: bool = False
    uploaded_at: datetime
    deleted_at: Optional[datetime] = None

    model_config = {"from_attributes": True}
```

### 5.4 `domain/security.py`

Hash de contraseñas con **PBKDF2-HMAC-SHA256** (600 000 iteraciones) y generación de
valores de refresh token.

```python
import datetime
import hashlib
import hmac
import os
import secrets

_ITERATIONS = 600_000
_HASH = "sha256"
_SALT_BYTES = 16


def hash_password(password: str) -> str:
    salt = os.urandom(_SALT_BYTES)
    key = hashlib.pbkdf2_hmac(_HASH, password.encode(), salt, _ITERATIONS)
    return f"{_ITERATIONS}:{salt.hex()}:{key.hex()}"


def verify_password(password: str, hashed: str) -> bool:
    try:
        iterations_str, salt_hex, key_hex = hashed.split(":")
        salt = bytes.fromhex(salt_hex)
        expected = bytes.fromhex(key_hex)
        actual = hashlib.pbkdf2_hmac(_HASH, password.encode(), salt, int(iterations_str))
        return hmac.compare_digest(actual, expected)
    except Exception:
        return False


def generate_refresh_token_value(expire_days: int) -> tuple[str, datetime.datetime]:
    token = secrets.token_urlsafe(32)
    expires_at = datetime.datetime.now(datetime.UTC) + datetime.timedelta(days=expire_days)
    return token, expires_at
```

### 5.5 `infra/token.py`

JWT (HS256) para el `access_token`.

```python
import datetime

import jwt

from app.projects.layout_example.infra.settings import settings


def create_token(user_id: str, email: str) -> str:
    expire = datetime.datetime.now(datetime.UTC) + datetime.timedelta(
        minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES,
    )
    payload = {"sub": user_id, "email": email, "exp": expire}
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
```

### 5.6 `domain/exceptions.py`

```python
class AuthError(Exception):
    pass


class InvalidCredentialsError(AuthError):
    pass


class UserAlreadyExistsError(AuthError):
    pass


class InvalidGoogleTokenError(AuthError):
    pass


class InvalidRefreshTokenError(AuthError):
    pass


class StorageError(Exception):
    pass


class StorageFileNotFoundError(StorageError):
    pass


class StorageAccessError(StorageError):
    pass
```

### 5.7 `domain/ports.py`

Interfaces (`Protocol`) que el dominio espera — implementadas en `infra/`.

```python
import datetime
from typing import Any, Optional, Protocol

from app.projects.layout_example.domain.models.file import File
from app.projects.layout_example.domain.models.geo_event import GeoEvent


class IRefreshTokenRepository(Protocol):
    async def create(self, token: str, user_id: str, email: str, expires_at: datetime.datetime, device_id: Optional[str]) -> None: ...
    async def find(self, token: str) -> Optional[dict[str, Any]]: ...
    async def delete(self, token: str) -> None: ...


class IUserRepository(Protocol):
    async def find_by_email(self, email: str) -> Optional[dict[str, Any]]: ...
    async def find_by_google_id(self, google_id: str) -> Optional[dict[str, Any]]: ...
    async def create_user(self, user: dict[str, Any]): ...


class IGoogleOAuthClient(Protocol):
    async def verify_id_token(self, id_token: str) -> dict | None: ...


class IGeoEventRepository(Protocol):
    async def create(self, data: dict[str, Any]) -> GeoEvent: ...
    async def find_by_id(self, id: int) -> Optional[GeoEvent]: ...
    async def find_all(self, user_id: Optional[str], event_type: Optional[str], limit: int, offset: int) -> list[GeoEvent]: ...
    async def delete(self, id: int) -> bool: ...


class IFileRepository(Protocol):
    async def create(self, data: dict[str, Any]) -> File: ...
    async def find_by_id(self, id: int) -> Optional[File]: ...
    async def find_by_project_and_user(self, project_slug: str, user_id: str, limit: int, offset: int) -> list[File]: ...
    async def delete(self, id: int) -> bool: ...


class IStorageClient(Protocol):
    async def get_upload_url(self, key: str, content_type: str, expires_in: int) -> str: ...
    async def delete(self, key: str) -> None: ...
    async def get_url(self, key: str, expires_in: int) -> str: ...
```

---

## 6. Capa `infra` — repositorios y ORM

### 6.1 `infra/repositories/user_repo.py` (MongoDB, colección `mobile_users`)

```python
from typing import Any, Optional

from app.projects.layout_example.infra.db.mongo import database


class UserRepository:

    def __init__(self) -> None:
        self._collection = database["mobile_users"]

    async def find_by_email(self, email: str) -> Optional[dict[str, Any]]:
        return await self._collection.find_one({"email": email})

    async def find_by_google_id(self, google_id: str) -> Optional[dict[str, Any]]:
        return await self._collection.find_one({"google_id": google_id})

    async def create_user(self, user: dict[str, Any]):
        return await self._collection.insert_one(user)
```

### 6.2 `infra/repositories/refresh_token_repo.py` (MongoDB, colección `mobile_refresh_tokens`)

```python
import datetime
from typing import Any, Optional

from app.projects.layout_example.infra.db.mongo import database


class RefreshTokenRepository:

    def __init__(self) -> None:
        self._collection = database["mobile_refresh_tokens"]

    async def create(
        self,
        token: str,
        user_id: str,
        email: str,
        expires_at: datetime.datetime,
        device_id: Optional[str],
    ) -> None:
        await self._collection.insert_one({
            "token": token,
            "user_id": user_id,
            "email": email,
            "expires_at": expires_at,
            "device_id": device_id,
            "created_at": datetime.datetime.now(datetime.UTC),
        })

    async def find(self, token: str) -> Optional[dict[str, Any]]:
        return await self._collection.find_one({"token": token})

    async def delete(self, token: str) -> None:
        await self._collection.delete_one({"token": token})
```

### 6.3 `infra/orm/base.py`

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

### 6.4 `infra/orm/geo_event.py`

```python
from datetime import datetime
from typing import Optional
from uuid import UUID

from sqlalchemy import BigInteger, DateTime, Double, String, Text, func
from sqlalchemy.dialects.postgresql import UUID as PG_UUID
from sqlalchemy.orm import Mapped, mapped_column

from app.projects.layout_example.infra.orm.base import Base


class GeoEventORM(Base):
    __tablename__ = "geo_events"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

    user_id: Mapped[Optional[UUID]] = mapped_column(PG_UUID(as_uuid=True), nullable=True)

    latitude: Mapped[float] = mapped_column(Double)
    longitude: Mapped[float] = mapped_column(Double)
    altitude: Mapped[Optional[float]] = mapped_column(Double, nullable=True)

    accuracy: Mapped[Optional[float]] = mapped_column(Double, nullable=True)
    speed: Mapped[Optional[float]] = mapped_column(Double, nullable=True)
    heading: Mapped[Optional[float]] = mapped_column(Double, nullable=True)

    event_type: Mapped[str] = mapped_column(Text, default="gps_ping")

    device_id: Mapped[Optional[str]] = mapped_column(String, nullable=True)
    platform: Mapped[Optional[str]] = mapped_column(String, nullable=True)
    app_version: Mapped[Optional[str]] = mapped_column(String, nullable=True)
    device_model: Mapped[Optional[str]] = mapped_column(String, nullable=True)

    recorded_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

DDL equivalente (créalo manualmente o vía Alembic):

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

### 6.5 `infra/orm/file.py`

```python
from datetime import datetime

from sqlalchemy import BigInteger, Boolean, DateTime, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column

from app.projects.layout_example.infra.orm.base import Base


class FileORM(Base):
    __tablename__ = "files"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

    project_slug: Mapped[str] = mapped_column(Text, nullable=False)
    user_id: Mapped[str] = mapped_column(Text, nullable=False)

    storage_provider: Mapped[str] = mapped_column(Text, nullable=False)
    bucket: Mapped[str] = mapped_column(Text, nullable=False)
    object_key: Mapped[str] = mapped_column(Text, nullable=False)
    url: Mapped[str | None] = mapped_column(Text, nullable=True)

    file_name: Mapped[str] = mapped_column(Text, nullable=False)
    content_type: Mapped[str | None] = mapped_column(String, nullable=True)
    size_bytes: Mapped[int | None] = mapped_column(BigInteger, nullable=True)

    is_public: Mapped[bool] = mapped_column(Boolean, default=False)

    uploaded_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

DDL equivalente:

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

### 6.6 `infra/repositories/geo_event_repo.py` (SQL crudo)

```python
from typing import Optional

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from app.projects.layout_example.domain.models.geo_event import GeoEvent


class GeoEventRepository:

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, data: dict) -> GeoEvent:
        result = await self._session.execute(
            text("""
                INSERT INTO geo_events (
                    user_id, latitude, longitude, altitude, accuracy,
                    speed, heading, event_type, device_id, platform,
                    app_version, device_model, recorded_at
                ) VALUES (
                    :user_id, :latitude, :longitude, :altitude, :accuracy,
                    :speed, :heading, :event_type, :device_id, :platform,
                    :app_version, :device_model, COALESCE(:recorded_at, NOW())
                ) RETURNING *
            """),
            data,
        )
        await self._session.commit()
        return GeoEvent.model_validate(dict(result.mappings().one()))

    async def find_by_id(self, id: int) -> Optional[GeoEvent]:
        result = await self._session.execute(
            text("SELECT * FROM geo_events WHERE id = :id"),
            {"id": id},
        )
        row = result.mappings().one_or_none()
        return GeoEvent.model_validate(dict(row)) if row else None

    async def find_all(
        self,
        user_id: Optional[str] = None,
        event_type: Optional[str] = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[GeoEvent]:
        filters = []
        params: dict = {"limit": limit, "offset": offset}

        if user_id:
            filters.append("user_id = :user_id")
            params["user_id"] = user_id
        if event_type:
            filters.append("event_type = :event_type")
            params["event_type"] = event_type

        where = f"WHERE {' AND '.join(filters)}" if filters else ""

        result = await self._session.execute(
            text(f"SELECT * FROM geo_events {where} ORDER BY recorded_at DESC LIMIT :limit OFFSET :offset"),
            params,
        )
        return [GeoEvent.model_validate(dict(row)) for row in result.mappings()]

    async def delete(self, id: int) -> bool:
        result = await self._session.execute(
            text("DELETE FROM geo_events WHERE id = :id"),
            {"id": id},
        )
        await self._session.commit()
        return result.rowcount > 0
```

### 6.7 `infra/repositories/geo_event_orm_repo.py` (SQLAlchemy ORM)

```python
from typing import Optional

from sqlalchemy import delete, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.projects.layout_example.domain.models.geo_event import GeoEvent
from app.projects.layout_example.infra.orm.geo_event import GeoEventORM


class GeoEventORMRepository:

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, data: dict) -> GeoEvent:
        event = GeoEventORM(**data)
        self._session.add(event)
        await self._session.commit()
        await self._session.refresh(event)
        return GeoEvent.model_validate(event)

    async def find_by_id(self, id: int) -> Optional[GeoEvent]:
        result = await self._session.get(GeoEventORM, id)
        return GeoEvent.model_validate(result) if result else None

    async def find_all(
        self,
        user_id: Optional[str] = None,
        event_type: Optional[str] = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[GeoEvent]:
        stmt = select(GeoEventORM).order_by(GeoEventORM.recorded_at.desc()).limit(limit).offset(offset)

        if user_id:
            stmt = stmt.where(GeoEventORM.user_id == user_id)
        if event_type:
            stmt = stmt.where(GeoEventORM.event_type == event_type)

        result = await self._session.execute(stmt)
        return [GeoEvent.model_validate(row) for row in result.scalars()]

    async def delete(self, id: int) -> bool:
        result = await self._session.execute(
            delete(GeoEventORM).where(GeoEventORM.id == id)
        )
        await self._session.commit()
        return result.rowcount > 0
```

### 6.8 `infra/repositories/file_orm_repo.py`

```python
from datetime import UTC, datetime
from typing import Optional

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from app.projects.layout_example.domain.models.file import File
from app.projects.layout_example.infra.orm.file import FileORM


class FileORMRepository:

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, data: dict) -> File:
        record = FileORM(**data)
        self._session.add(record)
        await self._session.commit()
        await self._session.refresh(record)
        return File.model_validate(record)

    async def find_by_id(self, id: int) -> Optional[File]:
        stmt = select(FileORM).where(FileORM.id == id, FileORM.deleted_at.is_(None))
        result = await self._session.execute(stmt)
        row = result.scalar_one_or_none()
        return File.model_validate(row) if row else None

    async def find_by_project_and_user(
        self,
        project_slug: str,
        user_id: str,
        limit: int = 50,
        offset: int = 0,
    ) -> list[File]:
        stmt = (
            select(FileORM)
            .where(
                FileORM.project_slug == project_slug,
                FileORM.user_id == user_id,
                FileORM.deleted_at.is_(None),
            )
            .order_by(FileORM.uploaded_at.desc())
            .limit(limit)
            .offset(offset)
        )
        result = await self._session.execute(stmt)
        return [File.model_validate(row) for row in result.scalars()]

    async def delete(self, id: int) -> bool:
        result = await self._session.execute(
            update(FileORM)
            .where(FileORM.id == id, FileORM.deleted_at.is_(None))
            .values(deleted_at=datetime.now(UTC))
        )
        await self._session.commit()
        return result.rowcount > 0
```

---

## 7. Capa `infra` — clientes externos

### 7.1 `infra/clients/google.py`

Verifica un Google ID token contra el endpoint público `tokeninfo` (el token ya trae
`email` y `sub` — no se necesita el `client_secret` para verificarlo).

```python
import httpx


class GoogleOAuthClient:
    TOKENINFO_URL = "https://oauth2.googleapis.com/tokeninfo"

    async def verify_id_token(self, id_token: str) -> dict | None:
        async with httpx.AsyncClient() as client:
            response = await client.get(f"{self.TOKENINFO_URL}?id_token={id_token}")
        if response.status_code != 200:
            return None
        return response.json()


google_oauth_client = GoogleOAuthClient()
```

### 7.2 `infra/clients/storage.py` (S3 / Cloudflare R2, vía `aioboto3`)

```python
import aioboto3

from app.projects.layout_example.infra.settings import settings


class StorageClient:

    def __init__(self) -> None:
        self._session = aioboto3.Session()
        self._bucket = settings.OBJECT_STORAGE_BUCKET
        self._client_kwargs = {
            "endpoint_url": settings.OBJECT_STORAGE_ENDPOINT,
            "aws_access_key_id": settings.OBJECT_STORAGE_ACCESS_KEY,
            "aws_secret_access_key": settings.OBJECT_STORAGE_SECRET_KEY,
            "region_name": settings.OBJECT_STORAGE_REGION,
        }

    async def upload(self, key: str, data: bytes, content_type: str = "application/octet-stream") -> str:
        async with self._session.client("s3", **self._client_kwargs) as s3:
            await s3.put_object(Bucket=self._bucket, Key=key, Body=data, ContentType=content_type)
        return key

    async def download(self, key: str) -> bytes:
        async with self._session.client("s3", **self._client_kwargs) as s3:
            response = await s3.get_object(Bucket=self._bucket, Key=key)
            return await response["Body"].read()

    async def delete(self, key: str) -> None:
        async with self._session.client("s3", **self._client_kwargs) as s3:
            await s3.delete_object(Bucket=self._bucket, Key=key)

    async def get_upload_url(self, key: str, content_type: str, expires_in: int = 3600) -> str:
        async with self._session.client("s3", **self._client_kwargs) as s3:
            return await s3.generate_presigned_url(
                "put_object",
                Params={"Bucket": self._bucket, "Key": key, "ContentType": content_type},
                ExpiresIn=expires_in,
            )

    async def get_url(self, key: str, expires_in: int = 3600) -> str:
        async with self._session.client("s3", **self._client_kwargs) as s3:
            return await s3.generate_presigned_url(
                "get_object",
                Params={"Bucket": self._bucket, "Key": key},
                ExpiresIn=expires_in,
            )


storage_client = StorageClient()
```

### 7.3 `infra/clients/llm.py` (opcional — cliente genérico para un LLM propio)

```python
import httpx

from app.projects.layout_example.infra.settings import settings


class LLMClient:

    def __init__(self) -> None:
        self._base_url = settings.LLM_URL

    async def complete(self, prompt: str, model: str = "default", **kwargs) -> str:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self._base_url}/completions",
                json={"model": model, "prompt": prompt, **kwargs},
            )
            response.raise_for_status()
        return response.json()["choices"][0]["text"]

    async def chat(self, messages: list[dict], model: str = "default", **kwargs) -> str:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self._base_url}/chat/completions",
                json={"model": model, "messages": messages, **kwargs},
            )
            response.raise_for_status()
        return response.json()["choices"][0]["message"]["content"]


llm_client = LLMClient()
```

---

## 8. Capa `domain` — servicios

### 8.1 `domain/auth_service.py`

```python
import datetime
from typing import Optional

from app.projects.layout_example.domain.exceptions import (
    InvalidCredentialsError,
    InvalidGoogleTokenError,
    InvalidRefreshTokenError,
    UserAlreadyExistsError,
)
from app.projects.layout_example.domain.ports import IGoogleOAuthClient, IRefreshTokenRepository, IUserRepository
from app.projects.layout_example.domain.security import generate_refresh_token_value, hash_password, verify_password


class AuthService:

    def __init__(
        self,
        user_repo: IUserRepository,
        google_client: IGoogleOAuthClient,
        refresh_token_repo: IRefreshTokenRepository,
        refresh_token_expire_days: int = 30,
    ) -> None:
        self._user_repo = user_repo
        self._google_client = google_client
        self._refresh_token_repo = refresh_token_repo
        self._refresh_token_expire_days = refresh_token_expire_days

    async def login(self, email: str, password: str) -> tuple[str, str]:
        user = await self._user_repo.find_by_email(email)

        if not user or not user.get("password"):
            raise InvalidCredentialsError

        if not verify_password(password, user["password"]):
            raise InvalidCredentialsError

        return str(user["_id"]), email

    async def register(self, email: str, password: str) -> str:
        if await self._user_repo.find_by_email(email):
            raise UserAlreadyExistsError

        result = await self._user_repo.create_user(
            {
                "email": email,
                "password": hash_password(password),
                "auth_provider": "local",
            }
        )
        return str(result.inserted_id)

    async def google_login(self, id_token: str) -> tuple[str, str]:
        data = await self._google_client.verify_id_token(id_token)

        if not data:
            raise InvalidGoogleTokenError

        email = data["email"]
        google_id = data["sub"]

        if not await self._user_repo.find_by_email(email):
            await self._user_repo.create_user(
                {
                    "email": email,
                    "google_id": google_id,
                    "auth_provider": "google",
                }
            )

        return google_id, email

    async def logout(self, refresh_token: str) -> None:
        await self._refresh_token_repo.delete(refresh_token)

    async def create_refresh_token(self, user_id: str, email: str, device_id: Optional[str] = None) -> str:
        token, expires_at = generate_refresh_token_value(self._refresh_token_expire_days)
        await self._refresh_token_repo.create(token, user_id, email, expires_at, device_id)
        return token

    async def rotate_refresh_token(self, old_token: str, device_id: Optional[str] = None) -> tuple[str, str, str]:
        doc = await self._refresh_token_repo.find(old_token)

        if not doc:
            raise InvalidRefreshTokenError

        expires_at: datetime.datetime = doc["expires_at"]
        if expires_at.tzinfo is None:
            expires_at = expires_at.replace(tzinfo=datetime.timezone.utc)

        if expires_at < datetime.datetime.now(datetime.UTC):
            await self._refresh_token_repo.delete(old_token)
            raise InvalidRefreshTokenError

        stored_device_id = doc.get("device_id")
        if stored_device_id and stored_device_id != device_id:
            raise InvalidRefreshTokenError

        await self._refresh_token_repo.delete(old_token)

        new_token, new_expires_at = generate_refresh_token_value(self._refresh_token_expire_days)
        await self._refresh_token_repo.create(new_token, doc["user_id"], doc["email"], new_expires_at, device_id)

        return doc["user_id"], doc["email"], new_token
```

### 8.2 `domain/storage_service.py`

```python
import uuid
from pathlib import Path
from typing import Optional

from app.projects.layout_example.domain.exceptions import StorageAccessError, StorageFileNotFoundError
from app.projects.layout_example.domain.models.file import File
from app.projects.layout_example.domain.ports import IFileRepository, IStorageClient
from app.projects.layout_example.infra.settings import settings


class StorageService:

    def __init__(self, file_repo: IFileRepository, storage_client: IStorageClient) -> None:
        self._repo = file_repo
        self._storage = storage_client

    async def generate_upload_url(
        self,
        project_slug: str,
        user_id: str,
        file_name: str,
        content_type: str,
        expires_in: int = 3600,
    ) -> dict:
        ext = Path(file_name).suffix
        object_key = f"{project_slug}/{user_id}/{uuid.uuid4()}{ext}"
        upload_url = await self._storage.get_upload_url(object_key, content_type, expires_in)
        return {"upload_url": upload_url, "object_key": object_key, "expires_in": expires_in}

    async def confirm_upload(
        self,
        project_slug: str,
        user_id: str,
        object_key: str,
        file_name: str,
        content_type: Optional[str],
        size_bytes: Optional[int],
        is_public: bool,
    ) -> File:
        if not object_key.startswith(f"{project_slug}/{user_id}/"):
            raise StorageAccessError

        return await self._repo.create({
            "project_slug": project_slug,
            "user_id": user_id,
            "storage_provider": settings.OBJECT_STORAGE_PROVIDER,
            "bucket": settings.OBJECT_STORAGE_BUCKET,
            "object_key": object_key,
            "url": None,
            "file_name": file_name,
            "content_type": content_type,
            "size_bytes": size_bytes,
            "is_public": is_public,
        })

    async def list_files(
        self,
        project_slug: str,
        user_id: str,
        limit: int = 50,
        offset: int = 0,
    ) -> list[File]:
        return await self._repo.find_by_project_and_user(project_slug, user_id, limit, offset)

    async def delete_file(self, file_id: int, project_slug: str, user_id: str) -> None:
        file = await self._repo.find_by_id(file_id)

        if not file:
            raise StorageFileNotFoundError

        if str(file.user_id) != user_id or file.project_slug != project_slug:
            raise StorageAccessError

        await self._storage.delete(file.object_key)
        await self._repo.delete(file_id)
```

---

## 9. Capa `api`

### 9.1 `api/schemas.py`

```python
from datetime import datetime
from typing import Optional
from uuid import UUID

from pydantic import BaseModel, EmailStr


class LoginRequest(BaseModel):
    email: EmailStr
    password: str
    device_id: Optional[str] = None


class GoogleLoginRequest(BaseModel):
    token: str
    device_id: Optional[str] = None


class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class RefreshTokenRequest(BaseModel):
    refresh_token: str
    device_id: Optional[str] = None


class LogoutRequest(BaseModel):
    refresh_token: str


class RegisterResponse(BaseModel):
    user_id: str


# ── Storage ──────────────────────────────────────────────────────────────────

class UploadUrlRequest(BaseModel):
    file_name: str
    content_type: str
    size_bytes: Optional[int] = None
    is_public: bool = False
    expires_in: int = 3600


class UploadUrlResponse(BaseModel):
    upload_url: str
    object_key: str
    expires_in: int


class ConfirmUploadRequest(BaseModel):
    object_key: str
    file_name: str
    content_type: Optional[str] = None
    size_bytes: Optional[int] = None
    is_public: bool = False


class FileResponse(BaseModel):
    id: int
    project_slug: str
    user_id: str
    storage_provider: str
    bucket: str
    object_key: str
    url: Optional[str] = None
    file_name: str
    content_type: Optional[str] = None
    size_bytes: Optional[int] = None
    is_public: bool
    uploaded_at: datetime


class GeoEventCreate(BaseModel):
    user_id: Optional[UUID] = None
    latitude: float
    longitude: float
    altitude: Optional[float] = None
    accuracy: Optional[float] = None
    speed: Optional[float] = None
    heading: Optional[float] = None
    event_type: str = "gps_ping"
    device_id: Optional[str] = None
    platform: Optional[str] = None
    app_version: Optional[str] = None
    device_model: Optional[str] = None
    recorded_at: Optional[datetime] = None


class GeoEventResponse(BaseModel):
    id: int
    user_id: Optional[UUID] = None
    latitude: float
    longitude: float
    altitude: Optional[float] = None
    accuracy: Optional[float] = None
    speed: Optional[float] = None
    heading: Optional[float] = None
    event_type: str
    device_id: Optional[str] = None
    platform: Optional[str] = None
    app_version: Optional[str] = None
    device_model: Optional[str] = None
    recorded_at: datetime
    created_at: datetime
```

### 9.2 `api/deps.py`

Wiring de dependencias — instancia los repos/servicios y los inyecta vía `Depends`.

```python
from typing import AsyncGenerator

import jwt
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.projects.layout_example.domain.auth_service import AuthService
from app.projects.layout_example.infra.clients.google import GoogleOAuthClient
from app.projects.layout_example.infra.db.postgres import async_session
from app.projects.layout_example.domain.storage_service import StorageService
from app.projects.layout_example.infra.clients.storage import storage_client
from app.projects.layout_example.infra.repositories.file_orm_repo import FileORMRepository
from app.projects.layout_example.infra.repositories.geo_event_orm_repo import GeoEventORMRepository
from app.projects.layout_example.infra.repositories.geo_event_repo import GeoEventRepository
from app.projects.layout_example.infra.repositories.refresh_token_repo import RefreshTokenRepository
from app.projects.layout_example.infra.repositories.user_repo import UserRepository
from app.projects.layout_example.infra.settings import settings

security = HTTPBearer()

_user_repo = UserRepository()
_google_client = GoogleOAuthClient()
_refresh_token_repo = RefreshTokenRepository()
_auth_service = AuthService(_user_repo, _google_client, _refresh_token_repo, settings.REFRESH_TOKEN_EXPIRE_DAYS)


def get_auth_service() -> AuthService:
    return _auth_service


async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session


def get_geo_event_repo(session: AsyncSession = Depends(get_db_session)) -> GeoEventRepository:
    return GeoEventRepository(session)


def get_geo_event_orm_repo(session: AsyncSession = Depends(get_db_session)) -> GeoEventORMRepository:
    return GeoEventORMRepository(session)


def get_storage_service(session: AsyncSession = Depends(get_db_session)) -> StorageService:
    return StorageService(FileORMRepository(session), storage_client)


def get_current_user(token=Depends(security)):
    try:
        payload = jwt.decode(
            token.credentials,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM],
        )
        return {"user_id": payload["sub"], "email": payload["email"]}
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token") from None
```

### 9.3 `api/auth.py`

```python
from fastapi import APIRouter, Depends, HTTPException

from app.projects.layout_example.api.deps import get_auth_service, get_current_user
from app.projects.layout_example.api.schemas import (
    GoogleLoginRequest,
    LoginRequest,
    LogoutRequest,
    RefreshTokenRequest,
    RegisterResponse,
    TokenResponse,
)
from app.projects.layout_example.domain.auth_service import AuthService
from app.projects.layout_example.domain.exceptions import (
    InvalidCredentialsError,
    InvalidGoogleTokenError,
    InvalidRefreshTokenError,
    UserAlreadyExistsError,
)
from app.projects.layout_example.infra.token import create_token

router = APIRouter(prefix="/auth", tags=["auth"])


@router.post("/login", response_model=TokenResponse)
async def login(payload: LoginRequest, service: AuthService = Depends(get_auth_service)):
    try:
        user_id, email = await service.login(payload.email, payload.password)
        refresh_token = await service.create_refresh_token(user_id, email, payload.device_id)
        return TokenResponse(access_token=create_token(user_id, email), refresh_token=refresh_token)
    except InvalidCredentialsError:
        raise HTTPException(status_code=401, detail="Invalid credentials") from None


@router.post("/register", response_model=RegisterResponse)
async def register(payload: LoginRequest, service: AuthService = Depends(get_auth_service)):
    try:
        user_id = await service.register(payload.email, payload.password)
        return RegisterResponse(user_id=user_id)
    except UserAlreadyExistsError:
        raise HTTPException(status_code=400, detail="User already exists") from None


@router.post("/google", response_model=TokenResponse)
async def google_login(payload: GoogleLoginRequest, service: AuthService = Depends(get_auth_service)):
    try:
        google_id, email = await service.google_login(payload.token)
        refresh_token = await service.create_refresh_token(google_id, email, payload.device_id)
        return TokenResponse(access_token=create_token(google_id, email), refresh_token=refresh_token)
    except InvalidGoogleTokenError:
        raise HTTPException(status_code=401, detail="Invalid Google token") from None


@router.post("/refresh-token", response_model=TokenResponse)
async def refresh_token(payload: RefreshTokenRequest, service: AuthService = Depends(get_auth_service)):
    try:
        user_id, email, new_refresh_token = await service.rotate_refresh_token(payload.refresh_token, payload.device_id)
        return TokenResponse(access_token=create_token(user_id, email), refresh_token=new_refresh_token)
    except InvalidRefreshTokenError:
        raise HTTPException(status_code=401, detail="Invalid or expired refresh token") from None


@router.post("/logout", status_code=204)
async def logout(payload: LogoutRequest, service: AuthService = Depends(get_auth_service)):
    await service.logout(payload.refresh_token)


@router.get("/me")
async def me(user=Depends(get_current_user)):
    return {"user": user}
```

### 9.4 `api/geo_events.py` (SQL crudo)

```python
from typing import Optional

from fastapi import APIRouter, Depends, HTTPException

from app.projects.layout_example.api.deps import get_geo_event_repo
from app.projects.layout_example.api.schemas import GeoEventCreate, GeoEventResponse
from app.projects.layout_example.infra.repositories.geo_event_repo import GeoEventRepository

router = APIRouter(prefix="/geo-events", tags=["geo-events"])


@router.post("/", response_model=GeoEventResponse, status_code=201)
async def create(payload: GeoEventCreate, repo: GeoEventRepository = Depends(get_geo_event_repo)):
    return await repo.create(payload.model_dump())


@router.get("/{id}", response_model=GeoEventResponse)
async def get_by_id(id: int, repo: GeoEventRepository = Depends(get_geo_event_repo)):
    event = await repo.find_by_id(id)
    if not event:
        raise HTTPException(status_code=404, detail="Geo event not found")
    return event


@router.get("/", response_model=list[GeoEventResponse])
async def get_all(
    user_id: Optional[str] = None,
    event_type: Optional[str] = None,
    limit: int = 50,
    offset: int = 0,
    repo: GeoEventRepository = Depends(get_geo_event_repo),
):
    return await repo.find_all(user_id=user_id, event_type=event_type, limit=limit, offset=offset)


@router.delete("/{id}", status_code=204)
async def delete(id: int, repo: GeoEventRepository = Depends(get_geo_event_repo)):
    deleted = await repo.delete(id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Geo event not found")
```

### 9.5 `api/geo_events_orm.py` (idéntico contrato, vía ORM)

```python
from typing import Optional

from fastapi import APIRouter, Depends, HTTPException

from app.projects.layout_example.api.deps import get_geo_event_orm_repo
from app.projects.layout_example.api.schemas import GeoEventCreate, GeoEventResponse
from app.projects.layout_example.infra.repositories.geo_event_orm_repo import GeoEventORMRepository

router = APIRouter(prefix="/geo-events-orm", tags=["geo-events-orm"])


@router.post("/", response_model=GeoEventResponse, status_code=201)
async def create(payload: GeoEventCreate, repo: GeoEventORMRepository = Depends(get_geo_event_orm_repo)):
    return await repo.create(payload.model_dump())


@router.get("/{id}", response_model=GeoEventResponse)
async def get_by_id(id: int, repo: GeoEventORMRepository = Depends(get_geo_event_orm_repo)):
    event = await repo.find_by_id(id)
    if not event:
        raise HTTPException(status_code=404, detail="Geo event not found")
    return event


@router.get("/", response_model=list[GeoEventResponse])
async def get_all(
    user_id: Optional[str] = None,
    event_type: Optional[str] = None,
    limit: int = 50,
    offset: int = 0,
    repo: GeoEventORMRepository = Depends(get_geo_event_orm_repo),
):
    return await repo.find_all(user_id=user_id, event_type=event_type, limit=limit, offset=offset)


@router.delete("/{id}", status_code=204)
async def delete(id: int, repo: GeoEventORMRepository = Depends(get_geo_event_orm_repo)):
    deleted = await repo.delete(id)
    if not deleted:
        raise HTTPException(status_code=404, detail="Geo event not found")
```

### 9.6 `api/storage.py`

```python
from fastapi import APIRouter, Depends, HTTPException

from app.projects.layout_example.api.deps import get_current_user, get_storage_service
from app.projects.layout_example.api.schemas import (
    ConfirmUploadRequest,
    FileResponse,
    UploadUrlRequest,
    UploadUrlResponse,
)
from app.projects.layout_example.domain.exceptions import StorageAccessError, StorageFileNotFoundError
from app.projects.layout_example.domain.storage_service import StorageService
from app.projects.layout_example.infra.settings import PROJECT_NAME

router = APIRouter(prefix="/storage", tags=["storage"])


@router.post("/upload-url", response_model=UploadUrlResponse)
async def get_upload_url(
    payload: UploadUrlRequest,
    current_user=Depends(get_current_user),
    service: StorageService = Depends(get_storage_service),
):
    result = await service.generate_upload_url(
        project_slug=PROJECT_NAME,
        user_id=current_user["user_id"],
        file_name=payload.file_name,
        content_type=payload.content_type,
        expires_in=payload.expires_in,
    )
    return UploadUrlResponse(**result)


@router.post("/confirm", response_model=FileResponse, status_code=201)
async def confirm_upload(
    payload: ConfirmUploadRequest,
    current_user=Depends(get_current_user),
    service: StorageService = Depends(get_storage_service),
):
    try:
        file = await service.confirm_upload(
            project_slug=PROJECT_NAME,
            user_id=current_user["user_id"],
            object_key=payload.object_key,
            file_name=payload.file_name,
            content_type=payload.content_type,
            size_bytes=payload.size_bytes,
            is_public=payload.is_public,
        )
    except StorageAccessError:
        raise HTTPException(status_code=403, detail="Invalid object_key for this user") from None
    return FileResponse.model_validate(file.model_dump())


@router.get("/files", response_model=list[FileResponse])
async def list_files(
    limit: int = 50,
    offset: int = 0,
    current_user=Depends(get_current_user),
    service: StorageService = Depends(get_storage_service),
):
    files = await service.list_files(
        project_slug=PROJECT_NAME,
        user_id=current_user["user_id"],
        limit=limit,
        offset=offset,
    )
    return [FileResponse.model_validate(f.model_dump()) for f in files]


@router.delete("/file/{file_id}", status_code=204)
async def delete_file(
    file_id: int,
    current_user=Depends(get_current_user),
    service: StorageService = Depends(get_storage_service),
):
    try:
        await service.delete_file(
            file_id=file_id,
            project_slug=PROJECT_NAME,
            user_id=current_user["user_id"],
        )
    except StorageFileNotFoundError:
        raise HTTPException(status_code=404, detail="File not found") from None
    except StorageAccessError:
        raise HTTPException(status_code=403, detail="Access denied") from None
```

### 9.7 `api/graphql/types.py`

```python
from datetime import datetime
from typing import Optional
from uuid import UUID

import strawberry


@strawberry.type
class GeoEventType:
    id: int
    user_id: Optional[UUID]
    latitude: float
    longitude: float
    altitude: Optional[float]
    accuracy: Optional[float]
    speed: Optional[float]
    heading: Optional[float]
    event_type: str
    device_id: Optional[str]
    platform: Optional[str]
    app_version: Optional[str]
    device_model: Optional[str]
    recorded_at: datetime
    created_at: datetime


@strawberry.input
class GeoEventInput:
    latitude: float
    longitude: float
    user_id: Optional[UUID] = None
    altitude: Optional[float] = None
    accuracy: Optional[float] = None
    speed: Optional[float] = None
    heading: Optional[float] = None
    event_type: str = "gps_ping"
    device_id: Optional[str] = None
    platform: Optional[str] = None
    app_version: Optional[str] = None
    device_model: Optional[str] = None
    recorded_at: Optional[datetime] = None
```

### 9.8 `api/graphql/resolvers/geo_events.py`

```python
from typing import Optional

import strawberry

from app.projects.layout_example.api.graphql.types import GeoEventInput, GeoEventType
from app.projects.layout_example.infra.repositories.geo_event_repo import GeoEventRepository


def _to_type(event) -> GeoEventType:
    return GeoEventType(**event.model_dump())


@strawberry.type
class GeoEventQuery:

    @strawberry.field
    async def geo_event(self, id: int, info: strawberry.types.Info) -> Optional[GeoEventType]:
        repo: GeoEventRepository = info.context["repo"]
        event = await repo.find_by_id(id)
        return _to_type(event) if event else None

    @strawberry.field
    async def geo_events(
        self,
        info: strawberry.types.Info,
        user_id: Optional[str] = None,
        event_type: Optional[str] = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[GeoEventType]:
        repo: GeoEventRepository = info.context["repo"]
        events = await repo.find_all(user_id=user_id, event_type=event_type, limit=limit, offset=offset)
        return [_to_type(e) for e in events]


@strawberry.type
class GeoEventMutation:

    @strawberry.mutation
    async def create_geo_event(self, input: GeoEventInput, info: strawberry.types.Info) -> GeoEventType:
        repo: GeoEventRepository = info.context["repo"]
        event = await repo.create(strawberry.asdict(input))
        return _to_type(event)

    @strawberry.mutation
    async def delete_geo_event(self, id: int, info: strawberry.types.Info) -> bool:
        repo: GeoEventRepository = info.context["repo"]
        return await repo.delete(id)
```

### 9.9 `api/graphql/router.py`

```python
import strawberry
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from strawberry.fastapi import GraphQLRouter

from app.projects.layout_example.api.deps import get_db_session
from app.projects.layout_example.api.graphql.resolvers.geo_events import GeoEventMutation, GeoEventQuery
from app.projects.layout_example.infra.repositories.geo_event_repo import GeoEventRepository


async def get_context(session: AsyncSession = Depends(get_db_session)):
    return {"repo": GeoEventRepository(session)}


schema = strawberry.Schema(query=GeoEventQuery, mutation=GeoEventMutation)

router = GraphQLRouter(schema, context_getter=get_context)
```

`api/graphql/__init__.py` y `api/graphql/resolvers/__init__.py`: vacíos.

### 9.10 `api/router.py`

Punto de entrada del proyecto — **esto es lo único que `discovery.py` busca**.
El health check usa `PROJECT_NAME` (derivado del nombre de la carpeta), no un string fijo.

```python
from fastapi import APIRouter

from app.projects.layout_example.api.auth import router as auth_router
from app.projects.layout_example.api.geo_events import router as geo_events_router
from app.projects.layout_example.api.geo_events_orm import router as geo_events_orm_router
from app.projects.layout_example.api.graphql.router import router as graphql_router
from app.projects.layout_example.api.storage import router as storage_router
from app.projects.layout_example.infra.settings import PROJECT_NAME

router = APIRouter()


@router.get("/health")
async def health():
    return {"project": PROJECT_NAME, "status": "ok"}


router.include_router(auth_router)
router.include_router(geo_events_router)
router.include_router(geo_events_orm_router)
router.include_router(graphql_router, prefix="/graphql")
router.include_router(storage_router)
```

---

## 10. Credenciales (`credentials/layout_example.env`)

Copia el ejemplo y rellena **tus propios valores** (no commitear este archivo — está en
`.gitignore`):

```bash
cp credentials/layout_example.env.example credentials/layout_example.env
```

```env
# ── App ───────────────────────────────────────────────────────
APP_NAME=Mobile API
APP_ENV=development
DEBUG=true
LOG_LEVEL=INFO

# ── PostgreSQL ────────────────────────────────────────────────
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=mobile_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# ── MongoDB ───────────────────────────────────────────────────
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>.mongodb.net/
MONGO_DATABASE=test

# ── Redis ─────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ── Object Storage (Cloudflare R2 / AWS S3) ──────────────────
OBJECT_STORAGE_BUCKET=mobile
OBJECT_STORAGE_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com
OBJECT_STORAGE_ACCESS_KEY=your_access_key
OBJECT_STORAGE_SECRET_KEY=your_secret_key
OBJECT_STORAGE_REGION=auto

# ── CORS ──────────────────────────────────────────────────────
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173

# ── Authentication (JWT) ─────────────────────────────────────
SECRET_KEY=change_this_to_a_long_random_secret_key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=120
REFRESH_TOKEN_EXPIRE_DAYS=30

# ── Google Login ──────────────────────────────────────────────
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google/callback

# ── Seed ──────────────────────────────────────────────────────
SEED_ADMIN_EMAIL=admin@mobileapi.com
SEED_ADMIN_PASSWORD=Admin1234!
SEED_ADMIN_DOCUMENT_ID=00000000

# ── LLM ───────────────────────────────────────────────────────
LLM_URL=http://127.0.0.1:9000
```

`SECRET_KEY` sugerido: generar uno aleatorio con

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Para `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`: crear credenciales OAuth 2.0 ("Web
application") en [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
— ver `docs/google-auth-testing.md` para el flujo de prueba completo.

---

## 11. Arrancar el proyecto

```bash
PYTHONPATH=src uv run uvicorn app.main:app --reload
```

| URL | Descripción |
|-----|-------------|
| `http://localhost:8000/` | Landing page |
| `http://localhost:8000/docs` | Swagger UI |
| `http://localhost:8000/layout_example/health` | `{"project": "layout_example", "status": "ok"}` |

Importa `docs/mobile-api.postman_collection.json` en Postman para probar todos los
endpoints (auth, geo-events, storage, GraphQL).

---

## 12. Crear tu propio proyecto a partir de la plantilla

1. **Copiar la carpeta**

   ```bash
   cp -r src/app/projects/layout_example src/app/projects/mi_proyecto
   ```

2. **Renombrar los imports**: dentro de `src/app/projects/mi_proyecto/`, reemplaza
   todas las ocurrencias de `app.projects.layout_example` por
   `app.projects.mi_proyecto`:

   ```bash
   find src/app/projects/mi_proyecto -name "*.py" \
     -exec sed -i '' 's/app\.projects\.layout_example/app.projects.mi_proyecto/g' {} +
   ```

   (En Linux, quita el `''` de `sed -i`.)

3. **Crear tu archivo de credenciales** — el nombre se deriva automáticamente del
   nombre de la carpeta (`PROJECT_NAME` en `settings.py`):

   ```bash
   cp credentials/layout_example.env.example credentials/mi_proyecto.env
   ```

   Rellena `credentials/mi_proyecto.env` con **tus propias** credenciales de Postgres,
   MongoDB, Redis, S3/R2 y Google OAuth (no reutilices las del proyecto de ejemplo).

4. **Reiniciar el servidor** — `discovery.py` monta automáticamente el nuevo proyecto
   en `/mi_proyecto/*`, sin tocar `main.py`:

   ```bash
   PYTHONPATH=src uv run uvicorn app.main:app --reload
   ```

   Verifica en `http://localhost:8000/mi_proyecto/health` →
   `{"project": "mi_proyecto", "status": "ok"}`.

5. A partir de aquí, modifica libremente `mi_proyecto`: agrega/quita endpoints,
   modelos, tablas, etc. — está completamente aislado del resto.

---

## 13. Resumen de endpoints

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/{slug}/health` | — | Estado del proyecto |
| POST | `/{slug}/auth/register` | — | Crear usuario local |
| POST | `/{slug}/auth/login` | — | Login con email/password |
| POST | `/{slug}/auth/google` | — | Login con Google ID token |
| POST | `/{slug}/auth/refresh-token` | — | Rota access+refresh token |
| POST | `/{slug}/auth/logout` | — | Invalida el refresh token |
| GET | `/{slug}/auth/me` | Bearer | Usuario actual |
| POST/GET/DELETE | `/{slug}/geo-events/...` | — | CRUD geo-events (SQL crudo) |
| POST/GET/DELETE | `/{slug}/geo-events-orm/...` | — | CRUD geo-events (ORM) |
| GET/POST | `/{slug}/graphql` | — | GraphQL (geo-events) |
| POST | `/{slug}/storage/upload-url` | Bearer | URL prefirmada de subida |
| POST | `/{slug}/storage/confirm` | Bearer | Confirmar archivo subido |
| GET | `/{slug}/storage/files` | Bearer | Listar archivos del usuario |
| DELETE | `/{slug}/storage/file/{id}` | Bearer | Eliminar archivo (soft delete) |
