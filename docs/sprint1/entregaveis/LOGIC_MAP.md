# LOGIC_MAP.md - Biveto App

## 0) Resumo Executivo

**Biveto** é uma aplicação de finanças pessoais completa com as seguintes capacidades principais:

- **Upload inteligente de documentos** (faturas, cupons) com OCR via Google Gemini
- **Gestão de transações** com suporte a parcelas, recorrentes e múltiplas contas
- **Faturas de cartão de crédito** com cálculo automático de períodos e projeções futuras
- **Sistema familiar** (household) com licenças e convites
- **Analytics e insights** sobre gastos com detecção de "vazamentos"
- **Assistente IA** para consultas financeiras

**Stack:** FastAPI + SQLAlchemy (async) | React 18 + TypeScript + Zustand | PostgreSQL/SQLite | Tailwind CSS

**Como rodar:** `make install && make dev` (backend :8000, frontend :5173)

---

## 1) Arquitetura & Módulos

### 1.1 Visão Geral

```
biveto-app/
├── backend/                    # API REST (Python/FastAPI)
│   └── app/
│       ├── core/               # Config, DB, Auth, Security
│       ├── models/             # 19 SQLAlchemy models
│       ├── schemas/            # 20 Pydantic schemas
│       ├── routers/            # 18 FastAPI routers
│       └── services/           # 21 business logic services
├── frontend/                   # SPA (React/TypeScript)
│   └── src/
│       ├── pages/              # 20 páginas
│       ├── components/         # Componentes reutilizáveis
│       ├── services/api.ts     # Cliente HTTP (Axios)
│       └── stores/             # Zustand (auth, theme)
└── docker-compose.yml          # PostgreSQL para dev/prod
```

### 1.2 Dependências Principais

| Camada | Tecnologia | Uso |
|--------|------------|-----|
| **Backend** | FastAPI 0.109 | Framework REST |
| | SQLAlchemy 2.0 (async) | ORM |
| | Pydantic 2.6 | Validação/Schemas |
| | python-jose + bcrypt | JWT Auth |
| | Google Gemini API | OCR/Vision |
| | Alembic | Migrations |
| **Frontend** | React 18 + TypeScript | UI Framework |
| | Zustand | State local |
| | React Query | State servidor |
| | React Hook Form + Zod | Forms/Validação |
| | Tailwind CSS 3.4 | Styling |
| | Recharts | Gráficos |

### 1.3 Integrações Externas

| Serviço | Propósito | Evidência |
|---------|-----------|-----------|
| **Google Gemini** | OCR de documentos/faturas | `services/llm_ocr_service.py:269-480` |
| **OpenAI** | Provider alternativo (opcional) | `services/llm_ocr_service.py:605-647` |
| **Anthropic** | Provider alternativo (opcional) | `services/llm_ocr_service.py:649-698` |
| **SMTP (Hostinger)** | Email (convites, reset senha) | `services/email_service.py:154-213` |

---

## 2) Fluxos do Usuário

### 2.1 Mapa de Navegação

```
Login/Register
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                        LAYOUT PRINCIPAL                       │
├───────────────────────────────────────────────────────────────┤
│  Sidebar (Desktop)          │  Bottom Nav (Mobile)           │
│  ────────────────           │  ─────────────────             │
│  • Dashboard                │  • Home • Transações           │
│  • Transações               │  • [+] Adicionar               │
│  • Contas                   │  • Insights • Chat             │
│  • Cartões                  │                                │
│  • Faturas                  │                                │
│  • Receitas                 │                                │
│  • Orçamento                │                                │
│  • Metas                    │                                │
│  • Dívidas                  │                                │
│  • Recorrentes              │                                │
│  • Adicionar                │                                │
│  • Insights                 │                                │
│  • Assistente (Chat)        │                                │
│  • Admin (se is_admin)      │                                │
└───────────────────────────────────────────────────────────────┘
```

**Evidência:** `frontend/src/components/Layout.tsx:34-57`

### 2.2 Fluxos Críticos

### 2.2 Fluxo Financeiro Consolidado (Regra Mestra)
Para eliminar variações contábeis e ruídos no MVP, o Biveto App segue um fluxo financeiro estrito:
1. **Origem do Gasto (A Transação):** Toda transação de saída deve ser classificada como **Despesa Imediata** (debita de Conta Corrente/Poupança/Dinheiro) ou **Despesa a Prazo** (debita de Cartão de Crédito).
2. **Passivos Transitórios:** Transações de Cartão de Crédito não afetam o saldo líquido das contas bancárias até a liquidação. Elas apenas compõem a fatura do período.
3. **Liquidação (Pagamento Mestre):** O pagamento da fatura é o evento unificador. Ele debita o valor exato de uma Conta Base e zera o passivo da Fatura.

### 2.3 Fluxos Críticos Revisados

#### FLUXO 1: Upload e Conciliação de Transações (Sem Tolerâncias)
1. Usuário faz upload de Fatura/Comprovante (OCR via LLM).
2. Backend extrai itens e Frontend exibe lista para conciliação.
3. Usuário valida a transação:
   - Se for transação nova: O sistema cria a transação e consolida.
   - Se for parcela/recorrente: O sistema exige **Match Exato** de valor e data para unificar a previsão (`is_paid=false`) com a transação real. Divergências bloqueiam a automação e exigem confirmação manual, evitando corrupção de saldo.

#### FLUXO 2: Pagamento de Fatura (Liquidação)
1. Usuário acessa `/invoices`, seleciona a fatura e clica em "Pagar".
2. Backend valida Requisitos de Pagamento (REQ-PAG-01).
3. Cria transação de saída (`EXPENSE`) isolada na Conta Base.
4. Atualiza status da fatura para `PAID` ou `PARTIAL`.

---

### 2.3 Estados de Tela

| Tela | Loading | Empty | Error | Success |
|------|---------|-------|-------|---------|
| Dashboard | Spinner central | N/A (sempre tem dados) | Toast | Dados carregados |
| Transactions | Skeleton list | EmptyState component | Toast | Lista renderizada |
| Upload | Progress bar | UploadSelector | Mensagem inline | ExtractedItemsList |
| Invoices | Skeleton cards | "Nenhuma fatura" | Toast | Lista de faturas |

---

## 3) Regras de Negócio e Requisitos de Pagamento

### 3.1 Requisitos Registrados de Pagamentos
Para evitar ruídos no fluxo de caixa, os pagamentos obedecem a requisitos estritos:
* **REQ-PAG-01 (Liquidez Restrita):** Pagamentos de faturas só podem ter como origem contas com alta liquidez (`type IN ['checking', 'savings', 'cash']`). Tentar pagar usando outro cartão de crédito gera erro 400.
* **REQ-PAG-02 (Pagamentos Parciais):** Pagamentos menores que o montante total devido mudam o status da fatura para `PARTIAL`. O sistema rola o saldo devedor remanescente automaticamente para a fatura do mês subsequente.
* **REQ-PAG-03 (Isolamento de Analytics):** Transações geradas pela ação de "Pagamento de Fatura" recebem uma marcação interna e são ignoradas no cálculo do `total_expense` mensal, evitando a contagem dupla da mesma despesa no dashboard.

### 3.2 Tabela de Regras Funcionais (Atualizada)

| REGRA_ID | Descrição | Gatilho | Modificação no MVP |
|----------|-----------|---------|--------------------|
| **RN001** | Transação parcelada cria série automaticamente | is_installment=true | Mantida. |
| **RN002** | Parcelas futuras são projetadas (`is_paid=false`) | create_future_installments | Mantida. |
| **RN003** | Fatura calculada pelo closing_day do cartão | Criar transação no cartão | Mantida. |
| **RN004** | Detecção e Consolidação (Match Exato) | Confirmar transação | **Atualizada:** Tolerâncias removidas. Exige exatidão de centavos. |
| **RN005** | Restrição de Pagamento de Fatura | `pay_invoice()` | **Atualizada:** Base para o REQ-PAG-01. |
| **RN006** | Família (Household) unifica visão de gastos | Listar transações | Mantida. |
| **RN007** | Status fatura: OPEN→CLOSED→PAID/PARTIAL | Pagamento de Fatura | **Atualizada:** Absorveu o REQ-PAG-02. |
| **RN008** | Analytics exclui pagamentos de fatura | Calcular total_expense | **Atualizada:** Base para o REQ-PAG-03. |
| **RN009** | Merchant normalizado para matching | Criar transação | Mantida. |
| **RN010** | Recorrente verificada antes de criar | Criar recorrente | Adiada (Pós-MVP). |
| **RN011** | Série de parcelas match por tolerância 5% | Buscar série | **Removida:** Gerava ruído contábil. |
| **RN012** | Registro público desabilitado | Tentar registrar | Mantida. |
| **RN013** | Admin pode ver todos usuários/licenças | Acessar `/admin` | Mantida. |
| **RN014** | OCR usa prompt específico por banco | Processar fatura | Mantida. |

### Casos Extremos Conhecidos

| Regra | Caso | Comportamento |
|-------|------|---------------|
| RN003 | Compra dia 20, fechamento dia 15 | Vai para fatura do mês seguinte |
| RN003 | Vencimento < fechamento | due_date é no mês seguinte ao fechamento |
| RN004 | PDF reenviado | Detecta duplicata, retorna 409 |
| RN005 | Parcela projetada com valor diferente | Atualiza valor se diferença > 1 centavo |
| RN012 | Duas compras parceladas similares | Match por data compatível (±45 dias) |

### Regras Duplicadas/Divergentes

| Regra | Localização 1 | Localização 2 | Divergência |
|-------|---------------|---------------|-------------|
| Normalização merchant | `transaction_service.py:713-725` | `installment_service.py:637-650` | Idênticas (OK) |
| Household user IDs | `transaction_service.py:560-570` | `analytics_service.py:193-203` | Idênticas (OK) |

---

## 4) Modelo de Dados

### 4.1 Entidades Principais

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│     User     │────<│  HouseholdMb │>────│   License    │
└──────────────┘     └──────────────┘     └──────────────┘
       │                                         │
       │ 1:N                                     │
       ▼                                         │
┌──────────────┐     ┌──────────────┐            │
│   Account    │────<│ CreditCard   │            │
└──────────────┘     └──────────────┘            │
       │                    │                    │
       │                    ▼                    │
       │            ┌──────────────┐             │
       │            │   Invoice    │             │
       │            └──────────────┘             │
       │                    │                    │
       ▼                    ▼                    │
┌──────────────────────────────────────┐         │
│            Transaction               │         │
│  ├── category_id                     │         │
│  ├── merchant_id                     │         │
│  ├── credit_card_id                  │         │
│  ├── invoice_id                      │         │
│  ├── installment_series_id           │         │
│  ├── recurring_id                    │         │
│  └── document_id                     │         │
└──────────────────────────────────────┘         │
       │                                         │
       ▼                                         │
┌──────────────┐     ┌──────────────┐            │
│InstallmentSer│     │  Recurring   │            │
└──────────────┘     └──────────────┘            │
```

### 4.2 Invariantes do Domínio

| Entidade | Invariante | Evidência |
|----------|------------|-----------|
| **Transaction.type** | `expense`, `income`, `transfer` | `models/transaction.py:8-11` |
| **Transaction.is_paid** | Default true; false para projetadas | `models/transaction.py:68` |
| **Transaction.ownership_type** | `personal`, `household` | `models/transaction.py:75` |
| **CreditCardInvoice.status** | `open`→`closed`→`paid`/`partial`/`overdue` | `models/credit_card_invoice.py:9-15` |
| **InstallmentSeries.status** | `active`→`completed`/`cancelled` | `models/installment.py:8-11` |
| **RecurringTransaction.status** | `active`, `paused`, `cancelled` | `models/recurring.py:15-18` |
| **RecurringTransaction.frequency** | `daily`, `weekly`, `monthly`, `yearly` | `models/recurring.py:8-12` |
| **CreditCard.closing_day** | 1-31 | `models/credit_card.py:40` |
| **CreditCard.due_day** | 1-31 | `models/credit_card.py:41` |
| **Account.type** | `checking`, `savings`, `credit_card`, `cash`, `investment` | Implícito nos schemas |

### 4.3 Conversão/Serialização

| Camada | Tipo | Arquivo |
|--------|------|---------|
| Request → Model | Pydantic → SQLAlchemy | `schemas/*.py` |
| Model → Response | `from_attributes = True` | `schemas/transaction.py:110-111` |
| Frontend → API | TypeScript → JSON (Axios) | `services/api.ts` |
| Enum handling | `.value` para persistir | `transaction_service.py:121, 616` |

---

## 5) Contratos (API/DTO)

### 5.1 Endpoints Principais

| Método | Rota | Descrição | Auth | Request | Response |
|--------|------|-----------|------|---------|----------|
| POST | `/api/v1/auth/login` | Login | Não | `{email, password}` | `{access_token, token_type}` |
| POST | `/api/v1/auth/register/invite/{token}` | Registro via convite | Não | `{name, email, password}` | UserResponse |
| GET | `/api/v1/users/me` | Usuário atual | Sim | - | UserResponse |
| GET | `/api/v1/transactions` | Listar transações | Sim | Query params | TransactionResponse[] |
| POST | `/api/v1/transactions` | Criar transação | Sim | TransactionCreate | TransactionResponse |
| POST | `/api/v1/transactions/confirm` | Confirmar de documento | Sim | TransactionConfirm | ConfirmResult |
| POST | `/api/v1/documents` | Upload documento | Sim | FormData (file) | DocumentResponse |
| GET | `/api/v1/invoices` | Listar faturas | Sim | Query params | InvoiceResponse[] |
| POST | `/api/v1/invoices/{id}/pay` | Pagar fatura | Sim | `{amount, account_id}` | PaymentResponse |
| GET | `/api/v1/analytics/monthly` | Resumo mensal | Sim | `{year, month}` | AnalyticsSummary |
| POST | `/api/v1/chat/message` | Chat com assistente | Sim | `{message}` | ChatResponse |

### 5.2 Schemas Principais

**TransactionCreate** (`schemas/transaction.py:18-30`)
```python
{
  type: "expense"|"income"|"transfer",
  amount: float (>0),
  date: date,
  account_id: int,
  category_id?: int,
  merchant_name?: str,
  credit_card_id?: int,
  create_recurring?: bool,
  recurring_frequency?: str
}
```

**TransactionConfirm** (`schemas/transaction.py:33-61`)
```python
{
  document_id?: int,
  account_id: int,
  amount: float,
  date: date,
  credit_card_id?: int,
  invoice_month?: int,  # Do PDF
  invoice_year?: int,
  is_installment?: bool,
  installment_current?: int,
  installment_total?: int,
  create_future_installments?: bool,
  force_duplicate?: bool
}
```

### 5.3 Códigos de Erro

| Código | Situação | Evidência |
|--------|----------|-----------|
| 400 | Validação falhou / Pagar fatura com cartão | `invoice_service.py:277-279` |
| 401 | Token inválido/expirado | `core/security.py` |
| 404 | Recurso não encontrado | Múltiplos |
| 409 | Duplicata detectada | `transaction_service.py:178-196` |
| 429 | Rate limit excedido | `core/rate_limit.py` |

### 5.4 Inconsistências Front/Back

| Item | Frontend | Backend | Status |
|------|----------|---------|--------|
| amount | string (form) | float | OK (parseFloat) |
| date | string (ISO) | date | OK (pydantic) |
| category_id | string | int | OK (parseInt) |

---

## 6) Regras UI/UX do Código

### 6.1 Design Tokens

**Cores** (`tailwind.config.js:10-44`)
```javascript
primary: {
  900: '#1E3A5F',  // Brand navy (main)
  400: '#829ab1',
}
accent: {
  400: '#4ade80',  // Brand green (main)
}
biveto: {
  navy: '#1E3A5F',
  green: '#4ade80',
}
expense: '#ef4444',  // Red
income: '#22c55e',   // Green
```

**Tipografia**
```javascript
fontFamily: {
  sans: ['Inter', 'system-ui', 'sans-serif'],
}
```

**Dark Mode:** `darkMode: 'class'` - toggle via `themeStore.ts`

### 6.2 Padrões de Componente

| Padrão | Uso | Evidência |
|--------|-----|-----------|
| Cards | `.card` class (rounded-xl, shadow-sm, padding) | `globals.css` |
| Buttons | `.btn-primary`, `.btn-secondary`, `.btn-icon` | `globals.css` |
| Inputs | `.input` class | `globals.css` |
| Avatar | `.avatar`, `.avatar-sm` | `globals.css` |
| Gradiente | `.gradient-biveto` | Brand gradient |
| Glass effect | `.glass` | Blur + transparency |

### 6.3 Padrões de Formulário

| Item | Implementação | Evidência |
|------|---------------|-----------|
| Validação | React Hook Form + Zod | `pages/Login.tsx` |
| Erro inline | Mensagem sob input | `text-red-500` |
| Loading button | `LoadingButton` component | `components/LoadingButton.tsx` |
| Máscaras | Não detectado | Ponta solta |

### 6.4 Padrões de Navegação

| Pattern | Implementação |
|---------|---------------|
| Sidebar Desktop | Fixed left, 64px wide |
| Bottom Nav Mobile | Fixed bottom, 5 items |
| Botão + central | Abre QuickTransactionModal |
| Menu hamburger | Overlay + slide from left |

### 6.5 Estados de Loading/Empty/Error

| Componente | Loading | Empty | Error |
|------------|---------|-------|-------|
| Dashboard | Spinner | N/A | Toast |
| Lists | Skeleton | EmptyState | Toast |
| Upload | Progress bar | UploadSelector | Inline message |
| Forms | Button disabled | N/A | Inline message |

### 6.6 Violações de Padrão

| Tela | Violação | Severidade |
|------|----------|------------|
| Chat | Estilo próprio (markdown) | Baixa |

---

## 7) Pontas Soltas e Riscos

### 7.1 Código Comentado (Features Incompletas)

| Arquivo | Linha | Descrição |
|---------|-------|-----------|
| `routers/auth.py` | 32-40 | Registro público desabilitado (intencional) |

### 7.2 Funções Não Utilizadas

| Arquivo | Função | Status |
|---------|--------|--------|
| `services/ocr_service.py` | `extract_financial_data()` | Substituído por LLMOCRService |

### 7.3 Endpoints Sem Consumidor Aparente

| Endpoint | Situação |
|----------|----------|
| `POST /api/v1/transactions/check-projected` | Usado internamente pelo Upload |
| `PUT /api/v1/invoices/{id}/update-statuses` | Job futuro / admin manual |

### 7.4 Testes Faltando

| Fluxo Crítico | Cobertura |
|---------------|-----------|
| Upload + OCR + Confirm | Sem teste E2E |
| Pagamento de fatura | Sem teste |
| Parcelas futuras | Sem teste |
| Household transactions | Sem teste |

**Testes existentes:** `backend/tests/test_auth.py` (apenas auth)

### 7.5 TODOs/FIXMEs Encontrados

Nenhum TODO/FIXME explícito encontrado no código.

### 7.6 Riscos Identificados

| Risco | Impacto | Mitigação Sugerida |
|-------|---------|-------------------|
| OCR dependente de Google Gemini | Alto | Fallback para OpenAI/Anthropic implementado |
| Sem rate limit granular por endpoint | Médio | Configurar slowapi por rota |
| Sem soft delete | Médio | Dados perdidos são irrecuperáveis |
| Parcelas futuras podem acumular | Baixo | Job de limpeza ou limite |
| Prompts de OCR hardcoded | Baixo | Mover para config/database |

### 7.7 Débitos Técnicos

| Item | Descrição |
|------|-----------|
| OCRService legado | `ocr_service.py` pode ser removido |
| Tipos genéricos em mutations | `data: unknown` em vários lugares |
| Timezone UTC hardcoded | `datetime.utcnow()` sem timezone aware |
| Arquivos de PDF em disco | Sem cleanup automático em `backend/uploads/` |

---

## Apêndice: Arquivos Chave por Funcionalidade

| Funcionalidade | Backend | Frontend |
|----------------|---------|----------|
| **Autenticação** | `routers/auth.py`, `services/auth_service.py`, `core/security.py` | `stores/authStore.ts`, `pages/Login.tsx` |
| **Transações** | `routers/transactions.py`, `services/transaction_service.py` | `pages/Transactions.tsx`, `pages/Upload.tsx` |
| **Faturas** | `routers/invoices.py`, `services/invoice_service.py` | `pages/Invoices.tsx` |
| **Parcelas** | `services/installment_service.py` | `pages/Upload.tsx` (ExtractedItemsList) |
| **OCR** | `services/llm_ocr_service.py`, `services/document_service.py` | `pages/Upload.tsx` |
| **Analytics** | `routers/analytics.py`, `services/analytics_service.py` | `pages/Dashboard.tsx`, `pages/Insights.tsx` |
| **Família** | `routers/family.py` | `pages/Family.tsx` |
| **Admin** | `routers/admin.py` | `pages/admin/*.tsx` |

---
