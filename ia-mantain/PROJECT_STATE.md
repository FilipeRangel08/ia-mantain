# 📊 PROJECT_STATE.md — IA-MANTAIN

> **Propósito:** Memória viva do projeto. Snapshot de progresso + registro de decisões, contexto, pendências e dívidas técnicas.
> **Quando ler:** Antes de começar nova sessão de desenvolvimento (cole no início de qualquer conversa nova com IA).
> **Quando atualizar:** Ao final de cada sessão com decisões arquiteturais.
> **Última atualização:** 2026-04-27 (Sessão 4 — backend/ia/AGENTS.md)
> **Atualização anterior:** 2026-04-27 (Sessão 3 — backend/api/AGENTS.md)

---

## 🎯 IDENTIDADE DO PROJETO

- **Nome:** ia-mantain (sucessor de `dashboard-manutencao`)
- **Domínio:** Mineradora ⛏️ (NÃO é refinaria de petróleo)
- **Objetivo:** Gestão e análise de Ordens de Manutenção (SAP) com assistente IA conversacional
- **Localização:** `C:\Users\filip\Desktop\github\sistema-manutencao\ia-mantain`
- **Predecessor:** dashboard-manutencao (Streamlit, ~1 mês e 15 dias de desenvolvimento)
- **Mantenedor:** Filipe (Santaluz, BA, Brasil)
- **Fase atual:** 📐 Planejamento arquitetural (definição dos AGENTS.md + refinamento do core)

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
- LangChain + Google Gemini (`create_sql_agent`, **NÃO** `pandas_dataframe_agent`)

### Banco
- Supabase (PostgreSQL + Auth + RLS)

### Deploy (a definir)
- Frontend: Vercel
- Backend: Railway / Render / Fly.io
- Banco: Supabase Cloud

---

## 👥 RBAC — 3 PAPÉIS

> ⚠️ **Correção registrada na Sessão 3:** o documento original mencionava 4 papéis. Decisão final: **3 papéis** com herança.

1. **Viewer** 👁️ — vê cockpit + interage com IA
2. **Operator** 🔧 — tudo de Viewer + upload + manipulação de dados + edição (futura)
3. **Admin** 👑 — tudo de Operator + reset banco + config turnos + gestão usuários

Hierarquia: `Viewer < Operator < Admin` (com herança de permissões).

---

## ⛔ REGRAS CRÍTICAS (NUNCA VIOLAR)

1. **IA = SQL Agent**, NÃO Pandas Agent
2. **Datas = Padrão Dual-Format** (ISO8601 → fallback BR `dayfirst=True`)
3. **Colunas RAW vs DERIVED vs CALC** — CALC nunca persiste no banco; DERIVED persiste com `versao_derivador`
4. **Constantes canônicas** — nunca hardcode nomes de colunas SAP
5. **Validação dupla RBAC** — UI + Backend
6. **Drill-down declarativo** — nunca manipular DOM imperativo
7. **Outer Join no df_sap_completo** — nunca `how='left'`
8. **FK lógica entre `ordens` e `apontamentos_horas`** — sem `REFERENCES` SQL (apontamento pode chegar antes da ordem)
9. **Concatenação data+hora obrigatória** no pipeline (banco usa `TIMESTAMPTZ`)

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

## 📅 Histórico de Sessões

### Sessão 4 — Camada IA (27/04/2026) ✅
- `backend/ia/AGENTS.md` finalizado
- Decisões D16–D19 registradas
- Tech Debts TD-11, TD-12, TD-13, TD-14, TD-15 adicionados
- 5 camadas anti-prompt-injection definidas
- Views `vw_ia_*` especificadas (5 views no MVP)
- Tool `gerar_grafico` (ECharts) modelada

### Sessão 1 — Definição inicial
- Estrutura geral do projeto definida
- `AGENTS.md` raiz criado
- `docs/ARCHITECTURE.md`, `docs/BUSINESS_RULES.md`, `docs/DATA_CONTRACT.md` criados
- Princípio DRY + cache + state management estabelecidos
- `frontend/AGENTS.md`, `backend/AGENTS.md`, `docs/SECURITY.md` criados

### Sessão 2 — Refinamento do core (27/04/2026) ✅
- Análise das planilhas reais do cliente (Notas, Ordens, Horas)
- Decisões D1–D9 registradas (ver abaixo)
- `backend/core/AGENTS.md` finalizado
  
### Sessão 3 — Camada API (27/04/2026) ✅
- `backend/api/AGENTS.md` finalizado
- Decisões D10–D15 registradas (ver abaixo)
- Tech Debts TD-08, TD-09, TD-10 adicionados
- Correção: RBAC ajustado de 4 para 3 papéis (Viewer/Operator/Admin)
- Pendências H1–H6 do módulo Horas & Efetivo registradas para sessão dedicada

---

## 🎯 Decisões Arquiteturais

### Sessão 4 — 27/04/2026

#### **D16. Gemini 2.5 Flash como modelo padrão**
- **Decisão:** `gemini-2.5-flash`, temperature=0, max_output_tokens=2048.
- **Por quê:** Melhor custo/benefício pra SQL Agent, free tier amigável (15 req/min, 1.500/dia).
- **Fallback:** modelo configurável via `IA_GEMINI_MODEL`.

#### **D17. Memória conversacional simplificada (MVP)**
- **Decisão:** `ConversationBufferWindowMemory(k=5)` em memória do processo.
- **Por quê:** Implementação rápida pra MVP. Persistência em banco fica como TD-11.
- **Trade-off:** restart do servidor zera histórico.

#### **D18. Views dedicadas `vw_ia_*` como interface da IA**
- **Decisão:** IA acessa apenas 5 views (`vw_ia_ordens`, `vw_ia_apontamentos`, `vw_ia_indicadores_mes`, `vw_ia_bad_actors`, `vw_ia_efetivo`).
- **Por quê:** Esconde campos sensíveis, renomeia em PT-BR amigável, reduz tokens, isola schema.
- **Implementação:** role `ia_readonly` com `GRANT SELECT` apenas nessas views. `REVOKE ALL` em tabelas.

#### **D19. Tool `gerar_grafico` para visualizações**
- **Decisão:** IA chama tool dedicada que retorna JSON config Apache ECharts.
- **Por quê:** IA decide quando faz sentido (vs sempre/nunca). Tipos suportados: bar, line, pie, scatter.
- **Frontend:** renderiza ECharts diretamente do JSON retornado.

### Sessão 3 — 27/04/2026

#### **D10. Supabase Auth como provider central**
- **Decisão:** Backend FastAPI **não emite JWT próprio** — apenas valida JWT do Supabase via JWKS.
- **Por quê:** Supabase já gerencia auth, sessions, recuperação de senha. Reinventar = trabalho duplicado e superfície de ataque maior.
- **Implementação:** middleware `verify_supabase_jwt` + cookie httpOnly `sb-access-token`.
- **Role:** vem de `app_metadata.role` no payload do JWT.

#### **D11. RBAC com 3 papéis hierárquicos**
- **Decisão:** Viewer < Operator < Admin (com herança).
- **Por quê:** MVP com 1-5 usuários não precisa de granularidade fina. Evolução pra mais papéis fica como roadmap.
- **Implementação:** `Role` StrEnum + `ROLE_HIERARCHY` dict + `require_role()` dependency.
- **Correção:** documento original mencionava 4 papéis (Visitante/Coordenador/Assistente/Supervisor) — descartado.

#### **D12. Upload sem persistência de arquivo (Opção A)**
- **Decisão:** Processa arquivo em memória, descarta após inserção. Tabela `uploads` registra apenas metadados.
- **Por quê:** MVP não precisa de auditoria física de arquivos. Evita custo de storage e complexidade de retenção.
- **Metadados persistidos:** hash SHA-256, nome original, tamanho, linhas processadas, status, fonte, usuário, timestamp.
- **Evolução futura:** TD-09 (storage físico para auditoria).

#### **D13. Configuração de turnos no sistema (não no SAP)**
- **Decisão:** Tabela `configuracao_turnos` editável por Admin. Derivador `turno` consulta config atual.
- **Por quê:** SAP não tem campo de turno; é regra interna da fábrica e pode mudar.
- **Implicação:** mudança de config exige reprocessamento dos registros existentes (TD-08).

#### **D14. Indicadores comparativos como categoria principal**
- **Decisão:** 3 famílias de indicadores no MVP:
  - **Comparativo mensal** (mês X vs mês Y)
  - **Desempenho geral** (consolidado por período)
  - **Bad actors** (drill-down por equipamento)
- **Endpoints estruturados genericamente.** Detalhamento de cada KPI em sessão dedicada.

#### **D15. Módulo Horas & Efetivo — estrutura preservada da spec antiga**
- **Decisão:** Aproveitar a spec `skill_horas_efetivo.md` como base. Endpoints REST estruturados, lógica adaptada do projeto Streamlit.
- **Tabela nova:** `efetivo_planejado` (matricula, nome, regime, horas S1-S5, mensal).
- **Endpoints:** `/efetivo/*`, `/colaboradores/{matricula}/raio-x`, `/indicadores/horas-comparativo`.
- **Pendências H1-H6** registradas para sessão dedicada (UI de edição, auto-save, regra Plan_Mes, visões mensal/semanal, métricas raio-X, filtros drill-down).

### Sessão 2 — 27/04/2026

#### **D1. Tabela `apontamentos_horas` separada** ⭐
- **Decisão:** Apontamentos de horas vão em tabela dedicada, não na tabela `ordens`.
- **Por quê:** 1 ordem tem N apontamentos (vários operadores, vários dias). Modelar como coluna em `ordens` quebraria a normalização e impediria análise por operador/turno.
- **Junção:** via `numero_ordem` (FK lógica, sem `REFERENCES` SQL).
- **Por que sem FK rígida:** apontamento pode chegar antes da ordem ser importada.

#### **D2. Princípio de Robustez (Lei de Postel)**
- **Decisão:** 3 categorias de colunas no upload: mapeadas / ignoradas-conhecidas / desconhecidas.
- **Por quê:** Planilhas SAP podem ganhar colunas extras sem aviso. Sistema não pode quebrar ao receber coluna desconhecida.
- **Implementação:** `COLUNAS_IGNORADAS_*` em `constants.py` + warning no log para desconhecidas.

#### **D3. Categorias de campos: RAW / DERIVED / CALC**
- **Decisão:** Persistir DERIVED (`status_canonico`, `classificacao`, `turno`) no banco.
- **Por quê:** Performance em queries + estabilidade quando fonte muda + permite versionamento de regras.
- **Versionamento:** `versao_derivador = 'v1'` permite reprocessar histórico quando regra muda.

#### **D4. Multi-source com `fonte_dado`**
- **Decisão:** Coluna `fonte_dado` na tabela `ordens` diferencia origem (notas / ordens / api).
- **Por quê:** Permite suportar múltiplos layouts SAP sem migrar dados antigos.
- **MVP:** apenas `fonte_dado = 'notas'`. ORDENS preparado mas não implementado.
- **Trade-off:** algumas colunas ficam NULL conforme fonte (ex: custos só em ORDENS).

#### **D5. Concatenação data+hora obrigatória**
- **Decisão:** Toda planilha SAP que separa data e hora é mesclada em datetime único no pipeline.
- **Por quê:** Banco armazena `TIMESTAMPTZ` único; turno/duração precisam de datetime completo.
- **Aplicação:** notas (`inicio_avaria`, `fim_avaria`), horas (`data_inicio_real`, `data_fim_real`).

#### **D6. Upload com 3 cards independentes**
- **Decisão:** Frontend tem cards separados para Notas / Horas / Ordens (Ordens bloqueado no MVP).
- **Por quê:** Usuário entende claramente qual planilha está enviando; validação por tipo é mais clara.
- **UX:** drag-and-drop em cada card, botão "Processar" só ativa com algum arquivo carregado.

#### **D7. Regex `\bprev` em vez de `\bprevent`**
- **Decisão:** Padrão de classificação preventiva usa radical curto.
- **Por quê:** Pega `"PREV-001"` que não casava com `prevent`.
- **Aprendizado:** sempre testar regex com casos reais SAP antes de incorporar.

#### **D8. MTTR adiado para versão futura** ⏳
- **Decisão:** Não calcular MTTR no MVP.
- **Por quê:** Coluna `Duração da parada` vem zerada do SAP (preenchimento manual descontinuado pelo cliente). Calcular via `fim_avaria - inicio_avaria` exigiria validação de qualidade dos dados que não cabe no MVP.
- **Frontend:** card de MTTR mostra mensagem "Em atualização" + ícone (?) que abre modal explicativo.
- **Registrado como:** TD-02 (tech debt).

#### **D9. Documentação dedicada ao usuário final**
- **Decisão:** 3 manuais novos em `docs/`:
  - `MANUAL_USUARIO.md` — passo a passo de exportação SAP + uso do dashboard
  - `MANUAL_SUPORTE.md` — troubleshooting + reset de dados
  - `LIMITACOES_MVP.md` — transparência sobre o que ainda não funciona
- **Por quê:** Reduz pressão sobre suporte; cria expectativa correta no usuário.

---

## 🔒 Decisões de segurança registradas

- httpOnly cookies para JWT
- Usuário `ia_readonly` para LangChain
- Validação MIME via `python-magic`
- System prompt blindado contra injection
- Rate limit IA: 30/hora/usuário

---

## 🏗️ Estado Atual da Arquitetura

### Tabelas no banco (PostgreSQL)

| Tabela | Status | Notas |
|---|---|---|
| `ordens` | 📐 Modelada | Multi-source (notas/ordens/api). Campos RAW + DERIVED. |
| `apontamentos_horas` | 📐 Modelada | FK lógica para `ordens.numero_ordem`. |
| `uploads` | 📐 Modelada | Auditoria de uploads (qual arquivo, quando, por quem). |

### Estrutura de pastas

```
ia-mantain/
├── AGENTS.md                       ✅ Criado (Sessão 1) + Seção 6 Segurança
├── PROJECT_STATE.md                ✅ Atualizado (Sessão 2)
├── docs/
│   ├── ARCHITECTURE.md             ⏳ A criar
│   ├── BUSINESS_RULES.md           ⏳ A criar
│   ├── DATA_CONTRACT.md            ⏳ A criar
│   ├── SECURITY.md                 ✅ Criado (Sessão 1)
│   ├── PRD.md                      ⏳ A criar
│   ├── HISTORICAL_BUGS.md          ⏳ A criar
│   ├── CHANGELOG.md                ⏳ A criar
│   ├── MANUAL_USUARIO.md           ⏳ A criar (P5)
│   ├── MANUAL_SUPORTE.md           ⏳ A criar (P6)
│   └── LIMITACOES_MVP.md           ⏳ A criar (P7)
├── backend/
│   ├── AGENTS.md                   ✅ Criado (Sessão 1)
│   ├── core/
│   │   └── AGENTS.md               ✅ Finalizado (Sessão 2)
│   ├── api/
│   │   └── AGENTS.md               ✅ Finalizado (Sessão 3)
│   ├── ia/
│   │   └── AGENTS.md               ✅ Finalizado (Sessão 4)
│   └── db/
│       └── AGENTS.md               ⏳ A criar
└── frontend/
    ├── AGENTS.md                   ⏳ A criar
    └── app/
        ├── (auth)/AGENTS.md        ⏳ A criar
        ├── cockpit/AGENTS.md       ⏳ A criar
        └── analista/AGENTS.md      ⏳ A criar
```

### ⏳ PRÓXIMO ARQUIVO A CRIAR

- [ ] `backend/db/AGENTS.md` (modelos SQLAlchemy + migrations Alembic + RLS Supabase)

---

## 📋 Pendências de Revisão

> **⚠️ Importante:** Estas pendências são consequência das decisões da Sessão 2.
> Devem ser revisadas **na fase final**, após todos os AGENTS.md estarem prontos.

### 🔴 Alta prioridade (impacto arquitetural)

#### **P1. `docs/ARCHITECTURE.md`** — Adicionar conceitos novos
- [ ] Adicionar tabela `apontamentos_horas` no diagrama de banco
- [ ] Atualizar fluxo de upload para 3 fontes (Notas / Horas / Ordens)
- [ ] Documentar relação 1:N entre `ordens` e `apontamentos_horas`
- [ ] Explicar princípio de FK lógica (sem `REFERENCES` SQL)
- [ ] Adicionar seção sobre versionamento de derivadores

#### **P2. `docs/DATA_CONTRACT.md`** — Adicionar contrato da planilha de Horas
- [ ] Documentar colunas RAW: `Nº pessoal`, `Ordem`, `Trabalho real`, datas/horas
- [ ] Documentar regra de concatenação data+hora
- [ ] Documentar que `numero_ordem` é a chave de junção
- [ ] Adicionar exemplos de linhas válidas/inválidas

#### **P3. `docs/BUSINESS_RULES.md`** — Adicionar regras novas
- [ ] Regra "1 ordem pode ter N apontamentos de hora"
- [ ] Regra "MTTR adiado — não calcular no MVP"
- [ ] Regra "Mistura de fontes não recomendada — usar reset"
- [ ] Regra de derivação de status por fonte (notas vs ordens)

### 🟡 Média prioridade (consistência)

#### **P4. `AGENTS.md` raiz** — Atualizar visão geral
- [ ] Mencionar "Apontamento de Horas" como 3ª fonte de dados
- [ ] Atualizar diagrama de fluxo de dados
- [ ] Confirmar que regras de DRY/cache continuam válidas com nova arquitetura

### 🟢 Baixa prioridade (criação nova)

#### **P5. Criar `docs/MANUAL_USUARIO.md`**
- [ ] Passo a passo: como exportar planilha do SAP (Notas / Horas / Ordens)
- [ ] Passo a passo: como subir cada planilha no dashboard
- [ ] Glossário de termos (turno, classificação, MTTR, etc)
- [ ] FAQ comum

#### **P6. Criar `docs/MANUAL_SUPORTE.md`**
- [ ] Como resetar dados via endpoint admin
- [ ] Como reprocessar derivadores (versão nova)
- [ ] Como adicionar nova `coluna ignorada` em `constants.py`
- [ ] Troubleshooting comum (erros de upload, datas inválidas, etc)

#### **P7. Criar `docs/LIMITACOES_MVP.md`**
- [ ] ⚠️ MTTR não disponível
- [ ] ⚠️ Layout ORDENS não suportado
- [ ] ⚠️ Custos só em ORDENS (futuro)
- [ ] ⚠️ Sem integração SAP direta
- [ ] ⚠️ Mistura de fontes exige reset
      
#### **P8. `PROJECT_STATE.md` — Correção RBAC** ✅ (resolvido nesta atualização)
- [x] Mudar de 4 para 3 papéis (Viewer/Operator/Admin)

#### **P9. Sessão dedicada — Módulo Horas & Efetivo**
- [ ] Resolver H1: UI de edição (tabela editável vs form)
- [ ] Resolver H2: estratégia de save (auto-save vs botão)
- [ ] Resolver H3: regra Plan_Mes (S1-S4+S5 parcial vs mês corrido)
- [ ] Resolver H4: manter visões mensal e semanal?
- [ ] Resolver H5: confirmar métricas do raio-X
- [ ] Resolver H6: filtros do drill-down
---

## 🚧 Tech Debt — Registro Formal

> Tech debt = decisões conscientes de "fazer pior agora pra entregar mais rápido".
> Cada item tem ID, prioridade, motivo, custo estimado e plano de remediação.

| ID | Item | Prioridade | Motivo da postergação | Custo | Quando atacar |
|---|---|---|---|---|---|
| **TD-01** | Implementar `proc_ordens.py` (suporte ao layout ORDENS) | 🟡 Média | Cliente usa layout NOTAS hoje. Migração será manual e planejada. | ~3 dias | Quando cliente decidir migrar |
| **TD-02** | Calcular MTTR via `fim_avaria - inicio_avaria` | 🟢 Baixa | Coluna `Duração da parada` zerada no SAP. Falta validação de qualidade dos timestamps. | ~2 dias | Após estabilização do MVP |
| **TD-03** | Endpoint `/admin/reprocessar-derivacoes` | 🟢 Baixa | Versão `v1` dos derivadores está estável. Necessário só quando mudar regra. | ~1 dia | Quando precisar mudar `VERSAO_DERIVADOR` |
| **TD-04** | Endpoint `/admin/reset-dados` | 🟠 **Alta** | **Essencial antes de produção.** Usuário precisa poder limpar tudo ao trocar de layout. | ~0.5 dia | **Pré-produção** |
| **TD-05** | Integração SAP direta (fonte `API_SAP`) | 🔵 Longo prazo | Requer credenciais SAP, biblioteca específica, segurança. Fora de escopo do MVP. | ~10 dias | Roadmap V2 |
| **TD-06** | Detecção automática do tipo de planilha no upload | 🟢 Baixa | UX atual com 3 cards é clara. Detecção automática seria "nice to have". | ~1 dia | Pós-MVP |
| **TD-07** | Análise financeira (custos) | 🟡 Média | Custos só vêm no layout ORDENS. Bloqueado por TD-01. | ~2 dias | Após TD-01 |
| **TD-08** | Reprocessamento automático ao mudar config de turno | 🟡 Média | Ao alterar `configuracao_turnos`, registros antigos ficam com turno desatualizado. Job assíncrono não é trivial. | ~1 dia | Quando admin mudar turnos pela 1ª vez |
| **TD-09** | Auditoria física de arquivos (storage de uploads originais) | 🟢 Baixa | MVP só guarda metadados. Para auditoria completa precisaria S3/disco organizado. | ~2 dias | Pós-MVP, sob demanda do cliente |
| **TD-10** | Detalhamento dos endpoints de Horas & Efetivo | 🟡 Média | Estrutura genérica criada, falta resolver pendências H1-H6 em sessão dedicada. | ~1 dia | Sessão dedicada do módulo Horas |
| **DT-10** | Refatorar `processamento.py` (era God Module ~450 linhas no projeto anterior) | 🟡 Média | Já mitigado pela nova estrutura modular (`core/proc_*.py`). | — | Validar na primeira implementação |
| **TD-11** | Persistir histórico de chat em tabela `ia_conversations` | 🟡 Média | Memória in-memory perde no restart, sem auditoria | ~1 dia | Pós-MVP |
| **TD-12** | Streaming de resposta IA via SSE | 🟢 Baixa | UX progressiva, não bloqueia | ~2 dias | Após validação MVP |
| **TD-13** | Cache Redis de respostas IA (TTL 1h) | 🟡 Média | Reduz custo + latência em perguntas repetidas | ~1 dia | Quando custo escalar |
| **TD-14** | Fine-tuning do system prompt com logs reais | 🟢 Baixa | Iteração contínua baseada em uso | Ongoing | Após 1.000+ perguntas |
| **TD-15** | Tool `glossario_lookup` dedicada | 🟢 Baixa | Hoje glossário fica no system prompt (mais tokens) | ~0,5 dia | Quando prompt crescer |

### Convenções de prioridade

- 🔴 **Crítica** = Bloqueia produção / segurança
- 🟠 **Alta** = Necessária pré-produção
- 🟡 **Média** = Importante mas não bloqueante
- 🟢 **Baixa** = Melhoria incremental
- 🔵 **Longo prazo** = Roadmap futuro

---

## 🐛 Bugs Históricos Registrados (do projeto anterior)

| ID | Descrição | Status |
|---|---|---|
| **BUG-001** | Datas com 56% de NaT | ✅ Resolvido com Padrão Dual-Format |
| **BUG-002** | `removeChild` no drill-down | ✅ Resolvido com renderização declarativa |
| **BUG-003** | `how='left'` perdia ordens sem apontamento | ✅ Resolvido com `how='outer'` |
| **DT-10** | `processamento.py` virou God Module (~450 linhas) | ⏳ A refatorar — ver tabela Tech Debt |

---

## 🎓 Aprendizados / Princípios Reforçados

> Coleção de princípios validados durante o desenvolvimento. Servem de referência para decisões futuras em situações análogas.

1. **Princípio de Robustez (Postel)** — Liberal na entrada, conservador na saída. *Aplicado em D2.*
2. **DERIVED > CALC quando há multi-source** — Quando a regra de derivação depende da fonte, persistir é mais seguro que calcular em runtime. *Aplicado em D3.*
3. **FK lógica > FK rígida em pipelines de ETL** — Dados podem chegar fora de ordem; validação na aplicação é mais flexível. *Aplicado em D1.*
4. **Versionamento de regras > regras mutáveis** — Quando uma lógica de negócio pode mudar, versionar permite reprocessar histórico. *Aplicado em D3 (`VERSAO_DERIVADOR`).*
5. **Documentação ao usuário > suporte reativo** — Investir em manual reduz pressão sobre suporte e cria expectativa correta. *Aplicado em D9.*
6. **Tech debt declarado > tech debt escondido** — Registrar formalmente o que foi postergado evita "amnésia coletiva" e priorização ruim. *Aplicado em todo TD-\*.*
7. **Testar regex com dados reais > testar mentalmente** — `\bprevent` parecia funcionar mas não pegava `"PREV-001"`. Sempre validar com fixtures do SAP. *Aplicado em D7.*

---

## 📋 ROADMAP COMPLETO

### Fase 1: AGENTS.md hierárquicos
- [x] `AGENTS.md` (raiz)
- [ ] `frontend/AGENTS.md`
- [x] `backend/AGENTS.md`
- [x] `docs/SECURITY.md`
- [x] `backend/core/AGENTS.md` ← Finalizado na Sessão 2
- [x] `backend/api/AGENTS.md` ← **PRÓXIMO**
- [ ] `backend/ia/AGENTS.md` (SQL Agent + ECharts)
- [ ] `backend/db/AGENTS.md` (schemas + migrations)
- [ ] `frontend/app/(auth)/AGENTS.md`
- [ ] `frontend/app/cockpit/AGENTS.md`
- [ ] `frontend/app/analista/AGENTS.md`

### Fase 2: Documentação técnica
- [ ] `docs/PRD.md`
- [ ] `docs/DATA_CONTRACT.md` (criado — revisar P2)
- [ ] `docs/BUSINESS_RULES.md` (criado — revisar P3)
- [ ] `docs/ARCHITECTURE.md` (criado — revisar P1)
- [ ] `docs/HISTORICAL_BUGS.md`
- [ ] `docs/CHANGELOG.md`
- [ ] `docs/MANUAL_USUARIO.md` (P5)
- [ ] `docs/MANUAL_SUPORTE.md` (P6)
- [ ] `docs/LIMITACOES_MVP.md` (P7)

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

## 🚀 Próximos Passos

### Imediato (próxima sessão)
- [ ] Criar `backend/api/AGENTS.md` (FastAPI: rotas, Pydantic, auth, validação de upload)

### Curto prazo
- [ ] Criar `backend/ia/AGENTS.md` (LangChain + Gemini + segurança)
- [ ] Criar `backend/db/AGENTS.md` (schemas + migrations)
- [ ] Criar AGENTS.md por rota do frontend (`(auth)`, `cockpit`, `analista`)

### Antes de começar a implementar
- [ ] Revisão geral de **TODOS** os AGENTS.md já criados
- [ ] Resolver pendências **P1–P7** listadas acima
- [ ] Criar manuais **P5, P6, P7**

### Pré-produção
- [ ] Implementar **TD-04** (`/admin/reset-dados`) — bloqueante

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
- [ ] Usar **TanStack Query**, NÃO `useState` pra dados servidos
- [ ] Usar **SQL Agent**, NÃO Pandas Agent
- [ ] Hierarquia de pastas conforme estrutura confirmada
- [ ] Limites de linha conforme tabela
- [ ] Cores HSL definidas em `lib/constants.ts`
- [ ] **Apontamentos de horas em tabela separada** (D1)
- [ ] **FK lógica** entre `ordens` e `apontamentos_horas` (sem `REFERENCES`)
- [ ] **Concatenar data+hora** no pipeline antes de persistir (D5)
- [ ] **Não calcular MTTR** no MVP (D8 / TD-02)
- [ ] **RBAC com 3 papéis** (Viewer/Operator/Admin) — não 4 (D11)
- [ ] **Supabase Auth** valida JWT, não emite (D10)
- [ ] **Upload Opção A** — não persiste arquivo, só metadados (D12)
- [ ] **Turnos configuráveis** no sistema, não no SAP (D13)

---

## 📞 Contatos / Stakeholders

- **Product Owner / Dev:** Filipe (Santaluz, BA, Brasil)
- **Cliente final:** Equipe de manutenção que usa SAP (mineradora)
- **Suporte:** A definir (possivelmente o próprio Filipe na fase MVP)

---

> 🤝 **Este arquivo é o "save game" do projeto.** Mantenha-o atualizado ao final de cada sessão.
