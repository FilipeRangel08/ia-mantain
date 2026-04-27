# 🌐 AGENTS.md — Camada API (FastAPI)

> **Escopo:** `backend/api/`
> **Responsabilidade:** Expor a lógica do `backend/core/` via HTTP REST, com autenticação Supabase, validação Pydantic v2, RBAC e tratamento de erros padronizado.
> **Stack:** FastAPI + SQLAlchemy 2.0 async + Pydantic v2 + Supabase Auth
> **Última atualização:** 2026-04-27

---

## 🎯 1. PRINCÍPIOS NÃO-NEGOCIÁVEIS

1. **Camada fina** — API só orquestra. Lógica de negócio **sempre** em `core/`.
2. **Async em todo lugar** — `async def` + `AsyncSession` + `asyncpg`. Nunca bloqueante.
3. **Validação dupla** — Pydantic (request) + RBAC (autorização) + RLS Supabase (banco).
4. **Pydantic v2** — `BaseModel` + `model_config` + `Field`. Nunca v1.
5. **Erros padronizados** — formato único em toda API (ver §8).
6. **Sem lógica de IA aqui** — endpoints `/ia/*` apenas delegam para `backend/ia/`.
7. **Supabase Auth como fonte de verdade** — backend **valida** JWT, não emite.
8. **Constantes canônicas** — nunca hardcode rota, role ou código de erro.
9. **OpenAPI completo** — toda rota com `summary`, `description`, `responses`, `tags`.
10. **Testes obrigatórios** — toda rota nova tem teste em `tests/api/`.

---

## 📁 2. ESTRUTURA DE PASTAS

```
backend/api/
├── AGENTS.md                          # este arquivo
├── __init__.py
├── main.py                            # FastAPI app + lifespan + middlewares
├── deps.py                            # dependency injection (db, current_user)
├── exceptions.py                      # exceções customizadas + handlers
├── constants.py                       # rotas, roles, códigos de erro
├── middleware/
│   ├── __init__.py
│   ├── auth.py                        # validação JWT Supabase
│   ├── cors.py                        # config CORS
│   ├── rate_limit.py                  # rate limiting (genérico)
│   └── error_handler.py               # captura global de exceções
├── schemas/                           # Pydantic v2 (request/response)
│   ├── __init__.py
│   ├── base.py                        # BaseSchema + ResponseEnvelope
│   ├── auth.py
│   ├── ordens.py
│   ├── apontamentos.py
│   ├── indicadores.py
│   ├── uploads.py
│   ├── ia.py
│   ├── colaboradores.py
│   ├── efetivo.py
│   ├── turnos.py
│   └── admin.py
└── v1/                                # versão 1 da API
    ├── __init__.py
    ├── router.py                      # router master (agrega todos)
    ├── health.py
    ├── auth.py
    ├── ordens.py
    ├── apontamentos.py
    ├── indicadores.py
    ├── uploads.py
    ├── ia.py
    ├── colaboradores.py
    ├── efetivo.py
    └── admin.py
```

**Convenções:**
- Cada arquivo em `v1/` = 1 router temático.
- Schemas separados em `schemas/` espelham a estrutura de routers.
- Limite por arquivo: **150 linhas (soft) / 250 (hard)** — se passar, dividir.

---

## 🔑 3. RBAC — 3 PAPÉIS

### 3.1 Hierarquia (com herança)

```
Viewer < Operator < Admin
```

| Role | Permissões |
|---|---|
| **Viewer** 👁️ | Consulta cockpit, indicadores, ordens, apontamentos, raio-X. Interage com IA. |
| **Operator** 🔧 | Tudo de Viewer + upload de planilhas + edição de efetivo. |
| **Admin** 👑 | Tudo de Operator + reset banco + config turnos + reprocessamento + gestão usuários. |

### 3.2 Implementação (Supabase + FastAPI)

```python
# constants.py
from enum import StrEnum

class Role(StrEnum):
    VIEWER = "viewer"
    OPERATOR = "operator"
    ADMIN = "admin"

ROLE_HIERARCHY = {
    Role.VIEWER: 0,
    Role.OPERATOR: 1,
    Role.ADMIN: 2,
}
```

```python
# deps.py
from fastapi import Depends, HTTPException, status
from .middleware.auth import verify_supabase_jwt
from .constants import Role, ROLE_HIERARCHY

def require_role(min_role: Role):
    """Dependency que exige role mínima (com herança)."""
    async def checker(user: dict = Depends(verify_supabase_jwt)):
        user_role = Role(user.get("role", Role.VIEWER))
        if ROLE_HIERARCHY[user_role] < ROLE_HIERARCHY[min_role]:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail={
                    "code": "INSUFFICIENT_ROLE",
                    "message": f"Esta ação requer role {min_role} ou superior.",
                    "required_role": min_role,
                    "user_role": user_role,
                }
            )
        return user
    return checker

# Aliases prontos
require_viewer = require_role(Role.VIEWER)
require_operator = require_role(Role.OPERATOR)
require_admin = require_role(Role.ADMIN)
```

### 3.3 Uso em rotas

```python
# v1/uploads.py
from fastapi import APIRouter, Depends
from ..deps import require_operator

router = APIRouter(prefix="/uploads", tags=["uploads"])

@router.post("/notas", dependencies=[Depends(require_operator)])
async def upload_notas(...):
    ...
```

---

## 🔒 4. AUTENTICAÇÃO (Supabase Auth)

### 4.1 Fluxo

```
┌──────────┐  1. login         ┌──────────────┐
│ Frontend │ ─────────────────>│   Supabase   │
│ (Next.js)│                   │     Auth     │
└──────────┘ <───── JWT ────── └──────────────┘
     │
     │  2. requisição com cookie httpOnly
     ↓
┌──────────┐  3. valida JWT    ┌──────────────┐
│  FastAPI │ ─────────────────>│   Supabase   │
│          │ <─── user data ── │   (JWKS)     │
└──────────┘                   └──────────────┘
```

### 4.2 Middleware de validação

```python
# middleware/auth.py
import jwt
from fastapi import HTTPException, Request, status
from jwt import PyJWKClient
from functools import lru_cache
from ..config import settings

@lru_cache(maxsize=1)
def get_jwks_client() -> PyJWKClient:
    return PyJWKClient(f"{settings.SUPABASE_URL}/auth/v1/.well-known/jwks.json")

async def verify_supabase_jwt(request: Request) -> dict:
    """Valida JWT do Supabase e retorna payload do usuário."""
    token = request.cookies.get("sb-access-token")
    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail={"code": "MISSING_TOKEN", "message": "Token ausente."}
        )
    try:
        signing_key = get_jwks_client().get_signing_key_from_jwt(token).key
        payload = jwt.decode(
            token,
            signing_key,
            algorithms=["RS256", "ES256"],
            audience="authenticated",
            options={"require": ["exp", "sub", "aud"]},
        )
        # Role vem de raw_app_meta_data (configurado no Supabase)
        payload["role"] = payload.get("app_metadata", {}).get("role", "viewer")
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, {"code": "TOKEN_EXPIRED", "message": "Token expirado."})
    except jwt.InvalidTokenError as e:
        raise HTTPException(401, {"code": "INVALID_TOKEN", "message": str(e)})
```

### 4.3 Cookies httpOnly (regra crítica)

- ✅ JWT vai em cookie `sb-access-token` (httpOnly, Secure, SameSite=Lax)
- ❌ **Nunca** retornar JWT no body de resposta
- ❌ **Nunca** aceitar token via header `Authorization` (frontend usa cookie)

---

## 📡 5. ESTRUTURA DE ENDPOINTS

### 5.1 Versionamento

- Prefixo: `/api/v1/`
- Quebra de contrato → criar `/api/v2/` (não modificar v1)

### 5.2 Mapa completo

```
🌍 PÚBLICO (sem auth)
GET    /api/v1/health                          # liveness probe
GET    /api/v1/auth/me                         # retorna user + role (ou 401)

👁️  VIEWER+
GET    /api/v1/ordens                          # lista paginada com filtros
GET    /api/v1/ordens/{numero}                 # detalhe
GET    /api/v1/apontamentos                    # lista paginada

GET    /api/v1/indicadores/maus-atores
GET    /api/v1/indicadores/comparativo-mensal
GET    /api/v1/indicadores/desempenho-geral
GET    /api/v1/indicadores/por-classificacao
GET    /api/v1/indicadores/horas-comparativo   # Plan vs Aprop

GET    /api/v1/colaboradores
GET    /api/v1/colaboradores/{matricula}/raio-x

POST   /api/v1/ia/perguntar
GET    /api/v1/ia/historico/{session_id}
DELETE /api/v1/ia/historico/{session_id}

🔧 OPERATOR+
POST   /api/v1/uploads/notas
POST   /api/v1/uploads/horas
POST   /api/v1/uploads/ordens                  # bloqueado MVP → 501
GET    /api/v1/uploads                         # histórico próprio
PUT    /api/v1/efetivo/{matricula}             # editar 1 colaborador
PATCH  /api/v1/efetivo/bulk                    # bulk edit (auto-save)

👑 ADMIN
DELETE /api/v1/admin/reset-dados               # TD-04
POST   /api/v1/admin/reprocessar               # TD-03
POST   /api/v1/admin/reprocessar-turnos        # TD-08
GET    /api/v1/admin/uploads                   # auditoria global
GET    /api/v1/admin/turnos
PUT    /api/v1/admin/turnos
GET    /api/v1/admin/usuarios
PATCH  /api/v1/admin/usuarios/{id}/role
```

### 5.3 Convenções REST

| Método | Uso |
|---|---|
| `GET` | Leitura idempotente |
| `POST` | Criar / processar (upload, IA, reprocessar) |
| `PUT` | Substituir recurso completo |
| `PATCH` | Atualização parcial / bulk |
| `DELETE` | Remover (sempre exigir confirmação no body) |

### 5.4 Paginação padrão

```python
# Query params
?page=1&page_size=50&order_by=criado_em&order_dir=desc

# Response envelope
{
  "data": [...],
  "pagination": {
    "page": 1,
    "page_size": 50,
    "total": 1234,
    "total_pages": 25
  }
}
```

**Limites:** `page_size` mín=10, máx=200, default=50.

---

## 📦 6. PYDANTIC v2 — SCHEMAS

### 6.1 BaseSchema obrigatório

```python
# schemas/base.py
from pydantic import BaseModel, ConfigDict
from typing import Generic, TypeVar
from datetime import datetime

T = TypeVar("T")

class BaseSchema(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,           # ORM mode (antigo)
        populate_by_name=True,
        str_strip_whitespace=True,
        validate_assignment=True,
    )

class Pagination(BaseSchema):
    page: int
    page_size: int
    total: int
    total_pages: int

class PagedResponse(BaseSchema, Generic[T]):
    data: list[T]
    pagination: Pagination
```

### 6.2 Convenções de schema

- **Sufixos:** `*Create`, `*Update`, `*Response`, `*Filter`
- **Nunca expor** campos sensíveis (`atualizado_por` UUID interno → só nome)
- **Datas:** sempre ISO8601 com timezone (`datetime` Pydantic gera automático)
- **Decimais financeiros:** `Decimal`, nunca `float`

```python
# schemas/ordens.py
from datetime import datetime
from decimal import Decimal
from .base import BaseSchema

class OrdemResponse(BaseSchema):
    numero_ordem: str
    descricao: str | None
    status_canonico: str
    classificacao: str
    turno: str | None
    inicio_avaria: datetime | None
    fim_avaria: datetime | None
    horas_apropriadas: Decimal

class OrdemFilter(BaseSchema):
    status: list[str] | None = None
    classificacao: list[str] | None = None
    turno: list[str] | None = None
    inicio_de: datetime | None = None
    inicio_ate: datetime | None = None
```

### 6.3 Validators

```python
from pydantic import field_validator

class EfetivoUpdate(BaseSchema):
    horas_planejadas_s1: Decimal
    horas_planejadas_s2: Decimal
    horas_planejadas_s3: Decimal
    horas_planejadas_s4: Decimal
    horas_planejadas_s5: Decimal

    @field_validator("*")
    @classmethod
    def horas_nao_negativas(cls, v: Decimal) -> Decimal:
        if v < 0:
            raise ValueError("Horas planejadas não podem ser negativas.")
        if v > Decimal("168"):  # 24h * 7 dias
            raise ValueError("Horas planejadas excedem o máximo semanal (168h).")
        return v
```

---

## 📤 7. UPLOAD DE ARQUIVOS

### 7.1 Regras (alinhado com D12 — Opção A)

1. **Não persistir arquivo em disco** — processa em memória, descarta.
2. **Validação MIME** via `python-magic` (não confiar em `Content-Type`).
3. **Hash SHA-256** do conteúdo → detecta re-upload duplicado.
4. **Limite de tamanho:** 50 MB (configurável).
5. **Streaming** com `UploadFile` — não carregar arquivo inteiro de uma vez.
6. **Tabela `uploads`** registra apenas metadados (hash, tamanho, linhas, status).

### 7.2 Handler padrão

```python
# v1/uploads.py
from fastapi import APIRouter, UploadFile, File, Depends, HTTPException
from ..deps import require_operator, get_db
from ..schemas.uploads import UploadResponse
from backend.core.proc_notas import processar_notas
from backend.core.upload_service import (
    validar_mime, calcular_hash, registrar_upload
)

router = APIRouter(prefix="/uploads", tags=["uploads"])

MAX_FILE_SIZE = 50 * 1024 * 1024  # 50 MB

@router.post(
    "/notas",
    response_model=UploadResponse,
    status_code=201,
    summary="Upload de planilha de Notas SAP",
    dependencies=[Depends(require_operator)],
)
async def upload_notas_endpoint(
    file: UploadFile = File(...),
    db = Depends(get_db),
    user: dict = Depends(require_operator),
):
    # 1. Lê stream com limite
    content = await file.read(MAX_FILE_SIZE + 1)
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(413, {
            "code": "FILE_TOO_LARGE",
            "message": f"Arquivo excede {MAX_FILE_SIZE // 1024 // 1024} MB."
        })

    # 2. Valida MIME
    mime = validar_mime(content)
    if mime not in {"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"}:
        raise HTTPException(415, {
            "code": "INVALID_MIME",
            "message": f"Tipo de arquivo não suportado: {mime}"
        })

    # 3. Hash + detecção de duplicata
    file_hash = calcular_hash(content)

    # 4. Processa via core (lógica fica lá)
    resultado = await processar_notas(content, db=db, user_id=user["sub"])

    # 5. Registra metadados
    upload_id = await registrar_upload(
        db=db,
        hash_=file_hash,
        nome_original=file.filename,
        tamanho_bytes=len(content),
        linhas_processadas=resultado.linhas_ok,
        linhas_erro=resultado.linhas_erro,
        fonte="notas",
        usuario_id=user["sub"],
    )

    return UploadResponse(
        upload_id=upload_id,
        linhas_processadas=resultado.linhas_ok,
        linhas_erro=resultado.linhas_erro,
        warnings=resultado.warnings,
    )
```

---

## 🚨 8. ERROR HANDLING

### 8.1 Formato padrão

```json
{
  "error": {
    "code": "UPLOAD_INVALID_COLUMNS",
    "message": "Planilha não contém colunas obrigatórias.",
    "details": {
      "missing_columns": ["Nota", "Ordem"],
      "received_columns": ["Nº", "Cód"]
    },
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-04-27T11:04:50-03:00"
  }
}
```

### 8.2 Exception customizada

```python
# exceptions.py
from fastapi import HTTPException

class APIError(HTTPException):
    def __init__(
        self,
        status_code: int,
        code: str,
        message: str,
        details: dict | None = None,
    ):
        super().__init__(
            status_code=status_code,
            detail={
                "code": code,
                "message": message,
                "details": details or {},
            }
        )

# Aliases comuns
class NotFoundError(APIError):
    def __init__(self, resource: str, id_: str):
        super().__init__(404, "NOT_FOUND", f"{resource} '{id_}' não encontrado.")

class ValidationError(APIError):
    def __init__(self, message: str, details: dict | None = None):
        super().__init__(422, "VALIDATION_ERROR", message, details)

class BusinessRuleError(APIError):
    def __init__(self, code: str, message: str, details: dict | None = None):
        super().__init__(400, code, message, details)
```

### 8.3 Handler global

```python
# middleware/error_handler.py
import uuid
import logging
from datetime import datetime, timezone
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

logger = logging.getLogger(__name__)

def register_error_handlers(app: FastAPI) -> None:
    @app.exception_handler(Exception)
    async def unhandled_exception_handler(request: Request, exc: Exception):
        trace_id = str(uuid.uuid4())
        logger.exception("Erro não tratado", extra={"trace_id": trace_id})
        return JSONResponse(
            status_code=500,
            content={
                "error": {
                    "code": "INTERNAL_ERROR",
                    "message": "Erro interno. Contate o suporte.",
                    "details": {},
                    "trace_id": trace_id,
                    "timestamp": datetime.now(timezone.utc).isoformat(),
                }
            }
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        return JSONResponse(
            status_code=422,
            content={
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Dados de entrada inválidos.",
                    "details": {"errors": exc.errors()},
                    "trace_id": str(uuid.uuid4()),
                    "timestamp": datetime.now(timezone.utc).isoformat(),
                }
            }
        )
```

### 8.4 Códigos de erro canônicos

```python
# constants.py
class ErrorCode(StrEnum):
    # Auth
    MISSING_TOKEN = "MISSING_TOKEN"
    INVALID_TOKEN = "INVALID_TOKEN"
    TOKEN_EXPIRED = "TOKEN_EXPIRED"
    INSUFFICIENT_ROLE = "INSUFFICIENT_ROLE"

    # Upload
    FILE_TOO_LARGE = "FILE_TOO_LARGE"
    INVALID_MIME = "INVALID_MIME"
    DUPLICATE_UPLOAD = "DUPLICATE_UPLOAD"
    UPLOAD_INVALID_COLUMNS = "UPLOAD_INVALID_COLUMNS"
    UPLOAD_PROCESSING_FAILED = "UPLOAD_PROCESSING_FAILED"

    # Negócio
    SOURCE_MISMATCH = "SOURCE_MISMATCH"          # mistura notas + ordens
    LAYOUT_NOT_SUPPORTED = "LAYOUT_NOT_SUPPORTED"  # MVP: só notas
    MTTR_NOT_AVAILABLE = "MTTR_NOT_AVAILABLE"

    # Genéricos
    NOT_FOUND = "NOT_FOUND"
    VALIDATION_ERROR = "VALIDATION_ERROR"
    INTERNAL_ERROR = "INTERNAL_ERROR"
```

---

## 🌍 9. CORS

```python
# middleware/cors.py
from fastapi.middleware.cors import CORSMiddleware
from ..config import settings

def setup_cors(app):
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ALLOWED_ORIGINS,  # ['https://app.exemplo.com']
        allow_credentials=True,                        # cookies httpOnly
        allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
        allow_headers=["Content-Type", "X-Requested-With"],
        expose_headers=["X-Trace-Id"],
        max_age=3600,
    )
```

**Regras:**
- ❌ **Nunca** `allow_origins=["*"]` em produção
- ✅ `allow_credentials=True` é obrigatório (cookies)
- ✅ Lista de origens vem do `.env`

---

## ⏱️ 10. RATE LIMITING

```python
# middleware/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=lambda r: r.state.user_id if hasattr(r.state, "user_id") else get_remote_address(r)
)

# Limites por tipo de endpoint
RATE_LIMITS = {
    "default": "100/minute",
    "upload": "10/minute",
    "ia": "30/hour",        # alinhado com SECURITY.md
    "admin": "20/minute",
}
```

**Aplicação:**

```python
@router.post("/perguntar")
@limiter.limit(RATE_LIMITS["ia"])
async def perguntar_ia(...):
    ...
```

---

## 🔌 11. DEPENDENCY INJECTION

```python
# deps.py
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from backend.db.session import async_session_maker

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

# Aliases já documentados em §3.2
# require_viewer, require_operator, require_admin
```

---

## 🔄 12. LIFESPAN

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from backend.db.session import engine
from backend.ia.agent_factory import warmup_ia_agent

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await warmup_ia_agent()                # pré-aquece agent IA
    yield
    # Shutdown
    await engine.dispose()                  # fecha pool de conexões

app = FastAPI(
    title="IA-Mantain API",
    version="1.0.0",
    description="Backend de gestão de manutenção SAP com IA conversacional.",
    lifespan=lifespan,
    docs_url="/api/docs",
    redoc_url="/api/redoc",
    openapi_url="/api/openapi.json",
)
```

---

## 📖 13. OPENAPI / SWAGGER

### 13.1 Configuração obrigatória por rota

```python
@router.get(
    "/ordens/{numero}",
    response_model=OrdemResponse,
    summary="Detalhe de uma ordem",
    description="Retorna ordem completa incluindo apontamentos de horas associados.",
    responses={
        200: {"description": "Ordem encontrada"},
        404: {"description": "Ordem não encontrada"},
        401: {"description": "Não autenticado"},
    },
    tags=["ordens"],
)
async def get_ordem(numero: str, ...):
    ...
```

### 13.2 Tags organizadas

```python
# main.py
tags_metadata = [
    {"name": "health", "description": "Liveness/readiness probes"},
    {"name": "auth", "description": "Autenticação e perfil do usuário"},
    {"name": "ordens", "description": "Ordens de manutenção SAP"},
    {"name": "apontamentos", "description": "Apontamentos de horas"},
    {"name": "indicadores", "description": "KPIs e gráficos do cockpit"},
    {"name": "uploads", "description": "Importação de planilhas SAP"},
    {"name": "ia", "description": "Assistente IA (SQL Agent)"},
    {"name": "colaboradores", "description": "Equipe e raio-X"},
    {"name": "efetivo", "description": "Planejamento de efetivo"},
    {"name": "admin", "description": "Operações administrativas"},
]

app.openapi_tags = tags_metadata
```

### 13.3 Acesso

- Swagger UI: `https://api.exemplo.com/api/docs`
- ReDoc: `https://api.exemplo.com/api/redoc`
- Em produção: **proteger com basic auth** (configurável via env)

---

## 🧪 14. TESTES

### 14.1 Stack

- **pytest** + **pytest-asyncio**
- **httpx.AsyncClient** (cliente HTTP async)
- **pytest-postgresql** ou **testcontainers** (banco real isolado)
- **factory-boy** (fixtures)

### 14.2 Estrutura

```
tests/
├── conftest.py                       # fixtures globais (db, client, users)
├── api/
│   ├── test_auth.py
│   ├── test_ordens.py
│   ├── test_uploads.py
│   ├── test_indicadores.py
│   ├── test_ia.py
│   ├── test_efetivo.py
│   └── test_admin.py
└── factories/
    ├── ordens.py
    └── apontamentos.py
```

### 14.3 Fixtures essenciais

```python
# conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from backend.api.main import app

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c

@pytest.fixture
async def viewer_client(client):
    """Client autenticado como Viewer."""
    client.cookies.set("sb-access-token", _make_jwt(role="viewer"))
    return client

@pytest.fixture
async def operator_client(client):
    client.cookies.set("sb-access-token", _make_jwt(role="operator"))
    return client

@pytest.fixture
async def admin_client(client):
    client.cookies.set("sb-access-token", _make_jwt(role="admin"))
    return client
```

### 14.4 Padrão de teste por endpoint

Para **toda rota nova**, escrever pelo menos:

1. ✅ Caso feliz (200/201)
2. ✅ Não autenticado (401)
3. ✅ Role insuficiente (403)
4. ✅ Validação de input (422)
5. ✅ Recurso inexistente (404) — quando aplicável

```python
# tests/api/test_uploads.py
async def test_upload_notas_sucesso(operator_client, planilha_notas_valida):
    response = await operator_client.post(
        "/api/v1/uploads/notas",
        files={"file": ("notas.xlsx", planilha_notas_valida, "application/...")}
    )
    assert response.status_code == 201
    assert response.json()["linhas_processadas"] > 0

async def test_upload_notas_viewer_proibido(viewer_client, planilha_notas_valida):
    response = await viewer_client.post(
        "/api/v1/uploads/notas",
        files={"file": ("notas.xlsx", planilha_notas_valida, "...")}
    )
    assert response.status_code == 403
    assert response.json()["error"]["code"] == "INSUFFICIENT_ROLE"

async def test_upload_arquivo_grande_demais(operator_client, arquivo_60mb):
    response = await operator_client.post(
        "/api/v1/uploads/notas",
        files={"file": ("big.xlsx", arquivo_60mb, "...")}
    )
    assert response.status_code == 413
    assert response.json()["error"]["code"] == "FILE_TOO_LARGE"
```

### 14.5 Cobertura mínima

- **Routers:** 80%
- **Schemas:** 70% (validators críticos = 100%)
- **Middlewares:** 90%

---

## ⏳ 15. PENDÊNCIAS (sessão dedicada de Horas & Efetivo)

> Os endpoints `/efetivo/*`, `/colaboradores/*` e `/indicadores/horas-comparativo` têm **estrutura genérica** definida.
> Detalhes específicos pendem da sessão dedicada do módulo Horas & Efetivo.

| ID | Pendência | Decisão necessária |
|---|---|---|
| **H1** | UI de edição de efetivo | Tabela editável (TanStack Table) ou form? *(impacto: schema PATCH bulk)* |
| **H2** | Estratégia de save | Auto-save com debounce ou botão "Salvar"? *(impacto: rate limiting)* |
| **H3** | Cálculo Plan_Mes | Manter regra S1-S4 + S5 parcial ou mês corrido? *(impacto: schema)* |
| **H4** | Visões mensal vs semanal | Manter as 2 ou só uma? *(impacto: query params)* |
| **H5** | Métricas do raio-X | Confirmar: Regime, Planejado, Apropriado, Saldo? |
| **H6** | Filtros do drill-down | Por semana, ordem, equipamento? |

---

## ✅ 16. CHECKPOINTS DE CONSISTÊNCIA

Antes de fazer commit em `backend/api/`, verificar:

- [ ] Toda rota nova tem teste em `tests/api/`
- [ ] Toda rota tem `summary` + `tags` + `responses` no decorator
- [ ] Schemas Pydantic v2 (não v1)
- [ ] Endpoint async (`async def`)
- [ ] RBAC aplicado (`Depends(require_*)`)
- [ ] Erros usam `APIError` (não `HTTPException` cru)
- [ ] Códigos de erro estão em `constants.ErrorCode`
- [ ] Sem hardcode de prefixo de rota (usar `constants.ROUTES`)
- [ ] Sem lógica de negócio no router (delegar a `core/`)
- [ ] Limite de linhas respeitado (150/250)
- [ ] CORS testado com cookie httpOnly
- [ ] Documentação OpenAPI renderiza sem warnings

---

## 🔗 17. REFERÊNCIAS CRUZADAS

- `backend/AGENTS.md` — visão geral do backend
- `backend/core/AGENTS.md` — lógica de negócio (delegada via API)
- `backend/ia/AGENTS.md` — endpoints `/ia/*` delegam pra cá
- `backend/db/AGENTS.md` — modelos SQLAlchemy usados nos schemas
- `docs/SECURITY.md` — políticas de auth, JWT, rate limit
- `docs/DATA_CONTRACT.md` — formatos esperados nos uploads
- `PROJECT_STATE.md` — decisões D10–D15 e tech debts TD-04, TD-08
