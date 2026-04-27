# 🛡️ AGENTS.md — backend/core

> **Escopo:** Lógica pura de domínio. Sem I/O, sem framework, sem dependências externas além de `pandas` e bibliotecas padrão.
> **Filosofia:** Esta é a camada mais estável do projeto. Tudo aqui deve ser testável de forma isolada, sem mocks de banco/API.

---

## 1. 🎯 Propósito da Camada

`backend/core` é o **coração do domínio**. Contém:

- ✅ Constantes e enums do domínio (nomes de colunas SAP, status canônicos, fontes)
- ✅ Funções puras de transformação (limpar, renomear, classificar, enquadrar turno)
- ✅ Derivadores (regras que transformam dados RAW em campos canônicos)
- ✅ Pipelines de processamento por fonte (`proc_notas`, `proc_ordens`, `proc_horas`)
- ❌ **NÃO contém:** acesso a banco, chamadas HTTP, leitura de arquivo, prompts de IA

> 💡 **Regra de ouro:** se você precisa de mock para testar, está fora de lugar. Mover para `backend/api` ou `backend/ia`.

---

## 2. 📁 Estrutura de Arquivos

```
backend/core/
├── AGENTS.md                # este arquivo
├── __init__.py
├── constants.py             # nomes de colunas, enums, mapas de rename
├── classificacao.py         # regex → TipoOrdem
├── turnos.py                # datetime → Turno (A/B/C/ADM)
├── derivadores.py           # status_canonico, data_canonica, etc
├── proc_notas.py            # pipeline da planilha de NOTAS
├── proc_ordens.py           # pipeline da planilha de ORDENS (futuro)
├── proc_horas.py            # pipeline da planilha de HORAS
└── tests/
    ├── test_classificacao.py
    ├── test_turnos.py
    ├── test_derivadores.py
    ├── test_proc_notas.py
    └── test_proc_horas.py
```

---

## 3. 🔄 Camada Anticorrupção (ACL)

### Problema

Os nomes das colunas vindas do SAP são **horríveis** e inconsistentes:
- `'Texto breve'` (ordens) vs `'Descrição'` (notas) → mesmo conceito
- `'Centro trab.respons.'` com pontos
- `'Nº pessoal'` com caractere especial
- `'TextPrioridade'` sem espaço

### Solução

**Toda coluna RAW SAP é renomeada IMEDIATAMENTE** ao entrar no sistema, via mapa centralizado em `constants.py`. A partir desse ponto, o resto do código só vê nomes em `snake_case` consistentes.

### Regras

- ✅ Use `ColNotas.X`, `ColOrdens.X`, `ColHoras.X` para referenciar nomes RAW
- ✅ Use `RENAME_NOTAS`, `RENAME_ORDENS`, `RENAME_HORAS` no início do pipeline
- ❌ **NUNCA** escreva `df['Texto breve']` no meio do código — use a constante
- ❌ **NUNCA** assuma que coluna existe — sempre verifique presença

---

## 4. 📊 Categorias de Campos

Cada campo no sistema pertence a **uma de três categorias**:

| Categoria | Persiste no banco? | Origem | Exemplos |
|---|---|---|---|
| **RAW** | ✅ Sim | Vem direto da planilha | `numero_ordem`, `descricao`, `fim_avaria` |
| **DERIVED** | ✅ Sim | Calculado por `derivadores.py` | `status_canonico`, `classificacao`, `turno` |
| **CALC** | ❌ Não | Calculado em runtime | `mes_ano`, `semana_trabalho`, `dia_da_semana` |

### Quando usar DERIVED em vez de CALC

Use **DERIVED** (persiste) quando:
- ✅ Lógica de derivação é complexa ou depende da fonte do dado
- ✅ Campo é frequentemente usado em filtros/queries (performance)
- ✅ Regra pode mudar com o tempo (precisa versionamento)

Use **CALC** (runtime) quando:
- ✅ Derivação trivial de 1 campo (`data.strftime('%b/%Y')`)
- ✅ Sempre dá o mesmo resultado independente da fonte

---

## 5. 🌐 Fontes de Dado (Multi-Source)

O sistema suporta **três fontes** de dados, distinguidas por uma coluna `fonte_dado` na tabela `ordens`:

```python
class FonteDado(str, Enum):
    NOTAS = 'notas'      # MVP — planilha de notas (avarias)
    ORDENS = 'ordens'    # Futuro — planilha com 'Status do sistema'
    API_SAP = 'api_sap'  # Longo prazo — integração direta
```

### Por que isso importa

- 🔄 Layout pode mudar sem migrar dados antigos
- 🔄 Lógica de derivação isolada por fonte (`if fonte == NOTAS: ...`)
- 🔄 Auditoria: sempre dá pra responder "de onde veio esse dado?"

### Limitação MVP

❗ **Misturar fontes não é recomendado.** Se o usuário trocar de layout sem reset, indicadores ficam inconsistentes. Existe endpoint admin de reset (`/admin/reset-dados`) — documentado em `docs/MANUAL_SUPORTE.md`.

---

## 6. 🛡️ Princípio de Robustez (Lei de Postel)

> *"Seja conservador no que você envia, e liberal no que você aceita."*

### Aplicação prática: 3 categorias de colunas no upload

| Categoria | O que fazer | Log? |
|---|---|---|
| ✅ **Mapeadas** (em `RENAME_*`) | Renomeia + mantém | Não |
| 🤷 **Ignoradas conhecidas** (em `COLUNAS_IGNORADAS_*`) | Descarta silenciosamente | Não |
| ⚠️ **Desconhecidas** (nada bate) | Descarta + log warning | **Sim** (alerta dev) |

### Por quê

- Se o SAP atualizar e adicionar coluna nova → sistema **não quebra**, só ignora e avisa
- Logs ajudam dev a perceber rapidamente que precisa atualizar `constants.py`
- Robustez > rigidez (planilhas SAP mudam com pouco aviso)

---

## 7. 📋 Constants — Fonte Única dos Nomes

### `backend/core/constants.py`

```python
"""
🚨 ÚNICA fonte da verdade dos nomes de colunas SAP.
Não duplique strings RAW em outros lugares — sempre importe daqui.
"""
from enum import Enum


# ============================================================
# FONTES DE DADO
# ============================================================
class FonteDado(str, Enum):
    NOTAS = 'notas'
    ORDENS = 'ordens'
    API_SAP = 'api_sap'


# ============================================================
# STATUS CANÔNICO (independe da fonte)
# ============================================================
class StatusCanonico(str, Enum):
    REALIZADA = 'REALIZADA'
    PENDENTE = 'PENDENTE'
    CANCELADA = 'CANCELADA'
    INDEFINIDO = 'INDEFINIDO'


# ============================================================
# CÓDIGOS SAP DE STATUS (usado quando fonte == ORDENS)
# ============================================================
STATUS_SAP_REALIZADA = {'ENC', 'TENC', 'ENCR', 'ENCT'}
STATUS_SAP_PENDENTE = {'ABG', 'LIB', 'CTAB', 'PCNF'}
STATUS_SAP_CANCELADA = {'CANC', 'STOR'}


# ============================================================
# COLUNAS DA PLANILHA DE NOTAS (MVP)
# ============================================================
class ColNotas:
    LINHA_SELECIONADA = 'Linha selecionada'
    DATA_NOTA = 'Data da nota'
    NOTA = 'Nota'
    ORDEM = 'Ordem'
    CRIADO_POR = 'Criado por'
    DESCRICAO = 'Descrição'
    LOCAL_INSTALACAO = 'Local de instalação'
    LOCAL_INSTALACAO_NOME = 'Denominação do loc.instalação'
    EQUIPAMENTO = 'Equipamento'
    OBJETO_TECNICO_NOME = 'Denominação do objeto técnico'
    CENTRO_TRABALHO = 'Centro trab.respons.'
    DURACAO_PARADA = 'Duração da parada'   # ⚠️ Não confiável no MVP
    EXECUTOR = 'Denominação executor (pess./usuár.)'
    GRP_PLNJ_PM = 'Grp.plnj.PM'
    INICIO_AVARIA = 'Início avaria'
    HORA_INICIO_AVARIA = 'Hora início avaria'
    FIM_AVARIA = 'Fim da avaria'
    HORA_FIM_AVARIA = 'Hora do fim avaria'
    INICIO_DESEJADO = 'Início desejado'
    ITEM_DOC_VENDAS = 'Item doc.vendas'
    ITEM_MANUTENCAO = 'Item manutenção'
    LOCALIZACAO = 'Localização'
    TEXTO_CODE_CODIFICACAO = 'Texto code para codificação'
    TEXT_PRIORIDADE = 'TextPrioridade'


# ============================================================
# COLUNAS DA PLANILHA DE ORDENS (futuro)
# ============================================================
class ColOrdens:
    TIPO_ORDEM = 'Tipo de ordem'
    DATA_BASE_INICIO = 'Data-base do início'
    NOTA = 'Nota'
    ORDEM = 'Ordem'
    CENTRO_TRABALHO = 'Centro trab.respons.'
    GRP_PLNJ_PM = 'Grp.plnj.PM'
    TEXTO_BREVE = 'Texto breve'
    STATUS_USUARIO = 'Status usuário'
    LOCAL_INSTALACAO = 'Local de instalação'
    LOCAL_INSTALACAO_NOME = 'Denominação do loc.instalação'
    EQUIPAMENTO = 'Equipamento'
    CUSTOS_ESTIMADOS = 'Custos estimados'
    CUSTOS_TOT_REAIS = 'Custos tot.reais'
    CUSTOS_TOTS_LIQUID = 'Custos tots.liquid.'
    CRIADO_POR = 'Criado por'
    ULTIMO_MODIFICADOR = 'Último modificador'
    STATUS_SISTEMA = 'Status do sistema'   # ⭐ Diferenciador realizada/pendente
    DATA_BASE_FIM = 'Data-base do fim'
    DATA_FIM_REAL_ORDEM = 'Data fim real da ordem'
    ELEMENTO_PEP = 'Elemento PEP'
    EXISTE_TXT_DESCR = 'Existe txt.descr.'
    PRIORIDADE = 'Prioridade'
    REVISAO = 'Revisão'


# ============================================================
# COLUNAS DA PLANILHA DE HORAS (apontamentos)
# ============================================================
class ColHoras:
    NUMERO_PESSOAL = 'Nº pessoal'
    ORDEM = 'Ordem'                        # 🔗 chave de junção com ordens
    TRABALHO_REAL = 'Trabalho real'        # horas decimais (ex: 2.5)
    DATA_INICIO_REAL = 'Data do início real'
    HORA_INICIO_REAL = 'Hora do início real'
    DATA_FIM_REAL = 'Data do fim real'
    HORA_FIM_REAL = 'Hora do fim real'


# ============================================================
# RENAME MAPS (RAW SAP → snake_case interno)
# ============================================================
RENAME_NOTAS = {
    ColNotas.NOTA: 'numero_nota',
    ColNotas.ORDEM: 'numero_ordem',
    ColNotas.DESCRICAO: 'descricao',
    ColNotas.CENTRO_TRABALHO: 'centro_trabalho',
    ColNotas.LOCAL_INSTALACAO: 'local_instalacao_codigo',
    ColNotas.LOCAL_INSTALACAO_NOME: 'local_instalacao_nome',
    ColNotas.EQUIPAMENTO: 'equipamento',
    ColNotas.OBJETO_TECNICO_NOME: 'objeto_tecnico_nome',
    ColNotas.GRP_PLNJ_PM: 'grupo_planejamento',
    ColNotas.CRIADO_POR: 'criado_por',
    ColNotas.DATA_NOTA: 'data_nota',
    ColNotas.DURACAO_PARADA: 'duracao_parada',  # ⚠️ não confiável no MVP
    ColNotas.EXECUTOR: 'executor',
    ColNotas.INICIO_DESEJADO: 'inicio_desejado',
    ColNotas.ITEM_DOC_VENDAS: 'item_doc_vendas',
    ColNotas.ITEM_MANUTENCAO: 'item_manutencao',
    ColNotas.LOCALIZACAO: 'localizacao_texto',
    ColNotas.TEXTO_CODE_CODIFICACAO: 'texto_code_codificacao',
    ColNotas.TEXT_PRIORIDADE: 'text_prioridade',
    # inicio_avaria/fim_avaria são MERGE de duas colunas — tratado em proc_notas.py
}

RENAME_ORDENS = {
    ColOrdens.ORDEM: 'numero_ordem',
    ColOrdens.NOTA: 'numero_nota',
    ColOrdens.TEXTO_BREVE: 'descricao',
    ColOrdens.CENTRO_TRABALHO: 'centro_trabalho',
    ColOrdens.LOCAL_INSTALACAO: 'local_instalacao_codigo',
    ColOrdens.LOCAL_INSTALACAO_NOME: 'local_instalacao_nome',
    ColOrdens.EQUIPAMENTO: 'equipamento',
    ColOrdens.GRP_PLNJ_PM: 'grupo_planejamento',
    ColOrdens.CRIADO_POR: 'criado_por',
    ColOrdens.TIPO_ORDEM: 'tipo_ordem_sap',
    ColOrdens.DATA_BASE_INICIO: 'data_base_inicio',
    ColOrdens.DATA_BASE_FIM: 'data_base_fim',
    ColOrdens.DATA_FIM_REAL_ORDEM: 'data_fim_real_ordem',
    ColOrdens.STATUS_SISTEMA: 'status_sistema',
    ColOrdens.STATUS_USUARIO: 'status_usuario',
    ColOrdens.CUSTOS_ESTIMADOS: 'custos_estimados',
    ColOrdens.CUSTOS_TOT_REAIS: 'custos_tot_reais',
    ColOrdens.CUSTOS_TOTS_LIQUID: 'custos_tots_liquid',
    ColOrdens.ELEMENTO_PEP: 'elemento_pep',
    ColOrdens.ULTIMO_MODIFICADOR: 'ultimo_modificador',
    ColOrdens.EXISTE_TXT_DESCR: 'existe_txt_descr',
    ColOrdens.PRIORIDADE: 'prioridade',
    ColOrdens.REVISAO: 'revisao',
}

RENAME_HORAS = {
    ColHoras.NUMERO_PESSOAL: 'numero_pessoal',
    ColHoras.ORDEM: 'numero_ordem',
    ColHoras.TRABALHO_REAL: 'trabalho_real',
    # data_inicio_real/data_fim_real são MERGE de duas colunas — tratado em proc_horas.py
}


# ============================================================
# COLUNAS A IGNORAR EXPLICITAMENTE
# ============================================================
COLUNAS_IGNORADAS_NOTAS: set[str] = {
    'Linha selecionada',
}
COLUNAS_IGNORADAS_ORDENS: set[str] = set()
COLUNAS_IGNORADAS_HORAS: set[str] = set()
```

---

## 8. 🔍 Classificação de Tipo de Ordem (Regex)

### Arquivo: `backend/core/classificacao.py`

```python
"""
Classifica ordens em categorias de domínio com base na DESCRIÇÃO (texto livre).
Funciona para qualquer fonte (notas, ordens, api) — depende só do texto.
"""
import re
from enum import Enum
import pandas as pd


class TipoOrdem(str, Enum):
    CORRETIVA = 'COR'
    PREVENTIVA = 'PREV'
    PREDITIVA = 'PRED'
    INSPECAO = 'INSP'
    CRITICA = 'CRI'
    INFRAESTRUTURA = 'INFRA'
    FABRICACAO = 'FAB'
    SEIS_S = '6S'
    INDEFINIDO = 'IND'


# Ordem importa: regras mais específicas vêm antes
REGRAS_CLASSIFICACAO: list[tuple[str, TipoOrdem]] = [
    (r'\b6\s?s\b|\bseis\s+s\b|seiri|seiton|seiso',  TipoOrdem.SEIS_S),
    (r'\bfabric|\bconf\b|\businag',                  TipoOrdem.FABRICACAO),
    (r'\binfraestrut|\bpredial|\bcivil',             TipoOrdem.INFRAESTRUTURA),
    (r'\bcrítica|\bcritica|\bemergência|\bemergencia', TipoOrdem.CRITICA),
    (r'\bprev|\bpmp\b|plano\s+anual',                TipoOrdem.PREVENTIVA),
    (r'\binspeç|\binspec|\bronda|check.?list',       TipoOrdem.INSPECAO),
    (r'\bpredit|análise\s+vibr|termograf',           TipoOrdem.PREDITIVA),
    (r'\bcorret|\breparo|\bconserto|\bsubstitu|\btroca', TipoOrdem.CORRETIVA),
]


def classificar(descricao: str) -> TipoOrdem:
    """
    Classifica uma única descrição. Retorna INDEFINIDO se nenhuma regra bater.

    >>> classificar("PREV-001 INSPEÇÃO MOTOR")
    <TipoOrdem.PREVENTIVA: 'PREV'>
    >>> classificar("CORRETIVA URGENTE BOMBA")
    <TipoOrdem.CORRETIVA: 'COR'>
    """
    if not isinstance(descricao, str) or not descricao.strip():
        return TipoOrdem.INDEFINIDO

    texto = descricao.lower()
    for padrao, tipo in REGRAS_CLASSIFICACAO:
        if re.search(padrao, texto, flags=re.IGNORECASE):
            return tipo
    return TipoOrdem.INDEFINIDO


def classificar_dataframe(df: pd.DataFrame, col: str = 'descricao') -> pd.Series:
    """Aplica classificação a uma coluna de DataFrame, retorna Series de TipoOrdem."""
    return df[col].apply(classificar)
```

### ⚠️ Regras anti-bug

- ❌ **NUNCA** use `\bprevent` — não pega `"PREV-001"`. Use `\bprev` (radical curto).
- ✅ **SEMPRE** teste a regex em `tests/test_classificacao.py` com casos reais SAP
- ✅ Adicione novos padrões **antes** de `CORRETIVA` (último fallback antes de IND)

---

## 9. 🕐 Enquadramento de Turno

### Arquivo: `backend/core/turnos.py`

```python
"""
Converte datetime → Turno (A/B/C/ADM).
Regras configuráveis via REGRAS_HORARIO (futuro: vir do banco).
"""
from datetime import time, datetime
from enum import Enum
import pandas as pd


class Turno(str, Enum):
    A = 'A'      # 06:00 - 14:00
    B = 'B'      # 14:00 - 22:00
    C = 'C'      # 22:00 - 06:00
    ADM = 'ADM'  # horário administrativo (segunda a sexta, 08-17)
    INDEFINIDO = 'INDEFINIDO'


REGRAS_HORARIO = {
    Turno.A: (time(6, 0), time(14, 0)),
    Turno.B: (time(14, 0), time(22, 0)),
    Turno.C: (time(22, 0), time(6, 0)),  # ⚠️ vira virada de dia
}


def enquadrar_turno(dt) -> Turno:
    """
    Recebe datetime/Timestamp e retorna Turno.
    NaN/None → INDEFINIDO.
    """
    if pd.isna(dt) or dt is None:
        return Turno.INDEFINIDO

    if not isinstance(dt, (datetime, pd.Timestamp)):
        return Turno.INDEFINIDO

    hora = dt.time()
    inicio_a, fim_a = REGRAS_HORARIO[Turno.A]
    inicio_b, fim_b = REGRAS_HORARIO[Turno.B]

    if inicio_a <= hora < fim_a:
        return Turno.A
    if inicio_b <= hora < fim_b:
        return Turno.B
    return Turno.C  # qualquer outro horário cai no C (vira dia)


def aplicar_turnos_dataframe(df: pd.DataFrame, col: str) -> pd.Series:
    return df[col].apply(enquadrar_turno)
```

---

## 10. ⚙️ Derivadores

### Arquivo: `backend/core/derivadores.py`

```python
"""
DERIVADORES: convertem dados RAW de diferentes fontes em campos canônicos.

🧠 IMPORTANTE: quando uma regra de derivação muda, incrementar VERSAO_DERIVADOR
e disponibilizar reprocessamento via /admin/reprocessar-derivacoes.
"""
import pandas as pd
from backend.core.constants import (
    FonteDado, StatusCanonico,
    STATUS_SAP_REALIZADA, STATUS_SAP_PENDENTE, STATUS_SAP_CANCELADA,
)
from backend.core.classificacao import classificar
from backend.core.turnos import enquadrar_turno

VERSAO_DERIVADOR = 'v1'


def derivar_status(linha: dict, fonte: FonteDado) -> StatusCanonico:
    """
    Regra MVP (NOTAS):
    - 'fim_avaria' preenchido → REALIZADA
    - vazio → PENDENTE

    Regra futura (ORDENS):
    - 'status_sistema' contém ENC/TENC → REALIZADA
    - contém CANC/STOR → CANCELADA
    - contém ABG/LIB/CTAB → PENDENTE
    - senão → INDEFINIDO
    """
    if fonte == FonteDado.NOTAS:
        fim_avaria = linha.get('fim_avaria')
        if pd.notna(fim_avaria):
            return StatusCanonico.REALIZADA
        return StatusCanonico.PENDENTE

    elif fonte == FonteDado.ORDENS:
        status_raw = linha.get('status_sistema', '')
        if not isinstance(status_raw, str):
            if pd.notna(linha.get('data_fim_real_ordem')):
                return StatusCanonico.REALIZADA
            return StatusCanonico.INDEFINIDO

        codigos = set(status_raw.upper().split())
        if codigos & STATUS_SAP_REALIZADA:
            return StatusCanonico.REALIZADA
        if codigos & STATUS_SAP_CANCELADA:
            return StatusCanonico.CANCELADA
        if codigos & STATUS_SAP_PENDENTE:
            return StatusCanonico.PENDENTE
        return StatusCanonico.INDEFINIDO

    return StatusCanonico.INDEFINIDO


def derivar_data_inicio_canonica(linha: dict, fonte: FonteDado):
    """Unifica data início vinda de fontes diferentes."""
    if fonte == FonteDado.NOTAS:
        return linha.get('inicio_avaria')
    elif fonte == FonteDado.ORDENS:
        return linha.get('data_base_inicio')  # ⚠️ planejada, não real
    return None


def derivar_data_fim_canonica(linha: dict, fonte: FonteDado):
    """Unifica data fim."""
    if fonte == FonteDado.NOTAS:
        return linha.get('fim_avaria')
    elif fonte == FonteDado.ORDENS:
        return linha.get('data_fim_real_ordem')
    return None


def aplicar_derivacao_dataframe(df: pd.DataFrame, fonte: FonteDado) -> pd.DataFrame:
    """
    Pipeline completo de derivação. Adiciona colunas:
    - status_canonico, classificacao, turno
    - data_inicio_canonica, data_fim_canonica
    - fonte_dado, versao_derivador
    """
    df = df.copy()

    df['status_canonico'] = df.apply(
        lambda r: derivar_status(r.to_dict(), fonte).value, axis=1,
    )
    df['classificacao'] = df['descricao'].apply(lambda d: classificar(d).value)
    df['data_inicio_canonica'] = df.apply(
        lambda r: derivar_data_inicio_canonica(r.to_dict(), fonte), axis=1,
    )
    df['data_fim_canonica'] = df.apply(
        lambda r: derivar_data_fim_canonica(r.to_dict(), fonte), axis=1,
    )
    df['turno'] = df['data_inicio_canonica'].apply(
        lambda d: enquadrar_turno(d).value
    )
    df['fonte_dado'] = fonte.value
    df['versao_derivador'] = VERSAO_DERIVADOR

    return df
```

---

## 11. 📋 Pipelines de Processamento

### Estrutura padrão

Todo `proc_*.py` segue o mesmo pipeline:

```
1. Recebe df_raw (planilha bruta)
2. Drop de COLUNAS_IGNORADAS_*
3. Detecta colunas DESCONHECIDAS (warning)
4. Concatena data + hora (quando aplicável)
5. Renomeia para snake_case via RENAME_*
6. Mantém apenas colunas mapeadas
7. Tipagem (datetime, numeric, string)
8. Validações de domínio
9. Aplica derivadores (se for tabela ordens)
10. Retorna DataFrame limpo
```

### `backend/core/proc_notas.py`

```python
"""Pipeline da planilha de NOTAS (MVP)."""
import pandas as pd
import logging
from backend.core.constants import (
    ColNotas, RENAME_NOTAS, COLUNAS_IGNORADAS_NOTAS, FonteDado,
)
from backend.core.derivadores import aplicar_derivacao_dataframe

logger = logging.getLogger(__name__)


def processar_notas(df_raw: pd.DataFrame) -> pd.DataFrame:
    df = df_raw.copy()

    # 1. Drop de colunas ignoradas
    df = df.drop(
        columns=[c for c in COLUNAS_IGNORADAS_NOTAS if c in df.columns],
        errors='ignore',
    )

    # 2. Detecta desconhecidas
    colunas_conhecidas = (
        set(RENAME_NOTAS.keys())
        | COLUNAS_IGNORADAS_NOTAS
        | {ColNotas.INICIO_AVARIA, ColNotas.HORA_INICIO_AVARIA,
           ColNotas.FIM_AVARIA, ColNotas.HORA_FIM_AVARIA}
    )
    desconhecidas = set(df_raw.columns) - colunas_conhecidas
    if desconhecidas:
        logger.warning(f"⚠️ Colunas desconhecidas em NOTAS: {desconhecidas}")

    # 3. Concatena data+hora ANTES de renomear (usa nomes RAW)
    df = _concatenar_data_hora(
        df, ColNotas.INICIO_AVARIA, ColNotas.HORA_INICIO_AVARIA, 'inicio_avaria',
    )
    df = _concatenar_data_hora(
        df, ColNotas.FIM_AVARIA, ColNotas.HORA_FIM_AVARIA, 'fim_avaria',
    )

    # 4. Renomeia
    df = df.rename(columns=RENAME_NOTAS)

    # 5. Mantém apenas mapeadas + concatenadas
    colunas_finais = list(RENAME_NOTAS.values()) + ['inicio_avaria', 'fim_avaria']
    df = df[[c for c in colunas_finais if c in df.columns]]

    # 6. Aplica derivadores
    df = aplicar_derivacao_dataframe(df, FonteDado.NOTAS)

    return df


def _concatenar_data_hora(df, col_data, col_hora, nome_saida):
    """Combina coluna de data + coluna de hora em datetime único."""
    def _combinar(row):
        data = row.get(col_data)
        hora = row.get(col_hora)
        if pd.isna(data):
            return pd.NaT
        if pd.isna(hora):
            return pd.Timestamp(data)
        try:
            return pd.Timestamp.combine(
                pd.Timestamp(data).date(),
                pd.Timestamp(hora).time(),
            )
        except (ValueError, TypeError):
            return pd.NaT

    df[nome_saida] = df.apply(_combinar, axis=1)
    return df
```

### `backend/core/proc_horas.py`

```python
"""
Pipeline da planilha de HORAS (apontamentos).

⚠️ IMPORTANTE:
- Horas só são apontadas em notas que têm ordem associada
- 1 ordem pode ter N apontamentos (vários operadores, vários dias)
- Junção com tabela 'ordens' é via numero_ordem (FK lógica)
"""
import pandas as pd
import logging
from backend.core.constants import (
    ColHoras, RENAME_HORAS, COLUNAS_IGNORADAS_HORAS,
)
from backend.core.turnos import enquadrar_turno
from backend.core.derivadores import VERSAO_DERIVADOR

logger = logging.getLogger(__name__)


def processar_horas(df_raw: pd.DataFrame) -> pd.DataFrame:
    df = df_raw.copy()

    # 1. Drop ignoradas
    df = df.drop(
        columns=[c for c in COLUNAS_IGNORADAS_HORAS if c in df.columns],
        errors='ignore',
    )

    # 2. Detecta desconhecidas
    colunas_conhecidas = (
        set(RENAME_HORAS.keys())
        | COLUNAS_IGNORADAS_HORAS
        | {ColHoras.DATA_INICIO_REAL, ColHoras.HORA_INICIO_REAL,
           ColHoras.DATA_FIM_REAL, ColHoras.HORA_FIM_REAL}
    )
    desconhecidas = set(df_raw.columns) - colunas_conhecidas
    if desconhecidas:
        logger.warning(f"⚠️ Colunas desconhecidas em HORAS: {desconhecidas}")

    # 3. Concatena data+hora
    df = _concatenar_data_hora(
        df, ColHoras.DATA_INICIO_REAL, ColHoras.HORA_INICIO_REAL, 'data_inicio_real',
    )
    df = _concatenar_data_hora(
        df, ColHoras.DATA_FIM_REAL, ColHoras.HORA_FIM_REAL, 'data_fim_real',
    )

    # 4. Renomeia
    df = df.rename(columns=RENAME_HORAS)

    # 5. Mantém apenas úteis
    colunas_finais = list(RENAME_HORAS.values()) + ['data_inicio_real', 'data_fim_real']
    df = df[[c for c in colunas_finais if c in df.columns]]

    # 6. Validações
    df = _validar_horas(df)

    # 7. Derivadores específicos
    df['turno'] = df['data_inicio_real'].apply(
        lambda d: enquadrar_turno(d).value if pd.notna(d) else None
    )
    df['duracao_calculada'] = (
        (df['data_fim_real'] - df['data_inicio_real']).dt.total_seconds() / 3600
    )
    df['versao_derivador'] = VERSAO_DERIVADOR

    return df


def _concatenar_data_hora(df, col_data, col_hora, nome_saida):
    def _combinar(row):
        data = row.get(col_data)
        hora = row.get(col_hora)
        if pd.isna(data):
            return pd.NaT
        if pd.isna(hora):
            return pd.Timestamp(data)
        try:
            return pd.Timestamp.combine(
                pd.Timestamp(data).date(),
                pd.Timestamp(hora).time(),
            )
        except (ValueError, TypeError):
            return pd.NaT

    df[nome_saida] = df.apply(_combinar, axis=1)
    return df


def _validar_horas(df: pd.DataFrame) -> pd.DataFrame:
    """Validações de consistência."""
    sem_ordem = df['numero_ordem'].isna().sum()
    if sem_ordem > 0:
        logger.warning(f"⚠️ {sem_ordem} apontamentos sem numero_ordem")

    mask_neg = df['trabalho_real'] < 0
    if mask_neg.any():
        logger.error(f"❌ {mask_neg.sum()} apontamentos com trabalho_real negativo (removidos)")
        df = df[~mask_neg]

    mask_inv = df['data_fim_real'] < df['data_inicio_real']
    if mask_inv.any():
        logger.warning(f"⚠️ {mask_inv.sum()} apontamentos com fim
