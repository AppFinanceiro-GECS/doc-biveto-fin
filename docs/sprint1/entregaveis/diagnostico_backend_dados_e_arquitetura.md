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