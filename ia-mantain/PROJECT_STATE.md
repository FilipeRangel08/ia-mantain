# 📊 PROJECT_STATE — IA-MANTAIN

> **Função:** Snapshot de progresso. Cole isto no início de qualquer conversa nova com IA pra retomar contexto.
> **Última atualização:** 2026-04-25

---

## 🎯 IDENTIDADE DO PROJETO

- **Nome:** ia-mantain
- **Domínio:** Mineradora ⛏️ (NÃO é refinaria de petróleo)
- **Localização:** `C:\Users\filip\Desktop\github\sistema-manutencao\ia-mantain`
- **Predecessor:** dashboard-manutencao (Streamlit, ~1 mês e 15 dias de desenvolvimento)
- **Mantenedor:** Filipe (Santaluz, BA, Brasil)

---

## 🛠️ STACK CONFIRMADA

### Frontend

- Next.js 14+ (App Router)
- TypeScript (strict)
- TailwindCSS + shadcn/ui
- Recharts (gráficos do cockpit) + Apache ECharts (gráficos da IA)
- TanStack Query (estado servido) + Zustand (estado de UI global)
- React Hook Form + Zod (formulários)
- Supabase Auth client (`@supabase/ssr`)

### Backend

- FastAPI (Python 3.11+)
- SQLAlchemy 2.0 async + Alembic
- Pydantic v2
- LangChain + Google Gemini (`create_sql_agent`, NÃO `pandas_dataframe_agent`)

### Banco

- Supabase (PostgreSQL + Auth + RLS)

### Deploy (a definir)

- Frontend: Vercel
- Backend: Railway/Render/Fly.io
- Banco: Supabase Cloud

---

## 👥 RBAC — 4 PAPÉIS

1. **Visitante** (não logado): só vê cockpit
2. **Coordenador**: + ver detalhes
3. **Assistente**: + upload + chat IA + editar efetivo
4. **Supervisor**: + gerenciar usuários + limpar banco

---

## ⛔ REGRAS CRÍTICAS (NUNCA VIOLAR)

1. **IA = SQL Agent**, NÃO Pandas Agent
2. **Datas = Padrão Dual-Format** (ISO8601 → fallback BR `dayfirst=True`)
3. **Colunas RAW vs CALC** — CALC nunca persiste no banco
4. **Constantes canônicas** — nunca hardcode nomes de colunas SAP
5. **Validação dupla RBAC** — UI + Backend
6. **Drill-down declarativo** — nunca manipular DOM imperativo
7. **Outer Join no df_sap_completo** — nunca `how='left'`

---

## 📏 LIMITES DE ARQUIVO

| Tipo | Soft ⚠️ | Hard 🛑 |
|---|---|---|
| Componentes React (.tsx) | 150 | 250 |
| Hooks (.ts) | 100 | 150 |
| Utils/helpers | 200 | 350 |
| Routers FastAPI | 150 | 250 |
| ETL Python | 250 | 400 |
| Schemas (Pydantic/Zod) | 200 | 300 |
| Testes | 400 | 600 |

---

## 🎨 DECISÕES VISUAIS

- **Dark mode:** obrigatório (turno noturno)
- **Mobile:** desktop first, cockpit responsivo até 768px
- **Cores:** paleta HSL inicial em `lib/constants.ts` (Filipe vai revisar)

---

## 📝 PADRÕES DE CÓDIGO

- Variáveis JS/TS: camelCase | Python: snake_case
- Componentes React: PascalCase
- Arquivos componente: kebab-case (`grafico-cascata.tsx`)
- Arquivos Python: snake_case (`proc_horas.py`)
- Constantes: UPPER_SNAKE_CASE
- Comentários e UI: Português (BR)
- Código (variáveis/funções): Inglês

---

## ✅ ARQUIVOS JÁ CRIADOS

### Estrutura

- [x] Pastas do projeto (frontend, backend, docs + subpastas)
- [x] Arquivos vazios (AGENTS.md hierárquicos + docs)

### Conteúdo escrito

- [x] `AGENTS.md` (raiz) + Seção 6 de Segurança
- [x] `frontend/AGENTS.md`
- [x] `backend/AGENTS.md`
- [x] `docs/SECURITY.md` ← NOVO

### ⏳ PRÓXIMO ARQUIVO A CRIAR

- [ ] `backend/core/AGENTS.md` ← AQUI PARAMOS

### 🔒 Decisões de segurança registradas

- httpOnly cookies para JWT
- Usuário ia_readonly para LangChain
- Validação MIME via python-magic
- System prompt blindado contra injection
- Rate limit IA: 30/hora/usuário

---

## 📋 ROADMAP COMPLETO

### Fase 1: AGENTS.md hierárquicos

- [x] `AGENTS.md` (raiz)
- [x] `frontend/AGENTS.md`
- [x] `backend/AGENTS.md`
- [x] `docs/SECURITY.md` ← NOVO
- [ ] `backend/core/AGENTS.md` (ETL/anticorrupção SAP)
- [ ] `backend/ia/AGENTS.md` (SQL Agent + ECharts)
- [ ] `backend/api/AGENTS.md` (routers REST)
- [ ] `backend/db/AGENTS.md` (schemas + migrations)
- [ ] `frontend/app/(auth)/AGENTS.md`
- [ ] `frontend/app/cockpit/AGENTS.md`
- [ ] `frontend/app/analista/AGENTS.md`

### Fase 2: Documentação técnica

- [ ] `docs/PRD.md`
- [ ] `docs/DATA_CONTRACT.md` (schema do banco + colunas RAW/CALC)
- [ ] `docs/BUSINESS_RULES.md` (REGRAS_HORARIO, classificação ordens)
- [ ] `docs/HISTORICAL_BUGS.md` (lições do projeto anterior)
- [ ] `docs/CHANGELOG.md`

### Fase 3: Skills (Workflows do Antigravity)

- [ ] `/criar-skill` (meta-skill)
- [ ] `/criar-componente`
- [ ] `/criar-router`
- [ ] `/criar-migration`
- [ ] `/refatorar-arquivo`
- [ ] `/registrar-bug`
- [ ] `/registrar-decisao`

### Fase 4: Setup técnico

- [ ] `.gitignore`
- [ ] `.env.example`
- [ ] `README.md`
- [ ] `frontend/package.json`
- [ ] `backend/requirements.txt`

---

## 🐛 BUGS HISTÓRICOS REGISTRADOS (do projeto anterior)

1. **BUG-001:** Datas com 56% de NaT (resolvido com Padrão Dual-Format)
2. **BUG-002:** `removeChild` no drill-down (resolvido com renderização declarativa)
3. **BUG-003:** `how='left'` perdia ordens sem apontamento (resolvido com `how='outer'`)
4. **DT-10:** `processamento.py` virou God Module com ~450 linhas (a refatorar no novo projeto)

---

## 🔄 PROTOCOLO DE RETOMADA

Quando começar nova conversa, eu (IA) DEVO:

1. ✅ Ler este arquivo COMPLETAMENTE antes de gerar qualquer código
2. ✅ Identificar o "PRÓXIMO ARQUIVO A CRIAR" (linha marcada com ⏳)
3. ✅ Confirmar com Filipe: "Vamos retomar em [arquivo X], correto?"
4. ✅ Só então gerar o conteúdo
5. ✅ Ao final, atualizar este arquivo (marcar item como [x] e mover ⏳)

---

## 📌 CHECKPOINTS DE CONSISTÊNCIA

Antes de gerar qualquer arquivo, eu (IA) DEVO verificar:

- [ ] O domínio é **mineradora** (não refinaria)
- [ ] Tempo do projeto anterior: **1 mês e 15 dias**
- [ ] Usar TanStack Query, NÃO useState pra dados servidos
- [ ] Usar SQL Agent, NÃO Pandas Agent
- [ ] Hierarquia de pastas conforme estrutura confirmada
- [ ] Limites de linha conforme tabela
- [ ] Cores HSL definidas em `lib/constants.ts`

---

> 🤝 **Este arquivo é o "save game"
