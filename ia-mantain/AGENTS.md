# 🏗️ IA-MANTAIN — Sistema Inteligente de Gestão de Manutenção SAP

> **Versão:** 1.0
> **Última atualização:** 2026-04-25
> **Mantenedor:** Filipe

---

## 1. 🎯 O QUE ESTE PROJETO É

Sistema de **Business Intelligence + IA conversacional** para análise de Ordens de Manutenção do SAP, focado em uma **Mineradora**. Migra um dashboard Streamlit monolítico para uma arquitetura **frontend/backend desacoplada**, com:

- 📊 **Cockpit visual** com gráficos interativos (drill-down em cascata)
- 🤖 **Analista IA** que conversa em linguagem natural sobre os dados
- 📤 **Upload e gestão** de planilhas SAP via interface
- 👥 **RBAC com 4 papéis** (Visitante / Coordenador / Assistente / Supervisor)

### Predecessor

Este projeto **substitui** o `dashboard-manutencao` (Streamlit + Pandas + LangChain). Toda lição aprendida no 1 mês e 15 dias de desenvolvimento do projeto anterior está documentada em `docs/HISTORICAL_BUGS.md` e nas regras deste arquivo.

---

## 2. 🛠️ STACK TECNOLÓGICA

### Frontend

- **Framework:** Next.js 14+ (App Router)
- **UI:** shadcn/ui + TailwindCSS
- **Gráficos:** Recharts (cockpit) + Apache ECharts (analista IA)
- **Estado:** Zustand + TanStack Query
- **Auth:** Supabase Auth client (`@supabase/ssr`)
- **Formulários:** React Hook Form + Zod

### Backend

- **Framework:** FastAPI (Python 3.11+)
- **ORM:** SQLAlchemy 2.0 (async) + Alembic
- **Validação:** Pydantic v2
- **IA:** LangChain + Google Gemini (`create_sql_agent`, **NÃO** `pandas_dataframe_agent`)
- **Auth:** Supabase Auth (validação JWT via middleware)

### Banco de Dados

- **Provider:** Supabase (PostgreSQL gerenciado)
- **Auth:** Supabase Auth nativo (email + senha + JWT)
- **Schema:** Relacional com 5 tabelas principais (ver `docs/DATA_CONTRACT.md`)
- **RLS:** Habilitado em todas as tabelas

### Deploy

- **Frontend:** Vercel
- **Backend:** Railway / Render / Fly.io (a definir)
- **Banco:** Supabase Cloud

---

## 3. 🗂️ ESTRUTURA DO PROJETO

```
ia-mantain/
├── AGENTS.md                    ← VOCÊ ESTÁ AQUI (regras globais)
├── frontend/                    ← Next.js (ver frontend/AGENTS.md)
│   ├── app/
│   │   ├── (auth)/              ← Login, signup
│   │   ├── cockpit/             ← Dashboards visuais
│   │   ├── analista/            ← Chat com IA
│   │   ├── dados/               ← Upload de planilhas
│   │   └── admin/               ← Gestão de usuários
│   ├── components/
│   ├── lib/
│   └── hooks/
│
├── backend/                     ← FastAPI (ver backend/AGENTS.md)
│   ├── api/                     ← Routers REST
│   ├── core/                    ← Pipeline ETL (anticorrupção SAP)
│   ├── ia/                      ← SQL Agent + geração de ECharts
│   ├── db/                      ← Schemas + migrations
│   └── schemas/                 ← Modelos Pydantic
│
└── docs/
    ├── PRD.md                   ← Visão de produto completa
    ├── DATA_CONTRACT.md         ← Schema do banco + colunas RAW/CALC
    ├── BUSINESS_RULES.md        ← REGRAS_HORARIO, classificação ordens
    ├── HISTORICAL_BUGS.md       ← Lições aprendidas (LEIA SEMPRE!)
    └── CHANGELOG.md
```

> 💡 **Cada subpasta tem seu próprio `AGENTS.md`** com regras específicas. Ele é carregado automaticamente quando você lê/edita arquivos daquela pasta.

---

## 4. ⛔ REGRAS CRÍTICAS GLOBAIS (NUNCA VIOLAR)

Estas regras são **inegociáveis**. Foram aprendidas com bugs reais do projeto anterior.

### 4.1 IA SQL Agent (não Pandas)

> ❌ **NUNCA** use `create_pandas_dataframe_agent`
> ✅ **SEMPRE** use `create_sql_agent` com a conexão Supabase

**Por quê?** Pandas Agent passa o DataFrame inteiro como string no prompt → estoura tokens e custa caro. SQL Agent gera SQL dinâmico que executa no Postgres → eficiente e escalável.

### 4.2 Datas: Padrão Dual-Format (BUG histórico de 56% NaT)

> ❌ **NUNCA** use `pd.to_datetime(df[col], dayfirst=True)` sozinho
> ✅ **SEMPRE** tente ISO8601 primeiro, depois fallback BR

```python
# ✅ CORRETO — Padrão Dual-Format
datas = pd.to_datetime(df[col], format='ISO8601', errors='coerce')
mask_nat = datas.isna() & df[col].notna()
datas[mask_nat] = pd.to_datetime(df.loc[mask_nat, col], dayfirst=True, errors='coerce')
df[col] = datas
```

**Por quê?** Dados podem vir do banco (ISO `2026-01-05T00:00:00`) ou do Excel BR (`05/01/2026`). `dayfirst=True` sozinho falha com ISO.

### 4.3 Colunas RAW vs CALC

> ❌ **NUNCA** persista colunas calculadas (CALC) no banco
> ✅ Colunas CALC são **sempre regeneradas** ao carregar do banco

**Lista CALC** (gerada no backend, nunca salva): `Data_Calc`, `Semana_Trabalho`, `Dia`, `Semana`, `Mês`, `Classificacao_Ordem`, `Status_SAP_Normalizado`, `Plan_Sn`, `Aprop_Sn`, `Mes_Ano`, `Nome_Exibicao`.

Ver lista completa em `docs/DATA_CONTRACT.md`.

### 4.4 Constantes Canônicas (Camada Anticorrupção SAP)

> ❌ **NUNCA** hardcode nomes de colunas SAP em queries/componentes
> ✅ **SEMPRE** importe das constantes em `backend/core/constants.py`

```python
# ✅ CORRETO
from backend.core.constants import COL_LOCAL_INSTALACAO, COL_CENTRO_TRABALHO, COL_DESCRICAO

# ❌ ERRADO
df[df['Centro trab.respons.'] == 'SL-MEC']           # hardcoded com acento
df['Denominação do loc.instalação']                  # se SAP mudar, quebra tudo
```

**Por quê?** Quando o SAP mudar nomes de colunas (e ele vai), só um arquivo precisa ser atualizado.

### 4.5 RBAC — 4 Papéis

| Papel | Pode fazer |
|---|---|
| **Visitante** (não logado) | ✅ Ver cockpit (gráficos) |
| **Coordenador** | ✅ Tudo de visitante + ver detalhes |
| **Assistente** | ✅ Tudo de coordenador + upload + editar efetivo + chat IA |
| **Supervisor** | ✅ Tudo de assistente + gerenciar usuários + limpar banco |

Validação em **DOIS níveis** (defesa em profundidade):

1. **UI** — Componente só renderiza se `papel` autoriza
2. **Backend** — Middleware valida JWT + papel em **toda** rota protegida

### 4.6 Drill-down em Cascata (BUG histórico de `removeChild`)

> ❌ **NUNCA** use re-render imperativo dentro de captura de clique em gráfico
> ✅ **SEMPRE** renderize níveis aninhados de forma declarativa (React state)

**Por quê?** No Streamlit isso causava `removeChild` no DOM. No React/Next.js o padrão é declarativo, mas a **filosofia** se mantém: estado de drill-down é aditivo, nunca destrutivo.

### 4.7 Outer Join no `df_sap_completo` (BUG-003)

> ✅ Usar `how='outer'` ao unificar horas + ordens
> ❌ Não usar `how='left'` (perde ordens sem apontamento)

Ver detalhes em `docs/HISTORICAL_BUGS.md` (BUG-003).

---

## 5. 🎨 PADRÕES DE CÓDIGO

### 5.1 Idioma

- **Código:** Inglês (variáveis, funções, classes)
- **Comentários:** Português (BR)
- **UI/Mensagens:** Português (BR)
- **Documentação interna:** Português (BR)
- **Commits:** Português (BR), padrão Conventional Commits

```typescript
// ✅ CORRETO
function calcularTotalHoras(apontamentos: Apontamento[]): number {
  // Soma todas as horas reais apontadas no mês
  return apontamentos.reduce((acc, a) => acc + a.trabalhoReal, 0)
}

// ❌ ERRADO
function calculate_total_horas(apontamentos) {  // mistura snake_case com PT
  // Sums all real hours
  return apontamentos.reduce((acc, a) => acc + a.trabalho_real, 0)
}
```

### 5.2 Naming

| Tipo | Padrão | Exemplo |
|---|---|---|
| Variáveis JS/TS | camelCase | `dataInicial`, `totalHoras` |
| Variáveis Python | snake_case | `data_inicial`, `total_horas` |
| Componentes React | PascalCase | `GraficoCascata`, `BadActorsTable` |
| Tipos/Interfaces | PascalCase | `Apontamento`, `OrdemSAP` |
| Constantes | UPPER_SNAKE | `COL_LOCAL_INSTALACAO`, `REGRAS_HORARIO` |
| Arquivos de componente | kebab-case | `grafico-cascata.tsx` |
| Arquivos Python | snake_case | `proc_horas.py` |

### 5.3 Tamanho de Arquivo

Limites por tipo de arquivo (filosofia: arquivos pequenos = mais fáceis de manter):

| Tipo | Soft limit ⚠️ | Hard limit 🛑 |
|---|---|---|
| Componentes React (`.tsx`) | 150 | 250 |
| Hooks customizados (`.ts`) | 100 | 150 |
| Utils / helpers | 200 | 350 |
| Routers FastAPI | 150 | 250 |
| Módulos de processamento (ETL) | 250 | 400 |
| Schemas (Pydantic / Zod) | 200 | 300 |
| Testes | 400 | 600 |

**Quando atingir o soft limit:** considere refatorar.
**Quando atingir o hard limit:** refatore antes de adicionar mais código.

> 💡 Lição do projeto anterior: `processamento.py` chegou a ~450 linhas misturando ETL de horas, ordens, classificação e helpers. Virou DT-10 (dívida de refatoração). Quebrar em módulos menores desde o início evita isso.

**Princípio geral:** Um arquivo deve ter **uma responsabilidade clara**. Se você não consegue descrever o que ele faz em uma frase, ele tá grande demais.

### 5.4 Comentários

```python
# ✅ Use comentários para explicar O PORQUÊ, não O QUE
# Calcula plan_mensal somando S1-S4 (S5 é parcial e tratada separadamente)
plan_mes = sum(plan_semanal[:4])

# ❌ Comentários óbvios são ruído
# Soma os valores
total = sum(valores)
```

### 5.5 Tipagem

- **Python:** Type hints obrigatórios em **todas** as funções públicas
- **TypeScript:** Strict mode habilitado, **proibido** `any` (use `unknown` se necessário)

---

## 6. 🔒 SEGURANÇA — PRINCÍPIOS UNIVERSAIS

> 🚨 **Este sistema gerencia dados operacionais críticos de uma mineradora.**
> Vazamento, adulteração ou indisponibilidade têm impacto real em segurança operacional, financeiro e legal (LGPD).

### 6.1 Os 7 Princípios Inegociáveis

#### 1️⃣ **Defesa em Profundidade**

Nunca confie em uma única camada. Se a validação do frontend falhar, a do backend pega. Se o RBAC falhar, o RLS do banco pega. Se o RLS falhar, os logs de auditoria detectam.

#### 2️⃣ **Princípio do Menor Privilégio**

Cada usuário, serviço, função e token tem **apenas** as permissões mínimas que precisa. Nunca dê acesso "só pra facilitar".

#### 3️⃣ **Zero Trust em Inputs**

**TODO** dado externo (formulário, upload, query string, header, JWT, resposta de API externa, **prompt da IA**) é **suspeito** até ser validado.

#### 4️⃣ **Falhe Seguro, Não Aberto**

Se algo der errado (exceção, timeout, valor nulo), o comportamento default é **negar acesso**, não permitir. Erro = 403, não 200 com dados parciais.

#### 5️⃣ **Segredos Nunca no Código**

API keys, tokens, senhas, JWT secrets → **sempre** em variáveis de ambiente. **Nunca** em código, comentários, logs ou commits.

#### 6️⃣ **Auditabilidade Total**

Toda ação sensível (login, upload, exclusão, mudança de papel, query da IA) é **logada** com: usuário, timestamp, IP, ação, resultado.

#### 7️⃣ **Atualizações de Dependências**

Dependências desatualizadas = vetor de ataque conhecido. `npm audit` e `pip-audit` são parte do CI.

### 6.2 Checklist Mínimo Antes de Cada PR/Commit

A IA **DEVE** verificar mentalmente antes de gerar código:

- [ ] 🔐 Algum segredo ou credencial hardcoded? → **NÃO**
- [ ] 🛡️ Input do usuário é validado (Pydantic/Zod)? → **SIM**
- [ ] 🚫 SQL é parametrizado (não string concatenada)? → **SIM**
- [ ] 🔑 Endpoint sensível tem `Depends(requer_papel(...))`? → **SIM**
- [ ] 📝 Erro expõe stack trace ao usuário? → **NÃO**
- [ ] 🌐 CORS está configurado pra origem específica (não `*`)? → **SIM**
- [ ] 📦 Dependência nova foi auditada? → **SIM**
- [ ] 📊 Ação sensível é logada? → **SIM**

### 6.3 Os "Nunca" do Sistema

| ❌ NUNCA | ✅ SEMPRE |
|---|---|
| Confiar em validação só do frontend | Validar também no backend |
| `eval()`, `exec()`, `os.system()` | Funções nativas tipadas |
| String concatenada em SQL | ORM ou query parametrizada |
| Logar senha, token, JWT, API key | Mascarar (`****`) ou omitir |
| `allow_origins=["*"]` em produção | Lista explícita de origens |
| `dangerouslySetInnerHTML` sem sanitização | DOMPurify ou evitar |
| Aceitar arquivo sem validar tipo/tamanho | Whitelist + limite + scan |
| Confiar em `user_id` do request body | Extrair sempre do JWT validado |
| Expor mensagens de erro internas | Mensagens genéricas + log interno |
| Commitar `.env` | `.env` no `.gitignore`, sempre |

### 6.4 Para a IA Geradora de Código

Quando gerar código, a IA **DEVE**:

1. **Assumir hostilidade dos inputs** — todo dado externo é potencialmente malicioso
2. **Preferir bibliotecas auditadas** a "soluções caseiras" pra crypto/auth
3. **Comentar decisões de segurança** explicitamente:

   ```python
   # 🔒 SEGURANÇA: validamos JWT antes de extrair user_id
   # Nunca confiar em user_id vindo do body do request.
   ```

4. **Não simplificar** validações pedindo permissão — segurança não é opcional
5. **Avisar explicitamente** quando código gerado tem implicação de segurança:
   > ⚠️ **Atenção de segurança:** este endpoint expõe dados de outros usuários. Confirme que o RBAC está correto antes de fazer deploy.

### 6.5 Detalhes Técnicos

> 📖 Princípios universais vivem aqui. Implementações técnicas detalhadas (validação de JWT, sanitização de upload, hardening de IA, etc.) ficam em **`docs/SECURITY.md`**.

## 7. 🚦 FLUXO DE TRABALHO COM IA (Antigravity / Cursor)

### Antes de gerar código, a IA DEVE

1. ✅ Ler o `AGENTS.md` da pasta onde vai criar/editar
2. ✅ Verificar se a tarefa tem precedente em `docs/HISTORICAL_BUGS.md`
3. ✅ Validar nomes contra `docs/DATA_CONTRACT.md` (se mexer com schema)
4. ✅ Confirmar que segue as **Regras Críticas Globais** (Seção 4)

### Antes de fazer mudanças destrutivas, a IA DEVE

- 🛑 Pedir confirmação ao usuário
- 🛑 Listar todos os arquivos que serão alterados
- 🛑 Explicar o impacto no resto do sistema

### Em caso de dúvida

> 📌 **Pergunte antes de inventar.** É melhor uma pergunta extra do que código que precisa ser refeito.

---

## 8. 🚫 ANTI-PATTERNS (não fazer)

| ❌ Não faça | ✅ Faça |
|---|---|
| `pd.read_excel()` direto em rota da API | Use `core/proc_horas.py` (camada centralizada) |
| `select * from ordens_sap` em endpoint | Use Pydantic schemas e SQLAlchemy ORM |
| Lógica de negócio em componente React | Mover pra `lib/` ou hook customizado |
| Componente acessando Supabase direto | Sempre via API do backend (frontend só consome REST) |
| `useState` para dados servidos | Use TanStack Query |
| CSS inline ou `style={{}}` | Sempre TailwindCSS |
| Validação só no frontend | **SEMPRE** validar no backend também |
| Senha em `.env` versionado | `.env` no `.gitignore`, exemplo em `.env.example` |
| Commit com `WIP` ou mensagem genérica | Conventional Commits (`feat:`, `fix:`, `docs:`) |

---

## 9. 📚 DOCUMENTOS IMPORTANTES

| Quando precisar de... | Leia |
|---|---|
| Visão geral do produto | `docs/PRD.md` |
| Schema do banco / colunas | `docs/DATA_CONTRACT.md` |
| Regras de negócio (turnos, classificação) | `docs/BUSINESS_RULES.md` |
| Bugs históricos / dívidas técnicas | `docs/HISTORICAL_BUGS.md` |
| Histórico de mudanças | `docs/CHANGELOG.md` |

---

## 10. 🎯 PRINCÍPIOS NORTEADORES

1. **DRY (Don't Repeat Yourself)** — Lógica duplicada é convite pra bug
2. **Fonte Única de Verdade** — Cada regra mora em UM lugar
3. **Separação de Responsabilidades** — Frontend renderiza, backend processa, banco persiste
4. **Falhe Rápido** — Erros na entrada, não no meio do pipeline
5. **Documentação Viva** — Código sem doc é código órfão
6. **Pragmatismo > Perfeccionismo** — Melhor entregar e iterar do que travar buscando o ideal

---

## 11. 🆘 EM CASO DE DÚVIDA

- **Sobre arquitetura?** → Leia o `AGENTS.md` da pasta + `docs/PRD.md`
- **Sobre dados?** → `docs/DATA_CONTRACT.md`
- **Sobre regras de negócio?** → `docs/BUSINESS_RULES.md`
- **"Já fizemos isso antes?"** → `docs/HISTORICAL_BUGS.md`
- **Nada do acima resolve?** → Pergunte ao Filipe

---

> 🤝 **Este arquivo é vivo.** Quando uma regra nova for descoberta (especialmente a partir de um bug), ela deve ser adicionada aqui ou em `docs/HISTORICAL_BUGS.md`. Conhecimento que não está documentado, não existe.
