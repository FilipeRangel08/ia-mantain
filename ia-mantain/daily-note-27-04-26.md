# 📊 PROJECT_STATE.md

> **Propósito:** Memória viva do projeto. Registra decisões, contexto, pendências e dívidas técnicas.
> **Quando ler:** Antes de começar nova sessão de desenvolvimento.
> **Quando atualizar:** Ao final de cada sessão com decisões arquiteturais.

---

## 🎯 Visão Geral do Projeto

**Nome:** dashboard-manutencao
**Stack:** Python + FastAPI + Streamlit + Pandas + LangChain + Google Gemini + PostgreSQL
**Objetivo:** Gestão e análise de Ordens de Manutenção (SAP) com assistente IA conversacional.
**Fase atual:** 📐 Planejamento arquitetural (definição dos AGENTS.md)

---

## 📅 Histórico de Sessões

### Sessão 1 — Definição inicial (sessões anteriores)
- Estrutura geral do projeto definida
- AGENTS.md raiz criado
- `docs/ARCHITECTURE.md`, `docs/BUSINESS_RULES.md`, `docs/DATA_CONTRACT.md` criados
- Princípio DRY + cache + state management estabelecidos

### Sessão 2 — Refinamento do core (27/04/2026) ✅
- Análise das planilhas reais do cliente (Notas, Ordens, Horas)
- Decisões D1-D9 abaixo
- `backend/core/AGENTS.md` finalizado

---

## 🎯 Decisões Arquiteturais

### Sessão 2 — 27/04/2026

#### **D1. Tabela `apontamentos_horas` separada** ⭐
- **Decisão:** Apontamentos de horas vão em tabela dedicada, não na tabela `ordens`.
- **Por quê:** 1 ordem tem N apontamentos (vários operadores, vários dias). Modelar como
  coluna em `ordens` quebraria a normalização e impediria análise por operador/turno.
- **Junção:** via `numero_ordem` (FK lógica, sem REFERENCES SQL).
- **Por que sem FK rígida:** apontamento pode chegar antes da ordem ser importada.

#### **D2. Princípio de Robustez (Lei de Postel)**
- **Decisão:** 3 categorias de colunas no upload: mapeadas / ignoradas-conhecidas / desconhecidas.
- **Por quê:** Planilhas SAP podem ganhar colunas extras sem aviso. Sistema não pode quebrar
  ao receber coluna desconhecida.
- **Implementação:** `COLUNAS_IGNORADAS_*` em `constants.py` + warning no log para desconhecidas.

#### **D3. Categorias de campos: RAW / DERIVED / CALC**
- **Decisão:** Persistir DERIVED (`status_canonico`, `classificacao`, `turno`) no banco.
- **Por quê:** Performance em queries + estabilidade quando fonte muda + permite versionamento de regras.
- **Versionamento:** `versao_derivador = 'v1'` permite reprocessar histórico quando regra muda.

#### **D4. Multi-source com `fonte_dado`**
- **Decisão:** Coluna `fonte_dado` na tabela `ordens` diferencia origem (notas/ordens/api).
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
- **Por quê:** Coluna `Duração da parada` vem zerada do SAP (preenchimento manual descontinuado pelo cliente).
  Calcular via `fim_avaria - inicio_avaria` exigiria validação de qualidade dos dados que não cabe no MVP.
- **Frontend:** card de MTTR mostra mensagem "Em atualização" + ícone (?) que abre modal explicativo.
- **Registrado como:** TD-02 (tech debt).

#### **D9. Documentação dedicada ao usuário final**
- **Decisão:** 3 manuais novos em `docs/`:
  - `MANUAL_USUARIO.md` — passo a passo de exportação SAP + uso do dashboard
  - `MANUAL_SUPORTE.md` — troubleshooting + reset de dados
  - `LIMITACOES_MVP.md` — transparência sobre o que ainda não funciona
- **Por quê:** Reduz pressão sobre suporte; cria expectativa correta no usuário.

---

## 🏗️ Estado Atual da Arquitetura

### Tabelas no banco (PostgreSQL)

| Tabela | Status | Notas |
|---|---|---|
| `ordens` | 📐 Modelada | Multi-source (notas/ordens/api). Campos RAW + DERIVED. |
| `apontamentos_horas` | 📐 Modelada | FK lógica para `ordens.numero_ordem`. |
| `uploads` | 📐 Modelada | Auditoria de uploads (qual arquivo, quando, por quem). |

### Camadas do projeto

```
projeto/
├── AGENTS.md                       ✅ Criado (sessão 1)
├── PROJECT_STATE.md                ✅ Atualizado (sessão 2)
├── docs/
│   ├── ARCHITECTURE.md             ✅ Criado (precisa revisão — ver pendências)
│   ├── BUSINESS_RULES.md           ✅ Criado (precisa revisão)
│   ├── DATA_CONTRACT.md            ✅ Criado (precisa revisão)
│   ├── MANUAL_USUARIO.md           ⏳ A criar
│   ├── MANUAL_SUPORTE.md           ⏳ A criar
│   └── LIMITACOES_MVP.md           ⏳ A criar
├── backend/
│   ├── AGENTS.md                   ⏳ A criar
│   ├── core/
│   │   └── AGENTS.md               ✅ Finalizado (sessão 2)
│   ├── api/
│   │   └── AGENTS.md               ⏳ A criar (próximo)
│   └── ia/
│       └── AGENTS.md               ⏳ A criar (após API)
└── frontend/
    └── AGENTS.md                   ⏳ A criar (último)
```

---

## 📋 Pendências de Revisão

> **⚠️ Importante:** Estas pendências são consequência das decisões da Sessão 2.
> Devem ser revisadas **na fase final**, após todos os AGENTS.md estarem prontos.

### 🔴 Alta prioridade (impacto arquitetural)

#### **P1. `docs/ARCHITECTURE.md`** — Adicionar conceitos novos
- [ ] Adicionar tabela `apontamentos_horas` no diagrama de banco
- [ ] Atualizar fluxo de upload para 3 fontes (Notas / Horas / Ordens)
- [ ] Documentar relação 1:N entre `ordens` e `apontamentos_horas`
- [ ] Explicar princípio de FK lógica (sem REFERENCES SQL)
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

---

## 🚧 Tech Debt — Registro Formal

> Tech debt = decisões conscientes de "fazer pior agora pra entregar mais rápido".
> Cada item tem ID, prioridade, motivo, custo estimado e plano de remediação.

| ID | Item | Prioridade | Motivo da postergação | Custo de remediação | Quando atacar |
|---|---|---|---|---|---|
| **TD-01** | Implementar `proc_ordens.py` (suporte ao layout ORDENS) | 🟡 Média | Cliente usa layout NOTAS hoje. Migração será manual e planejada. | ~3 dias (pipeline + testes) | Quando cliente decidir migrar |
| **TD-02** | Calcular MTTR via `fim_avaria - inicio_avaria` | 🟢 Baixa | Coluna `Duração da parada` zerada no SAP. Falta validação de qualidade dos timestamps. | ~2 dias (lógica + validação + UI) | Após estabilização do MVP |
| **TD-03** | Endpoint `/admin/reprocessar-derivacoes` | 🟢 Baixa | Versão `v1` dos derivadores está estável. Necessário só quando mudar regra. | ~1 dia (endpoint + job assíncrono) | Quando precisar mudar `VERSAO_DERIVADOR` |
| **TD-04** | Endpoint `/admin/reset-dados` | 🟠 **Alta** | **Essencial antes de produção.** Usuário precisa poder limpar tudo ao trocar de layout. | ~0.5 dia (endpoint + confirmação UI) | **Pré-produção** |
| **TD-05** | Integração SAP direta (fonte `API_SAP`) | 🔵 Longo prazo | Requer credenciais SAP, biblioteca específica, segurança. Fora de escopo do MVP. | ~10 dias (estudo + integração + testes) | Roadmap V2 |
| **TD-06** | Detecção automática do tipo de planilha no upload | 🟢 Baixa | UX atual com 3 cards é clara. Detecção automática seria "nice to have". | ~1 dia (heurística + fallback) | Pós-MVP |
| **TD-07** | Análise financeira (custos) | 🟡 Média | Custos só vêm no layout ORDENS. Bloqueado por TD-01. | ~2 dias (queries + UI) | Após TD-01 |

### Convenções

- 🔴 **Crítica** = Bloqueia produção / segurança
- 🟠 **Alta** = Necessária pré-produção
- 🟡 **Média** = Importante mas não bloqueante
- 🟢 **Baixa** = Melhoria incremental
- 🔵 **Longo prazo** = Roadmap futuro

---

## 🎓 Aprendizados / Princípios Reforçados

> Coleção de princípios validados durante o desenvolvimento. Servem de referência para
> decisões futuras em situações análogas.

1. **Princípio de Robustez (Postel)** — Liberal na entrada, conservador na saída.
   *Aplicado em D2.*

2. **DERIVED > CALC quando há multi-source** — Quando a regra de derivação depende da fonte,
   persistir é mais seguro que calcular em runtime.
   *Aplicado em D3.*

3. **FK lógica > FK rígida em pipelines de ETL** — Dados podem chegar fora de ordem;
   validação na aplicação é mais flexível.
   *Aplicado em D1.*

4. **Versionamento de regras > regras mutáveis** — Quando uma lógica de negócio pode mudar,
   versionar permite reprocessar histórico.
   *Aplicado em D3 (`VERSAO_DERIVADOR`).*

5. **Documentação ao usuário > suporte reativo** — Investir em manual reduz pressão sobre suporte
   e cria expectativa correta.
   *Aplicado em D9.*

6. **Tech debt declarado > tech debt escondido** — Registrar formalmente o que foi postergado
   evita "amnésia coletiva" e priorização ruim.
   *Aplicado em todo TD-*.*

7. **Testar regex com dados reais > testar mentalmente** — `\bprevent` parecia funcionar mas
   não pegava `"PREV-001"`. Sempre validar com fixtures do SAP.
   *Aplicado em D7.*

---

## 🚀 Próximos Passos

### Imediato (próxima sessão)
- [ ] Criar `backend/api/AGENTS.md` (FastAPI: rotas, Pydantic, auth, validação de upload)

### Curto prazo
- [ ] Criar `backend/ia/AGENTS.md` (LangChain + Gemini + segurança)
- [ ] Criar `backend/AGENTS.md` (visão geral do backend)
- [ ] Criar `frontend/AGENTS.md` (Streamlit: tabs, cache, state)

### Antes de começar a implementar
- [ ] Revisão geral de **TODOS** os AGENTS.md já criados
- [ ] Resolver pendências P1-P7 listadas acima
- [ ] Criar manuais P5, P6, P7

### Pré-produção
- [ ] Implementar TD-04 (`/admin/reset-dados`) — bloqueante

---

## 📞 Contatos / Stakeholders

- **Product Owner / Dev:** Filipe (Santaluz, BR)
- **Cliente final:** Equipe de manutenção que usa SAP
- **Suporte:** A definir (possivelmente o próprio Filipe na fase MVP)
