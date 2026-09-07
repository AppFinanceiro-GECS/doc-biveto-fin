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
