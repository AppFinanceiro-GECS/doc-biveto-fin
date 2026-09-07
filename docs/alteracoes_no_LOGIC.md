### Resumo das Alterações: Escopo MVP e Regras de Negócio

#### 1. Criação do Fluxo Financeiro Consolidado (Nova Seção 2.2)
* **Como era:** O documento apenas listava os fluxos de sistema de forma isolada, sem amarrar a transição contábil do dinheiro.
* **O que mudou:** Inserção da "Regra Mestra do MVP", que formaliza que transações de cartão de crédito são apenas "passivos transitórios" e que o dinheiro real só sai da conta de liquidez do usuário no momento do pagamento da fatura.

#### 2. Remoção das Tolerâncias de IA e OCR (Seções 2.3 e 3.3)
* **Como era:** A antiga regra **RN012** fazia o agrupamento automático de compras com uma tolerância de até 5% no valor e permitia diferenças de centavos.
* **O que mudou:** As tolerâncias foram totalmente removidas para garantir a precisão do MVP. Agora exige-se um **Match Exato**. Caso o Google Gemini Flash (provedor padrão de IA do projeto[cite: 1]) leia valores divergentes, a consolidação automática é bloqueada e o sistema exige a revisão manual.

#### 3. Registro dos Requisitos de Pagamento (Nova Seção 3.1)
* **Como era:** A restrição de não pagar crédito usando crédito estava solta na documentação (antiga RN006).
* **O que mudou:** Criação e documentação de três requisitos oficiais:
  * **REQ-PAG-01 (Liquidez Restrita):** Restringe pagamentos exclusivamente para contas de liquidez (Corrente, Poupança ou Dinheiro em espécie).
  * **REQ-PAG-02 (Pagamentos Parciais):** Garante que saldos não pagos integram automaticamente a fatura do mês seguinte (rolagem de dívida).
  * **REQ-PAG-03 (Isolamento de Analytics):** Isola a transação de "Pagamento de Fatura" nos gráficos do Analytics, impedindo a visualização duplicada da mesma despesa.

#### 4. O que NÃO mudou (Arquitetura e Infraestrutura)
* A arquitetura base permanece inalterada: frontend construído em React e backend estruturado com FastAPI cobrindo os 29 módulos originais.
* O banco de dados padrão para o ambiente de desenvolvimento continua sendo o SQLite.
* A segurança e o acesso continuam utilizando a autenticação via JWT.
* O ambiente de execução local no Linux utilizará a versão atualizada da linguagem por meio do "alias" configurado para o Python 3.12 no terminal.

## A seguir as alterações que vão para o arquivo: 

### 2.2 Fluxo Financeiro Consolidado (Regra Mestra do MVP)

Para eliminar variações e garantir a exatidão contábil do MVP, o sistema obedece a um fluxo financeiro consolidado de três etapas:
1. **Origem do Gasto (A Transação):** Toda transação de saída deve ser classificada como **Despesa Imediata** (debita de Conta Corrente/Poupança/Dinheiro) ou **Despesa a Prazo** (debita de Cartão de Crédito).
2. **Passivos Transitórios:** Transações de Cartão de Crédito não afetam o saldo líquido das contas bancárias até a liquidação. Elas apenas compõem a fatura do período.
3. **Liquidação (Pagamento Mestre):** O pagamento da fatura é o evento unificador. Ele debita o valor exato de uma Conta Base e zera o passivo da Fatura (mudando status para `PAID` ou `PARTIAL`).

### 2.3 Fluxos Críticos Revisados

#### FLUXO 1: Upload e Conciliação de Transações (Sem Tolerâncias)
1. Usuário faz upload de Fatura/Comprovante (OCR via LLM)
2. Backend extrai itens e Frontend exibe lista para conciliação
3. Usuário valida a transação:
   - Se nova: Cria transação e consolida.
   - Se parcela/recorrente: O sistema exige "Match Exato" de valor e data para unificar a transação futura (`is_paid=false`) com a real. Se houver divergência, exige confirmação manual do usuário para evitar corrupção de saldo.

#### FLUXO 2: Pagamento de Fatura (Liquidação)
1. Usuário acessa `/invoices`, seleciona fatura `OPEN` ou `CLOSED` e clica "Pagar".
2. Backend valida Requisitos de Pagamento (REQ-PAG-01 e REQ-PAG-02).
3. Cria transação de saída (`EXPENSE`) isolada na Conta Base selecionada.
4. Atualiza status da fatura para `PAID` ou `PARTIAL`.

---

## 3) Regras de Negócio e Requisitos de Pagamento

### 3.1 Requisitos Registrados de Pagamentos
Para evitar ruídos no fluxo de caixa, os pagamentos obedecem aos seguintes requisitos estritos:
* **REQ-PAG-01 (Liquidez Restrita):** Pagamentos de faturas ou dívidas só podem ter como origem contas com alta liquidez (`type IN ['checking', 'savings', 'cash']`). Pagar crédito com crédito gera erro 400.
* **REQ-PAG-02 (Pagamentos Parciais):** Pagamentos menores que o montante total devido devem ser aceitos. A fatura recebe o status `PARTIAL` e o sistema rola o saldo remanescente para o mês subsequente.
* **REQ-PAG-03 (Isolamento de Analytics):** Transações geradas pela ação de "Pagamento de Fatura" recebem marcação interna e são ignoradas no cálculo do `total_expense` mensal, evitando contagem dupla (já que o gasto ocorreu na transação original do cartão).

### 3.2 Tabela de Regras Atualizada (MVP)

| REGRA_ID | Descrição | Gatilho | Entrada/Saída | Status / Modificação MVP |
|----------|-----------|---------|---------------|--------------------------|
| **RN001** | Transação parcelada cria série automaticamente | Confirmar item com is_installment=true | `{current, total}` → InstallmentSeries | Mantida sem alterações. |
| **RN002** | Parcelas futuras são projetadas (`is_paid=false`) | create_future_installments | Série → N transações futuras | Mantida sem alterações. |
| **RN003** | Fatura calculada pelo closing_day do cartão | Criar transação com credit_card_id | Data transação → período fatura | Mantida sem alterações. |
| **RN004** | Detecção e Consolidação (Match Exato) | Confirmar transação de documento | Retorna conflito ou atualiza existente | **Atualizada:** Remove tolerâncias e variações. Exige exatidão de centavos para consolidar previsão. |
| **RN005** | Restrição de Pagamento de Fatura | `pay_invoice()` | 400 Bad Request se origem for crédito | Mantida e expandida no REQ-PAG-01. |
| **RN006** | Família (Household) unifica visão de gastos | Listar transações | user.license_id → household_user_ids | Mantida sem alterações. |
| **RN007** | Status fatura: OPEN→CLOSED→PAID/PARTIAL | Pagamento de Fatura | Baseado em fechamento e valor | Mantida e expandida no REQ-PAG-02. |

### 3.3 Casos Extremos e Conflitos Removidos

| Regra Original | Caso Extremo / Conflito | Resolução no Novo Fluxo |
|----------------|-------------------------|-------------------------|
| RN004 (Antiga) | Parcela com variação de centavos lida pelo OCR. | O sistema **não** faz atualização cega. Diferenças bloqueiam a consolidação automática, forçando o usuário a aprovar manualmente. |
| RN012 (Removida)| Tolerância de 5% para agrupar compras. | **Removida do MVP**. Gerava ruído contábil severo e falso-positivos em compras frequentes no mesmo estabelecimento. |
| RN010 (Antiga) | Pagamento maior que a fatura fechada. | A validação do pagamento bloqueia a transação na origem; o valor pago deve obrigatoriamente ser `<= total_due`. |
