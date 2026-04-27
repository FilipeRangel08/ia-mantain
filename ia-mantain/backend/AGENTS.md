# 🐍 Backend — FastAPI + Supabase + LangChain

> **Escopo:** Este arquivo se aplica a TODO arquivo dentro de `backend/`.
> **Stack:** Python 3.11+ · FastAPI · SQLAlchemy 2.0 (async) · Pydantic v2 · Alembic · LangChain · Google Gemini · Supabase
> **Última atualização:** 2026-04-25

---

## 1. 🎯 RESPONSABILIDADE DO BACKEND

O backend é o **cérebro** do sistema. Ele:

- ✅ Processa planilhas Excel do SAP (ETL completo)
- ✅ Persiste dados no Supabase (Postgres)
- ✅ Expõe API REST consumida pelo frontend
- ✅ Valida JWTs do Supabase Auth (middleware)
- ✅ Aplica regras de negócio (classificação, turnos, KPIs)
- ✅ Hospeda o agente IA (SQL Agent + geração de ECharts)

O backend **NÃO**:

- ❌ Renderiza HTML (isso é Next.js)
- ❌ Mantém estado de sessão de UI (cada request é independente)
- ❌ Confia em dados validados só no frontend (revalida tudo)
- ❌ Acessa o SAP diretamente (consome arquivos exportados pelo usuário)

---

## 2. 🗂️ ESTRUTURA DE PASTAS

```
backend/
├── AGENTS.md                     ← VOCÊ ESTÁ AQUI
│
├── api/                          ← Routers REST (ver api/AGENTS.md)
│   ├── routers/
│   │   ├── auth.py               ← Login, signup, logout
│   │   ├── ordens.py             ← CRUD de ordens SAP
│   │   ├── apontamentos.py       ← Horas apontadas
│   │   ├── upload.py             ← Recebe planilhas Excel
│   │   ├── ia.py                 ← Endpoint do agente IA
│   │   └── admin.py              ← Gestão de usuários (Supervisor)
│   ├── deps.py                   ← Dependências (auth, db, papel)
│   └── main.py                   ← App FastAPI + middlewares
│
├── core/                         ← Pipeline ETL (ver core/AGENTS.md)
│   ├── constants.py              ← 🚨 NOMES CANÔNICOS DE COLUNAS SAP
│   ├── proc_horas.py             ← Processa apontamentos de horas
│   ├── proc_ordens.py            ← Processa ordens (abertas + encerradas)
│   ├── proc_unifica.py           ← Outer join: horas + ordens
│   ├── classificacao.py          ← Mapeia descrição → tipo de ordem
│   ├── turnos.py                 ← Regras de enquadramento horário
│   └── datas.py                  ← Parser dual-format (ISO + BR)
│
├── ia/                           ← Agente IA (ver ia/AGENTS.md)
│   ├── agent.py                  ← create_sql_agent + memória
│   ├── prompts.py                ← System prompts versionados
│   ├── echarts_generator.py      ← Converte resposta IA → JSON ECharts
│   └── memory.py                 ← ConversationBufferWindowMemory
│
├── db/                           ← Banco (ver db/AGENTS.md)
│   ├── session.py                ← AsyncSession + engine
│   ├── models.py                 ← SQLAlchemy ORM models
│   ├── migrations/               ← Alembic migrations
│   └── seeds/                    ← Dados iniciais (papéis, etc.)
│
├── schemas/                      ← Modelos Pydantic
│   ├── ordem.py
│   ├── apontamento.py
│   ├── usuario.py
│   └── ia.py                     ← Request/Response do chat IA
│
├── tests/                        ← Testes pytest
├── .env.example
├── requirements.txt
└── pyproject.toml
```

---

## 3. ⛔ REGRAS CRÍTICAS DO BACKEND

### 3.1 Camada Anticorrupção SAP (constantes canônicas)

> 🚨 **REGRA NÚMERO UM DO BACKEND**

O SAP exporta planilhas com nomes de colunas **horríveis e instáveis**:

- `'Centro trab.respons.'` (com acento + abreviação + ponto final)
- `'Denominação do loc.instalação'` (mistura português + abreviação)
- `'Trab.real'`, `'Tipo de ordem'`, etc.

**Estes nomes mudam quando:**

- O SAP é atualizado
- A configuração da exportação muda
- O usuário troca o idioma da interface

**Solução: Centralizar TUDO em `core/constants.py`**

```python
# backend/core/constants.py
"""
🚨 ÚNICA fonte da verdade para nomes de colunas SAP.
Quando o SAP mudar nomes, SÓ este arquivo precisa ser atualizado.
"""

# === COLUNAS RAW (vindas direto da planilha SAP) ===
COL_ORDEM = 'Ordem'
COL_DESCRICAO = 'Descrição'
COL_LOCAL_INSTALACAO = 'Denominação do loc.instalação'
COL_CENTRO_TRABALHO = 'Centro trab.respons.'
COL_STATUS_SAP = 'Status do sistema'
COL_TIPO_ORDEM = 'Tipo de ordem'
COL_TRABALHO_REAL = 'Trab.real'
COL_DATA_INICIO = 'Data início real'
# ... (lista completa)

# === MAPEAMENTOS (RAW → CALC) ===
RENAME_MAP = {
    COL_TRABALHO_REAL: 'trabalho_real',
    COL_LOCAL_INSTALACAO: 'local_instalacao',
    # ...
}

# === TIPOS DE ORDEM CONHECIDOS ===
TIPOS_ORDEM_VALIDOS = {'COR', 'PREV', 'IG', 'PP', 'CRI', 'INFRA', 'FAB'}
```

**Regras de uso:**

```python
# ✅ CORRETO
from backend.core.constants import COL_CENTRO_TRABALHO, COL_TRABALHO_REAL

df_filtrado = df[df[COL_CENTRO_TRABALHO] == 'SL-MEC']
total = df[COL_TRABALHO_REAL].sum()

# ❌ ERRADO — hardcoded
df_filtrado = df[df['Centro trab.respons.'] == 'SL-MEC']  # 🚨
total = df['Trab.real'].sum()                              # 🚨
```

### 3.2 Padrão Dual-Format para Datas

> 🐛 **Bug histórico:** projeto anterior teve **56% de NaT** (datas inválidas) em produção.

**Causa:** uso de `pd.to_datetime(df[col], dayfirst=True)` sozinho falhava com:

- Datas vindas do banco (formato ISO `2026-01-05T00:00:00`)
- Datas vindas do Excel BR (formato `05/01/2026`)

**Solução obrigatória — sempre em `core/datas.py`:**

```python
# backend/core/datas.py
import pandas as pd

def parse_data_dual_format(serie: pd.Series) -> pd.Series:
    """
    Parser robusto: tenta ISO8601 primeiro, fallback para formato BR.

    Por quê: dados podem vir do banco (ISO) ou do Excel BR (dd/mm/yyyy).
    Usar dayfirst=True sozinho falha com ISO.
    """
    # 1ª tentativa: ISO8601 (do banco)
    datas = pd.to_datetime(serie, format='ISO8601', errors='coerce')

    # 2ª tentativa: BR (do Excel) — só onde a 1ª falhou
    mask_nat = datas.isna() & serie.notna()
    if mask_nat.any():
        datas[mask_nat] = pd.to_datetime(
            serie[mask_nat],
            dayfirst=True,
            errors='coerce'
        )

    return datas
```

> ❌ **NUNCA** chame `pd.to_datetime` direto fora de `core/datas.py`.
> ✅ **SEMPRE** use `parse_data_dual_format()`.

### 3.3 Colunas RAW vs CALC

**RAW** = vem da planilha SAP, persiste no banco.
**CALC** = derivada em runtime, **NUNCA** persiste.

```python
# ❌ ERRADO — persistir CALC no banco
class Ordem(Base):
    semana_trabalho = Column(Integer)  # 🚨 isso é CALC!
    classificacao = Column(String)     # 🚨 isso é CALC!

# ✅ CORRETO — só RAW no banco
class Ordem(Base):
    numero = Column(String, primary_key=True)
    descricao = Column(String)
    data_inicio = Column(DateTime)
    # CALC é gerado quando o frontend pede
```

**Lista de colunas CALC** (regerar SEMPRE ao carregar):

- `Data_Calc`, `Semana_Trabalho`, `Dia`, `Semana`, `Mês`, `Mes_Ano`
- `Classificacao_Ordem`, `Status_SAP_Normalizado`
- `Plan_Sn`, `Aprop_Sn`, `Nome_Exibicao`

Detalhes em `docs/DATA_CONTRACT.md`.

### 3.4 IA = SQL Agent (NUNCA Pandas Agent)

```python
# ✅ CORRETO
from langchain_community.agent_toolkits import create_sql_agent
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri(SUPABASE_DB_URL)
agent = create_sql_agent(llm=gemini, db=db, ...)

# ❌ ERRADO
from langchain_experimental.agents import create_pandas_dataframe_agent
agent = create_pandas_dataframe_agent(llm, df, ...)  # 🚨 estoura tokens
```

**Por quê?**

- Pandas Agent serializa DataFrame inteiro como string no prompt → custo absurdo
- SQL Agent gera SQL dinâmico → executa direto no Postgres → eficiente

Ver `ia/AGENTS.md` para padrões detalhados.

### 3.5 RBAC — Validação Backend Obrigatória

> ❌ **NUNCA** confie só na validação do frontend.
> ✅ **TODA** rota protegida valida JWT + papel via dependency.

```python
# backend/api/deps.py
from fastapi import Depends, HTTPException, status
from typing import Annotated

async def get_usuario_atual(token: str = Depends(oauth2_scheme)) -> Usuario:
    """Valida JWT do Supabase e retorna o usuário."""
    # ... validação ...

def requer_papel(*papeis_permitidos: str):
    """Factory de dependency que valida papel do usuário."""
    async def dep(usuario: Usuario = Depends(get_usuario_atual)) -> Usuario:
        if usuario.papel not in papeis_permitidos:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Requer papel: {papeis_permitidos}"
            )
        return usuario
    return dep

# Uso em routers:
@router.post("/upload")
async def upload_planilha(
    arquivo: UploadFile,
    usuario: Annotated[Usuario, Depends(requer_papel("assistente", "supervisor"))]
):
    ...
```

---

## 4. 🌐 PADRÕES DE API

### 4.1 Estrutura de um Router

```python
# backend/api/routers/ordens.py
from fastapi import APIRouter, Depends, HTTPException
from typing import Annotated
from backend.schemas.ordem import OrdemResponse, OrdemFiltros
from backend.api.deps import get_db, get_usuario_atual
from backend.db.models import Usuario

router = APIRouter(prefix="/ordens", tags=["ordens"])

@router.post("/", response_model=list[OrdemResponse])
async def listar_ordens(
    filtros: OrdemFiltros,
    db: Annotated[AsyncSession, Depends(get_db)],
    usuario: Annotated[Usuario, Depends(get_usuario_atual)],
) -> list[OrdemResponse]:
    """Lista ordens SAP com filtros opcionais."""
    # 1. Aplica filtros
    # 2. Aplica RLS (Row-Level Security via Supabase)
    # 3. Retorna paginado
    ...
```

### 4.2 Schemas Pydantic

```python
# backend/schemas/ordem.py
from pydantic import BaseModel, Field
from datetime import datetime

class OrdemFiltros(BaseModel):
    """Filtros para busca de ordens."""
    data_inicio: datetime | None = None
    data_fim: datetime | None = None
    centro_trabalho: str | None = None
    tipo_ordem: list[str] | None = Field(default=None, max_length=10)

class OrdemResponse(BaseModel):
    """Ordem retornada pela API (com colunas CALC já computadas)."""
    numero: str
    descricao: str
    classificacao: str          # CALC
    semana_trabalho: int        # CALC
    horas_apontadas: float
    status_normalizado: str     # CALC

    class Config:
        from_attributes = True  # ORM mode
```

### 4.3 Versionamento da API

- Prefixo global: `/api/v1`
- Mudanças breaking → `/api/v2`
- Deprecation explícita com header `X-Deprecated`

---

## 5. 🔐 AUTENTICAÇÃO (Supabase Auth)

### 5.1 Fluxo

```
1. Frontend faz login via Supabase Auth client
2. Supabase retorna JWT
3. Frontend envia JWT em todo request: `Authorization: Bearer <jwt>`
4. Backend valida JWT (assinatura + expiração)
5. Backend extrai user_id do JWT
6. Backend busca papel do usuário no banco
7. Backend autoriza/nega
```

### 5.2 Middleware de Validação

```python
# backend/api/main.py
from fastapi import FastAPI
from backend.api.middleware import jwt_middleware

app = FastAPI(title="IA Mantain API", version="1.0")
app.middleware("http")(jwt_middleware)

# CORS apenas para o frontend
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=[FRONTEND_URL],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 5.3 RLS no Supabase

> ✅ Habilitar Row-Level Security em **todas** as tabelas
> ✅ Backend usa `service_role` key (bypass RLS) com validação manual de papel
> ✅ Frontend (caso acesse direto, ex: realtime) usa `anon` key (RLS aplicado)

---

## 6. 🗃️ BANCO DE DADOS (SQLAlchemy 2.0 async)

### 6.1 Sessão Async

```python
# backend/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
import os

engine = create_async_engine(
    os.getenv("DATABASE_URL"),  # postgresql+asyncpg://...
    echo=False,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

### 6.2 Migrations com Alembic

> ❌ **NUNCA** altere schema do banco direto via SQL ad-hoc
> ✅ **SEMPRE** crie migration: `alembic revision --autogenerate -m "descricao"`

```bash
# Criar migration
alembic revision --autogenerate -m "adiciona_tabela_ordens"

# Aplicar
alembic upgrade head

# Reverter última
alembic downgrade -1
```

### 6.3 Models (apenas RAW)

```python
# backend/db/models.py
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from datetime import datetime

class Base(DeclarativeBase):
    pass

class Ordem(Base):
    __tablename__ = "ordens"

    numero: Mapped[str] = mapped_column(primary_key=True)
    descricao: Mapped[str]
    centro_trabalho: Mapped[str]
    data_inicio: Mapped[datetime | None]
    # ⚠️ Apenas colunas RAW. NUNCA classificacao, semana_trabalho, etc.
```

---

## 7. 📦 PIPELINE ETL (core/)

### 7.1 Princípio: Funções puras + composição

```python
# ❌ ERRADO — God function de 400 linhas
def processar_planilha(arquivo):
    df = pd.read_excel(arquivo)
    # 50 linhas de limpeza
    # 80 linhas de classificação
    # 120 linhas de unificação
    # 150 linhas de cálculos
    return df

# ✅ CORRETO — composição de funções pequenas
def processar_planilha(arquivo: BinaryIO) -> pd.DataFrame:
    df_raw = ler_excel(arquivo)
    df_limpo = limpar_dados(df_raw)
    df_horas = proc_horas(df_limpo)
    df_ordens = proc_ordens(df_limpo)
    df_unificado = unificar(df_horas, df_ordens)
    return df_unificado
```

Cada função:

- ✅ Faz **uma coisa** bem feita
- ✅ Tem **type hints** completos
- ✅ Tem **docstring** explicando *por quê* (não o quê)
- ✅ É **testável** isoladamente

### 7.2 Outer Join Obrigatório

```python
# ✅ CORRETO
df_completo = pd.merge(
    df_horas,
    df_ordens,
    on='numero_ordem',
    how='outer',  # 🚨 NUNCA 'left'
    indicator=True
)

# ❌ ERRADO — perde ordens sem apontamento
df_completo = pd.merge(df_horas, df_ordens, on='numero_ordem', how='left')
```

Razão: `BUG-003` do projeto anterior. Detalhes em `docs/HISTORICAL_BUGS.md`.

---

## 8. 🤖 AGENTE IA (resumo — detalhes em ia/AGENTS.md)

```python
# Estrutura geral
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_community.agent_toolkits import create_sql_agent

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash-exp", temperature=0)
db = SQLDatabase.from_uri(DATABASE_URL)
agent = create_sql_agent(
    llm=llm,
    db=db,
    agent_type="tool-calling",
    verbose=True,
    extra_tools=[gerar_grafico_echarts],  # tool customizada
)

# Memória conversacional (k=3 mensagens)
memory = ConversationBufferWindowMemory(k=3, return_messages=True)
```

**Regras:**

- ✅ Memória de 3 mensagens (não infinita)
- ✅ Cache do agente em `app.state` (recriar só se schema mudar)
- ✅ System prompt versionado em `ia/prompts.py`
- ✅ Tool customizada para gerar JSON ECharts

---

## 9. 🧪 TESTES

### Stack

- **pytest** + **pytest-asyncio**
- **httpx** para testar endpoints
- **pytest-cov** para cobertura

### Estrutura

```python
# backend/tests/test_classificacao.py
import pytest
from backend.core.classificacao import classificar_ordem

@pytest.mark.parametrize("descricao,esperado", [
    ("MANUTENÇÃO CORRETIVA BOMBA P-101", "COR"),
    ("INSPEÇÃO PREVENTIVA SEMESTRAL", "PREV"),
    ("FABRICAÇÃO DE FLANGE 4 POLEGADAS", "FAB"),
])
def test_classificar_ordem(descricao, esperado):
    assert classificar_ordem(descricao) == esperado
```

### Cobertura mínima

- `core/`: **80%** (lógica crítica)
- `api/routers/`: **70%**
- `ia/`: **50%** (LLM é difícil de testar)

---

## 10. 🚨 TRATAMENTO DE ERROS

### 10.1 Hierarquia de Exceções Customizadas

```python
# backend/core/exceptions.py
class IAManutencaoException(Exception):
    """Exceção base do projeto."""
    pass

class PlanilhaInvalidaError(IAManutencaoException):
    """Planilha SAP com formato inválido."""
    pass

class ColunaSAPAusenteError(PlanilhaInvalidaError):
    """Coluna esperada do SAP não encontrada."""
    def __init__(self, coluna: str):
        super().__init__(f"Coluna '{coluna}' não encontrada na planilha SAP")
```

### 10.2 Handler Global

```python
# backend/api/main.py
@app.exception_handler(IAManutencaoException)
async def ia_exception_handler(request, exc: IAManutencaoException):
    return JSONResponse(
        status_code=422,
        content={"erro": str(exc), "tipo": exc.__class__.__name__}
    )
```

### 10.3 Mensagens de Erro

- ✅ Em **português**, claras, acionáveis
- ❌ Nunca exponha stack traces ao usuário final
- ✅ Logs internos em inglês (padrão dev)

---

## 11. 📝 LOGGING

```python
# backend/core/logger.py
import logging
import sys

def configurar_logger():
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s | %(levelname)-8s | %(name)s | %(message)s',
        stream=sys.stdout,
    )

# Uso
logger = logging.getLogger(__name__)
logger.info(f"Processando planilha com {len(df)} linhas")
logger.error("Falha ao classificar ordem", exc_info=True)
```

> ❌ **NUNCA** use `print()` em código de produção.
> ✅ **SEMPRE** use `logger`.

---

## 12. ⚙️ VARIÁVEIS DE AMBIENTE

```bash
# .env (NUNCA commitar)
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...
SUPABASE_JWT_SECRET=...
GOOGLE_API_KEY=AIza...
FRONTEND_URL=http://localhost:3000
ENVIRONMENT=development  # development | staging | production
```

> ✅ **SEMPRE** acesse via `os.getenv("VAR")` ou Pydantic Settings
> ❌ **NUNCA** hardcode credenciais no código

```python
# backend/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    google_api_key: str
    supabase_url: str
    environment: str = "development"

    class Config:
        env_file = ".env"

settings = Settings()  # singleton
```

---

## 13. 📏 LIMITES DE TAMANHO

| Tipo | Soft ⚠️ | Hard 🛑 |
|---|---|---|
| Routers (.py) | 150 | 250 |
| Schemas Pydantic | 200 | 300 |
| ETL/processamento | 250 | 400 |
| Models SQLAlchemy | 200 | 300 |
| Testes | 400 | 600 |

**Sinais de refatoração:**

- 🚩 Função com mais de 50 linhas
- 🚩 Função com mais de 4 parâmetros
- 🚩 Aninhamento de mais de 3 `if/for`
- 🚩 Mesma lógica em 2+ lugares

---

## 14. ⛔ ANTI-PATTERNS DO BACKEND

| ❌ Não faça | ✅ Faça |
|---|---|
| `print()` para debug | `logger.info()` |
| `pd.to_datetime(x, dayfirst=True)` | `parse_data_dual_format(x)` |
| Hardcode de coluna SAP | Importar de `core/constants.py` |
| Endpoint sem `response_model` | Sempre tipar resposta |
| `def` em rota async | `async def` |
| `requests.get()` em endpoint | `httpx.AsyncClient` |
| Senha em plain text | Hash via Supabase Auth |
| SQL string concatenado | SQLAlchemy ORM ou parâmetros |
| `.commit()` sem try/except | Use context manager async |
| Pandas Agent na IA | SQL Agent, sempre |
| Passar DataFrame inteiro pra IA | IA gera SQL, executa no banco |
| Lógica de negócio em router | Extrair pra `core/` |

---

## 15. 📚 REFERÊNCIAS RÁPIDAS

| Quando precisar de... | Veja |
|---|---|
| Padrões globais | `../AGENTS.md` |
| Schema do banco | `../docs/DATA_CONTRACT.md` |
| Regras de negócio | `../docs/BUSINESS_RULES.md` |
| Bugs históricos | `../docs/HISTORICAL_BUGS.md` |
| Constantes SAP | `core/constants.py` |
| FastAPI docs | <https://fastapi.tiangolo.com> |
| SQLAlchemy 2.0 async | <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html> |
| LangChain SQL Agent | <https://python.langchain.com/docs/integrations/toolkits/sql_database> |

---

## 16. 🎯 PRINCÍPIOS NORTEADORES DO BACKEND

1. **Falhe rápido, falhe alto** — Erros na entrada (validação Pydantic), não no meio do pipeline
2. **Idempotência** — Reprocessar a mesma planilha não duplica dados
3. **Observabilidade** — Tudo importante vai pro log com contexto
4. **Imutabilidade quando possível** — Funções puras > efeitos colaterais
5. **Async por padrão** — IO bloqueante mata performance
6. **Tipagem é documentação** — Type hints são obrigatórios

---

> 🐍 **Lembrete:** O backend é o **guardião dos dados**. Cada decisão técnica aqui afeta a integridade do que o usuário vê. Quando em dúvida entre "rápido de fazer" e "correto", escolha **correto**. A dívida técnica do projeto anterior cobrou caro.
