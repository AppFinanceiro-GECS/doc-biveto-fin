# Diagnóstico Técnico e Proposta Inicial de Arquitetura — Biveto-fin

## 1. Objetivo e escopo

Este documento apresenta o diagnóstico técnico do backend e da camada de dados atualmente implementados no Biveto-fin e propõe uma arquitetura inicial para a evolução da aplicação em direção ao MVP definido na Sprint 1.

A análise considera três fontes:

1. o **código-fonte atual**, utilizado como referência do estado técnico efetivamente implementado;
2. as **regras de negócio consolidadas pelo Grupo 3**, utilizadas como referência para o comportamento esperado dos fluxos financeiros;
3. o **escopo priorizado pelo Grupo 4**, utilizado para determinar quais funcionalidades pertencem ao lançamento e quais permanecem para etapas posteriores.

A proposta não busca reescrever a aplicação nem introduzir uma arquitetura excessivamente sofisticada.

O objetivo é:

- preservar componentes tecnicamente adequados;
- corrigir divergências entre código e regras aprovadas;
- reduzir a superfície funcional;
- melhorar a confiabilidade do processamento de documentos;
- manter a inteligência artificial isolada das regras financeiras;
- preservar rastreabilidade suficiente para diagnosticar falhas.

O princípio central do MVP é manter apenas aquilo que sustenta diretamente a principal proposta de valor do Biveto: receber uma fatura, interpretar seus dados e organizá-los financeiramente com pouca intervenção manual.

---

## 2. Visão geral do estado atual

O backend do Biveto-fin é implementado em Python utilizando FastAPI e segue uma arquitetura de **monólito modular**.

A versão analisada possui aproximadamente:

| Métrica | Valor observado |
|---|---:|
| Arquivos Python em `backend/app` | 292 |
| Linhas Python em `backend/app` | 54.386 |
| Módulos funcionais em `backend/app/modules` | 28 |
| Rotas HTTP declaradas | aproximadamente 250 |
| Tabelas SQLAlchemy mapeadas | 52 |
| Migrations Alembic | 36 |

As principais tecnologias encontradas são:

| Camada | Tecnologia |
|---|---|
| Backend | FastAPI |
| ORM | SQLAlchemy 2 assíncrono |
| Validação | Pydantic |
| Migrations | Alembic |
| Desenvolvimento | SQLite + aiosqlite |
| Produção suportada | PostgreSQL + asyncpg |
| Autenticação | JWT |
| Jobs agendados | APScheduler |
| Rate limiting | slowapi |
| IA | Google Gemini e providers alternativos |
| Testes | pytest |
| Empacotamento | Docker |

O sistema atual possui funcionalidades que ultrapassam o escopo necessário ao lançamento, como assistente de IA, gamificação, lista de mercado, dívidas, notificações, automações e outras expansões. O próprio documento do Grupo 4 identifica esse crescimento como uma das razões para reduzir o produto ao fluxo principal.

A conclusão inicial é que o principal problema não está na tecnologia utilizada, mas na **quantidade de responsabilidades acumuladas ao redor do núcleo financeiro**.

---

## 3. Diagnóstico do backend

### 3.1 Estrutura e organização

O backend é dividido em módulos de domínio dentro de:

```text
backend/app/modules/
```

Entre os módulos existentes estão:

```text
accounts
admin
analytics
api_keys
auth
automations
benefit_cards
budgets
calendar
categories
chat
credit_cards
debts
documents
gamification
goals
grocery
household
income
income_splits
installments
known_services
mcp
notifications
receipts
recurring
review
transactions
```

O fluxo predominante das requisições é:

```text
Request HTTP
    │
    ▼
Router FastAPI
    │
    ▼
Schema Pydantic
    │
    ▼
Service
    │
    ▼
SQLAlchemy / AsyncSession
    │
    ▼
Banco de dados
```

A aplicação possui uma única instância FastAPI com vários módulos internos, caracterizando um monólito modular.

Essa arquitetura continua adequada ao MVP.

Não foi identificada necessidade de migração para microserviços ou de criação imediata de uma nova camada de `Repository`.

### 3.2 Pontos positivos

Foram identificados os seguintes aspectos positivos:

- organização por domínio;
- utilização de FastAPI;
- validação estruturada com Pydantic;
- acesso assíncrono ao banco;
- autenticação já implementada;
- migrations versionadas;
- suporte existente a PostgreSQL;
- empacotamento Docker;
- pipeline de integração contínua;
- serviços específicos para processamento de documentos e IA;
- possibilidade de reaproveitamento significativo da implementação atual.

Um aspecto especialmente importante é que o fluxo de documentos já possui elementos úteis para confiabilidade, como:

- identificação do documento;
- hash;
- detecção de duplicidade;
- status de processamento;
- armazenamento de resultados de extração.

Esses mecanismos podem ser fortalecidos sem introduzir nova infraestrutura.

### 3.3 Pontos de atenção

O principal ponto de atenção é a complexidade funcional.

A Parte 1 definida para o lançamento concentra-se em:

- autenticação;
- transação manual;
- upload e extração de fatura;
- parcelamento;
- cálculo e pagamento de fatura;
- resumo mensal;
- publicação.

Funcionalidades dependentes de scheduler, como recorrentes, notificações e automações, não devem compor o caminho crítico da primeira publicação.

Também existem pontos técnicos que precisam ser adequados:

- armazenamento de documentos no disco local;
- algumas regras financeiras antigas ainda presentes;
- configuração de IA parcialmente acoplada ao código;
- rastreabilidade operacional que pode ser melhorada;
- baixa cobertura dos fluxos financeiros mais críticos.

### 3.4 Conclusão do backend

Não se recomenda substituir o FastAPI nem transformar o sistema em uma arquitetura distribuída.

A evolução recomendada é:

```text
Monólito modular atual
        │
        ▼
Redução da superfície funcional
        │
        ▼
Correção das regras divergentes
        │
        ▼
Fortalecimento do fluxo de documentos/IA
        │
        ▼
Monólito modular focado no MVP
```

---

## 4. Diagnóstico dos dados

### 4.1 Persistência

A aplicação utiliza SQLAlchemy 2 de forma assíncrona e suporta:

```text
SQLite
e
PostgreSQL
```

SQLite pode continuar sendo utilizado para desenvolvimento local.

Para produção recomenda-se PostgreSQL.

O domínio possui forte relacionamento entre:

```text
Usuário
   │
   ├── Contas
   │     └── Transações
   │
   ├── Cartões
   │     └── Faturas
   │           └── Transações
   │
   └── Documentos
```

Por isso, o banco relacional atual é adequado.

Não foi identificada necessidade de banco NoSQL.

### 4.2 Modelo de dados

A versão analisada possui 52 tabelas SQLAlchemy.

As principais entidades relacionadas ao lançamento são:

```text
users
licenses
invitations

accounts

transactions
transaction_payments
transaction_audit
categories
merchants

credit_cards
credit_card_invoices

installment_series

documents
document_extractions
```

As demais entidades acompanham funcionalidades pós-entrega ou de backlog.

O Grupo 4 adotou como princípio que retirar algo do MVP significa **adiar**, e não necessariamente apagar imediatamente código ou dados existentes.

Portanto, não se recomenda realizar grandes migrations destrutivas apenas para reduzir o número de tabelas.

### 4.3 Migrations

O projeto utiliza Alembic e possui histórico versionado de alterações de schema.

Esse mecanismo deve ser preservado.

Toda alteração estrutural necessária para implementar as novas regras do MVP deve continuar sendo registrada através de migration.

### 4.4 Armazenamento de documentos e rastreabilidade

Atualmente os documentos são armazenados localmente em:

```text
./uploads
```

enquanto o banco guarda metadados e informações sobre o processamento.

Para desenvolvimento, essa solução continua aceitável.

Para produção, recomenda-se:

```text
DocumentService
      │
      ▼
StorageService
      │
      ├── Local Storage
      │      └── desenvolvimento
      │
      └── Object Storage
             └── produção
```

Além da mudança de armazenamento, recomenda-se preservar uma cadeia de rastreabilidade mínima:

```text
Documento recebido
      │
      ▼
Extração realizada
      │
      ▼
Resultado apresentado
      │
      ▼
Correção/confirmação do usuário
      │
      ▼
Transação criada
```

O objetivo não é implementar event sourcing.

O objetivo é conseguir responder, quando necessário:

> **O que aconteceu com este documento e quais transações foram geradas a partir dele?**

Idealmente, devem permanecer rastreáveis:

- documento original;
- hash;
- status;
- provider/modelo utilizado;
- resultado extraído;
- confirmação realizada;
- transações relacionadas.

### 4.5 Conclusão dos dados

A estrutura de persistência pode ser mantida:

```text
Desenvolvimento
→ SQLite

Produção
→ PostgreSQL

ORM
→ SQLAlchemy

Migrations
→ Alembic

Documentos locais
→ desenvolvimento

Object Storage
→ produção
```

A principal evolução proposta é reforçar a **rastreabilidade entre documento, extração e efeito financeiro**.

---

## 5. Impacto do MVP aprovado sobre a aplicação

### 5.1 Núcleo do lançamento

A Parte 1 definida pelo Grupo 4 contém:

- autenticação e convite;
- transação manual;
- upload e extração;
- parcelamento;
- cálculo e pagamento de fatura;
- resumo mensal;
- publicação.

Isso concentra o lançamento principalmente nos módulos:

```text
auth
admin
accounts
transactions
categories
documents
installments
credit_cards
analytics
```

### 5.2 Módulos pós-entrega

Foram definidos para etapa posterior:

```text
budgets
household
recurring
goals
```

Esses módulos podem permanecer no repositório sem se tornar dependências do lançamento.

### 5.3 Módulos fora do lançamento

Funcionalidades como:

```text
debts
chat
notifications
automations
grocery
gamification
benefit_cards
receipts
calendar
mcp
```

não pertencem ao núcleo inicial ou foram explicitamente classificadas como backlog.

### 5.4 Matriz de evolução

| Módulo | Fase | Situação | Ação |
|---|---|---|---|
| `auth` | Lançamento | Base adequada | **Manter** |
| `admin` | Lançamento/suporte | Convites/licenças | **Manter escopo mínimo** |
| `accounts` | Lançamento | Liquidez precisa adequação | **Alterar** |
| `transactions` | Lançamento | Núcleo financeiro | **Manter e adequar regras** |
| `categories` | Lançamento/suporte | Utilizado por transações | **Manter** |
| `documents` | Lançamento | IA, storage e rastreabilidade | **Alterar** |
| `installments` | Lançamento | Possui tolerâncias antigas | **Alterar** |
| `credit_cards` | Lançamento | Novas regras de pagamento | **Alterar** |
| `analytics` | Lançamento | Isolamento de pagamento | **Alterar** |
| `household` | Pós-entrega | Funcionalidade válida | **Preservar** |
| `budgets` | Pós-entrega | Escopo simplificado | **Preservar** |
| `goals` | Pós-entrega | Meta simplificada | **Preservar** |
| `recurring` | Pós-entrega | Depende de scheduler | **Adiar** |
| Demais módulos de backlog | Futuro | Fora do lançamento | **Não expor** |

---

## 6. Divergências entre o código atual e as regras do MVP

### 6.1 Match exato

A regra consolidada pelo Grupo 3 remove a antiga tolerância utilizada na conciliação.

A consolidação automática passa a exigir:

```text
valor exato
+
data exata
```

Divergências devem exigir revisão manual.

A implementação atual ainda possui regras de tolerância no fluxo de parcelas.

Portanto:

```text
installments
→ manter funcionalidade
→ alterar implementação
```

### 6.2 Liquidez no pagamento de fatura

REQ-PAG-01 restringe o pagamento de fatura a contas de liquidez.

O conceito precisa ser formalizado no código.

A validação não deve ser simplesmente:

```text
não é cartão de crédito?
```

Ela deve responder:

```text
esta conta é uma origem de liquidez permitida?
```

### 6.3 Pagamento parcial

REQ-PAG-02 determina:

```text
Pagamento menor que total
        │
        ▼
status PARTIAL
        │
        ▼
saldo remanescente
        │
        ▼
fatura seguinte
```

A estrutura atual possui parte do suporte necessário, mas o fluxo completo deve ser validado e completado.

### 6.4 Analytics

REQ-PAG-03 determina que a transação de pagamento da fatura não seja contabilizada novamente como despesa.

Recomenda-se que esse tipo de transação possua identificação explícita de origem, evitando depender apenas da reconstrução através da entidade de fatura.

### 6.5 IA e barreira determinística

O Grupo 4 mantém a extração por IA no MVP, mas exige revisão do usuário antes da criação das transações.

Esse comportamento deve ser formalizado arquiteturalmente através de uma **barreira determinística** entre a IA e o domínio financeiro:

```text
           ÁREA PROBABILÍSTICA

               Serviço de IA
                    │
                    ▼
             Dados extraídos

════════════════════════════════════
        BARREIRA DETERMINÍSTICA
════════════════════════════════════

             validação de schema
             validação de valores
             detecção de duplicidade
             match exato
             revisão do usuário

                    │
                    ▼

           DOMÍNIO FINANCEIRO
```

A IA pode:

- interpretar documentos;
- identificar valores;
- identificar datas;
- identificar estabelecimentos;
- sugerir parcelas ou categorias.

A IA não deve:

- calcular saldo;
- decidir liquidez;
- determinar pagamento de fatura;
- executar rollover;
- decidir regras do Analytics;
- criar diretamente efeitos financeiros sem validação.

Isso mantém o único componente probabilístico isolado das regras determinísticas.