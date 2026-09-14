# Diagnóstico Técnico e Proposta Inicial de Arquitetura — Biveto-fin

**Sprint:** Sprint 1 — Reformulação e planejamento  
**Escopo:** Diagnóstico do backend e dos dados e proposta inicial de arquitetura para o MVP  

---

## 1. Objetivo e escopo

Este documento apresenta o diagnóstico técnico do backend e da camada de dados atualmente implementados no Biveto-fin e propõe uma arquitetura inicial para a evolução da aplicação em direção ao MVP.

A análise considera quatro fontes, em ordem de autoridade:

1. **código, testes e branch `main`**, que representam o comportamento efetivamente versionado;
2. **ADRs e documentação oficial do repositório**, utilizados como registro das decisões técnicas existentes;
3. **validação do cliente**, utilizada para confirmar prioridades, corrigir interpretações e direcionar o lançamento;
4. **propostas produzidas pelos grupos**, utilizadas como sugestões de evolução enquanto ainda não estiverem incorporadas à `main`.

Essa distinção é importante porque algumas regras apresentadas durante a Sprint 1 ainda não estão versionadas no repositório. Uma proposta de regra ou alteração somente passa a representar a baseline oficial do projeto após ser integrada por pull request.

A proposta arquitetural não busca reescrever o Biveto-fin.

O objetivo é:

- preservar o que já funciona;
- reduzir a superfície do lançamento;
- corrigir problemas que realmente bloqueiam ou aumentam o risco da publicação;
- aumentar cobertura de testes e observabilidade;
- manter a IA isolada das regras financeiras;
- permitir evolução incremental e rastreável.

O princípio de produto permanece válido: o núcleo do Biveto está na experiência em que **a fatura chega e o sistema consegue interpretá-la e organizá-la para o usuário**. O próprio cliente confirmou esse direcionamento como base adequada para o recorte do MVP.

---

## 2. Visão geral do estado atual

O backend do Biveto-fin é implementado em Python utilizando FastAPI e segue uma arquitetura de **monólito modular**.

Na versão analisada foram identificados aproximadamente:

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
| IA | Google Gemini e provider alternativo |
| Testes | pytest |
| Empacotamento | Docker |

O produto possui uma superfície funcional significativamente maior do que o fluxo necessário para o lançamento, incluindo módulos como gamificação, mercado, chat de IA, dívidas, automações e outras funcionalidades.

Entretanto, a validação do cliente mostrou que **complexidade existente não deve ser confundida com funcionalidade quebrada**. Alguns módulos que haviam sido classificados para alteração já possuem comportamento maduro e decisões técnicas registradas.

A estratégia passa, portanto, a ser:

> **reduzir o que é exposto no lançamento sem alterar desnecessariamente o que já funciona.**

---

## 3. Diagnóstico do backend

### 3.1 Estrutura e organização

O backend está organizado por domínios em:

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

O fluxo predominante é:

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

Essa arquitetura de monólito modular continua adequada.

Não se recomenda introduzir microserviços ou uma nova camada `Repository` apenas por padronização arquitetural.

### 3.2 Pontos positivos

O backend já possui uma base madura em vários aspectos:

- separação por domínio;
- API estruturada em FastAPI;
- validação por Pydantic;
- acesso assíncrono ao banco;
- autenticação e refresh de sessão;
- migrations com Alembic;
- suporte a PostgreSQL;
- Docker;
- integração contínua;
- processamento especializado de documentos;
- detecção de documentos duplicados;
- pipeline de extração de faturas desenvolvido e calibrado ao longo do projeto.

A validação do cliente mostrou que a extração de documentos é uma das partes mais trabalhadas do sistema, contendo classificador, prompts por banco, validações, tratamento de diferentes layouts e provider alternativo. Por isso, mudanças nessa área devem ser orientadas por regressão e dados, e não apenas por simplificação.

### 3.3 Pontos de atenção

Os principais riscos técnicos observados passam a ser:

- baixa cobertura automatizada dos fluxos financeiros centrais;
- identificadores de modelos de IA espalhados pelo código;
- necessidade de migrar para identificadores de modelos vigentes sem regredir a extração;
- uso amplo de `print()` no backend em vez de logging estruturado;
- migração de dados executada durante o boot da aplicação;
- armazenamento de documentos no filesystem local;
- processamento em background dependente do processo FastAPI;
- configuração de segurança que precisa ser endurecida para produção.

O scheduler não deve mais ser tratado como funcionalidade experimental.

A validação do cliente confirmou que o APScheduler sobe junto com a API, registra seus jobs e expõe seu estado no `/health`. A limitação relevante é outra: sua arquitetura atual é adequada a uma única réplica da API.

### 3.4 Conclusão do backend

A evolução recomendada passa a ser:

```text
Monólito modular atual
        │
        ▼
Proteger comportamento existente com testes
        │
        ▼
Corrigir bloqueadores técnicos do lançamento
        │
        ▼
Reduzir módulos expostos
        │
        ▼
Publicar MVP
```

A prioridade não é refatorar os maiores arquivos antes do lançamento.

A prioridade é **caracterizar e proteger por testes o comportamento atual antes de modificá-lo**.

---

## 4. Diagnóstico dos dados

### 4.1 Persistência

A aplicação utiliza SQLAlchemy 2 assíncrono e suporta:

```text
SQLite
e
PostgreSQL
```

SQLite permanece adequado ao desenvolvimento e aos testes.

Para produção recomenda-se PostgreSQL.

O domínio é fortemente relacional:

```text
Usuário
   │
   ├── Contas
   │     └── Transações
   │
   ├── Cartões
   │     └── Faturas
   │
   └── Documentos
```

Não foi identificada necessidade de introduzir banco NoSQL.

### 4.2 Modelo de dados

As entidades centrais incluem:

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

As demais tabelas atendem funcionalidades complementares ou futuras.

Funcionalidades ocultadas do MVP não devem ter suas entidades removidas apenas para simplificar numericamente o banco.

Antes de remover código ou schema, é necessário identificar dependências. A validação do cliente mostrou, por exemplo, que `benefit_cards` é utilizado pelo fluxo de recibos e não representa simples duplicação de fontes de renda.

### 4.3 Migrations

Alembic deve continuar sendo a fonte oficial de evolução de schema e dados.

Foi identificada uma dívida relevante: uma migração de dados de cartões é disparada após o startup da API.

Esse comportamento deve ser substituído por uma migration Alembic idempotente executada de forma explícita durante o processo de implantação.

O fluxo recomendado passa a ser:

```text
Deploy
  │
  ▼
Backup
  │
  ▼
alembic upgrade head
  │
  ▼
Startup da aplicação
```

e não:

```text
Startup
  │
  ▼
migração silenciosa em background
```

### 4.4 Armazenamento de documentos e rastreabilidade

Atualmente os arquivos são armazenados em:

```text
./uploads
```

Para desenvolvimento isso continua aceitável.

Para produção recomenda-se abstrair o armazenamento:

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

A rastreabilidade mínima deve permitir seguir:

```text
Documento
    │
    ▼
Extração
    │
    ▼
Revisão
    │
    ▼
Transação
```

Devem permanecer identificáveis:

- documento original;
- hash;
- status;
- modelo/provider utilizado;
- resultado extraído;
- confirmação realizada;
- transações relacionadas.

Não se propõe event sourcing.

O objetivo é conseguir diagnosticar uma falha financeira ou de extração sem depender exclusivamente de logs brutos.

### 4.5 Conclusão dos dados

A estrutura geral permanece adequada:

```text
Desenvolvimento → SQLite
Produção        → PostgreSQL
ORM             → SQLAlchemy
Migrations      → Alembic
Arquivos locais → desenvolvimento
Object Storage  → produção
```

As principais evoluções são:

- mover migrações de dados para Alembic;
- preservar rastreabilidade do fluxo documental;
- evitar migrations destrutivas sem análise de dependência.

---

## 5. Impacto do MVP aprovado sobre a aplicação

### 5.1 Núcleo do lançamento

Após o retorno do cliente, o núcleo de lançamento passa a considerar:

```text
auth
accounts
transactions
categories
documents
installments
credit_cards
analytics
household
notifications
```

Além da infraestrutura de suporte necessária.

`household` passa a ser considerado Parte 1 porque já funciona, foi solicitado pelo cliente e não exige uma nova frente de desenvolvimento relevante.

`notifications` também deixa de ser classificado simplesmente como backlog. O agendador e a geração das notificações já funcionam; o principal débito é a entrega efetiva, especialmente por email.

### 5.2 Módulos pós-entrega

Permanecem adequados para uma etapa posterior:

```text
budgets
goals
recurring
automations
```

Recorrentes continuam podendo ser adiados, mas não porque o scheduler esteja “em validação”.

A justificativa passa a ser validar por testes e homologação que o job gera corretamente as recorrências ao longo de um ciclo real de uso.

### 5.3 Módulos fora da superfície do lançamento

Podem permanecer ocultados:

```text
debts
chat
grocery
gamification
calendar
benefit_cards
simulators
```

com algumas ressalvas:

- `benefit_cards` deve permanecer preservado devido às dependências existentes;
- `receipts` pode ter sua tela ocultada, mas seu backend permanece relevante caso o upload de cupom faça parte do MVP;
- `mcp` não é ferramenta interna: é uma integração opcional destinada ao usuário e apenas fica fora do MVP de loja;
- `calendar` fica fora por prioridade de produto, não por ausência de service.

### 5.4 Matriz de evolução

| Módulo | Fase | Ação recomendada |
|---|---|---|
| `auth` | Lançamento | **Manter e testar** |
| `accounts` | Lançamento | **Manter** |
| `transactions` | Lançamento | **Manter e criar testes de caracterização** |
| `categories` | Lançamento | **Manter** |
| `documents` | Lançamento | **Preservar pipeline e migrar configuração de modelos** |
| `installments` | Lançamento | **Preservar comportamento atual** |
| `credit_cards` | Lançamento | **Manter e aumentar cobertura de testes** |
| `analytics` | Lançamento | **Manter e validar por testes** |
| `household` | Lançamento | **Manter** |
| `notifications` | Lançamento | **Completar entrega por email** |
| `recurring` | Pós-entrega | **Validar funcionalmente antes de expor** |
| `automations` | Pós-entrega | **Ocultar** |
| `receipts` | Suporte/backlog | **Preservar backend; avaliar UI** |
| `benefit_cards` | Backlog | **Ocultar sem remover dependências** |
| `grocery` | Backlog | **Ocultar e preservar** |
| `gamification` | Backlog | **Ocultar** |
| `calendar` | Backlog | **Ocultar por prioridade** |
| `mcp` | Fora do MVP mobile | **Preservar integração** |

Para tornar a estratégia “cortar é adiar” operacional, recomenda-se controlar módulos opcionais do frontend por configuração, em vez de apagar rotas e telas.

Conceitualmente:

```text
VITE_ENABLED_MODULES
```

pode controlar navegação e registro das rotas opcionais, permitindo alterar o recorte do produto sem recuperar código do histórico.

---

## 6. Divergências entre o código atual e as propostas da Sprint

### 6.1 Matching de parcelas

A proposta da Sprint sugeria remover a tolerância e exigir correspondência exata de valor e data.

Após validação do cliente, essa alteração **não é recomendada para o MVP**.

O código atual possui mecanismos diferentes para:

- relacionar uma parcela nova com uma série existente;
- detectar uma transação duplicada.

O fuzzy match de parcelas possui tolerância própria e já é protegido por teste automatizado. Removê-lo sem evidência de erros reais pode quebrar um comportamento já calibrado.

A decisão passa a ser:

> **preservar a lógica atual de matching e somente alterá-la mediante caso real reproduzível e teste que demonstre o problema.**

A revisão manual de itens extraídos continua sendo mantida.

### 6.2 Regras de fatura

As propostas `REQ-PAG-01`, `REQ-PAG-02` e `REQ-PAG-03` não devem ser tratadas como funcionalidades totalmente novas.

A validação do cliente identificou comportamentos equivalentes já implementados e documentados sob a nomenclatura atual do repositório. O principal risco é a ausência de cobertura automatizada para esses fluxos críticos.

A prioridade passa a ser criar testes para:

```text
compra após fechamento
pagamento com origem inválida
pagamento parcial
pagamento total
status da fatura
Analytics sem dupla contagem
```

Antes de alterar essas regras, o comportamento existente deve ser protegido.

### 6.3 Gemini e configuração dos modelos

A extração não deve ser reestruturada apenas para reduzir custo.

O projeto já havia escolhido Gemini Flash após benchmark.

O problema técnico relevante identificado é outro: os nomes dos modelos estão espalhados por diferentes pontos da aplicação e a configuração precisa suportar migração de versões sem alteração em vários módulos.

Recomenda-se centralizar:

```text
vision_model
classifier_model
chat_model
```

e garantir que nenhum service conheça diretamente um identificador fixo de modelo.

A escolha entre modelos deve seguir:

```text
Modelo vigente
      │
      ▼
Teste com fixtures sintéticas
      │
      ▼
Medição de acerto
      │
      ├── latência
      └── custo
      │
      ▼
Decisão
```

Não se deve assumir que extração é a funcionalidade mais cara sem medição. O cliente destacou que o projeto ainda não possui contabilização de custo por funcionalidade.

### 6.4 IA e barreira determinística

Continua válida a separação conceitual entre processamento probabilístico e domínio financeiro:

```text
          IA / extração
               │
               ▼
        Dados extraídos

══════════════════════════════
 Barreira determinística
══════════════════════════════

      validação de schema
      validação dos dados
      detecção de duplicidade
      confirmação do usuário

               │
               ▼
       Domínio financeiro
```

A expressão **barreira determinística** representa um princípio arquitetural proposto neste documento, e não uma nova tecnologia ou camada obrigatória.

A IA pode interpretar e sugerir.

Ela não deve decidir diretamente:

- saldo;
- estado financeiro;
- pagamento;
- regras de fatura;
- efeitos contábeis.

---

## 7. Proposta inicial de arquitetura do MVP

### 7.1 Visão geral

A arquitetura proposta permanece simples:

```text
                        Usuário
                           │
                           ▼
                    React / PWA
                           │
                         HTTPS
                           │
                           ▼
               ┌──────────────────────┐
               │       FastAPI        │
               │   Monólito Modular   │
               │                      │
               │ Auth                 │
               │ Accounts             │
               │ Transactions         │
               │ Documents            │
               │ Installments         │
               │ Cards / Invoices     │
               │ Analytics            │
               │ Household            │
               │ Notifications        │
               └──────────┬───────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
        PostgreSQL   Object Storage   IA
```

Nenhuma nova infraestrutura distribuída é necessária para implementar essa proposta.

### 7.2 Entrada confiável

Para documentos:

```text
Upload
  │
  ▼
Autenticação
  │
  ▼
Validação
  │
  ▼
Hash / duplicidade
  │
  ▼
Persistência
  │
  ▼
Extração
```

O princípio permanece:

> **validar, deduplicar e persistir antes de executar processamento externo.**

### 7.3 Domínio financeiro

O núcleo:

```text
Accounts
Transactions
Invoices
Installments
Analytics
```

permanece determinístico e testável.

Antes de modificar suas regras, devem ser criados testes que descrevam o comportamento existente.

### 7.4 Documentos e IA

O pipeline existente deve ser preservado:

```text
Documento
   │
   ▼
Classificação
   │
   ▼
Extração / OCR
   │
   ▼
Validação
   │
   ▼
Revisão
   │
   ▼
Domínio financeiro
```

A principal mudança arquitetural necessária é **centralizar a configuração dos modelos**, não substituir o pipeline existente.

Faturas utilizadas em testes devem ser sintéticas, nunca documentos financeiros reais.

### 7.5 Processamento e scheduler

O primeiro lançamento pode continuar utilizando:

```text
FastAPI
+
BackgroundTasks
+
status persistido
```

para processamento de documentos.

Não é necessário introduzir Redis/Celery agora.

O APScheduler já faz parte da aplicação e pode continuar sendo utilizado em uma única instância.

Recorrentes e automações podem ser ativados posteriormente sem mudança de arquitetura.

### 7.6 Observabilidade e operação

A observabilidade mínima deixa de ser apenas recomendação futura.

Antes da publicação, recomenda-se substituir o uso de `print()` por logging estruturado e registrar adequadamente:

- nível;
- timestamp;
- contexto da operação;
- erro;
- duração;
- status do processamento.

A validação do cliente identificou logging como uma das dívidas de sustentação diretamente relacionadas à publicação.

Health check e informações sobre scheduler já existentes devem ser preservados.

---

## 8. Decisões técnicas recomendadas

| Área | Decisão |
|---|---|
| Fonte de verdade | **`main` + testes + ADRs antes de propostas ainda não integradas** |
| Arquitetura | **Manter monólito modular** |
| Backend | **Manter FastAPI** |
| ORM | **Manter SQLAlchemy Async** |
| Migrations | **Manter Alembic e retirar migração do boot** |
| Desenvolvimento | **Manter SQLite** |
| Produção | **Utilizar PostgreSQL** |
| Documentos | **Object storage em produção** |
| Pipeline de extração | **Preservar comportamento calibrado** |
| Gemini | **Centralizar identificação dos modelos e validar por regressão** |
| Matching de parcelas | **Preservar fuzzy match até existir evidência para alteração** |
| IA | **Isolar do domínio financeiro** |
| Testes | **Caracterizar regras críticas antes de refatorar** |
| Logging | **Adotar logging estruturado antes da publicação** |
| Scheduler | **Considerar operacional em uma única réplica** |
| Processamento assíncrono | **Manter solução simples inicialmente** |
| Frontend | **Controlar módulos opcionais por configuração** |
| Escala inicial | **Uma instância** |
| Microserviços / Redis / Celery | **Não introduzir no MVP** |

As decisões novas ou alteradas devem ser registradas em ADR quando representarem mudança arquitetural relevante.

Mudança em regra de negócio deve vir acompanhada de teste.

---

## 9. Riscos técnicos e próximos passos

### 9.1 Riscos técnicos

| Risco | Impacto | Prioridade | Tratamento |
|---|---|---:|---|
| Modelos de IA configurados em vários pontos | Falha de extração/manutenção | Alta | Centralizar configuração |
| Núcleo financeiro com baixa cobertura | Regressão silenciosa | Alta | Testes de caracterização |
| Alteração prematura do fuzzy match | Parcelas associadas incorretamente | Alta | Preservar comportamento |
| Uso de `print()` em produção | Diagnóstico difícil | Alta | Logging estruturado |
| Migração de dados no boot | Deploy não determinístico | Alta | Mover para Alembic |
| Arquivos no filesystem local | Persistência/backup frágeis | Alta | Object storage |
| Entrega de notificações incompleta | Funcionalidade parcial | Média/Alta | Completar email |
| Configuração insegura | Risco de sessão/dados | Alta | Segredos obrigatórios e revisão |
| Poucos E2E confiáveis | Falhas chegam ao usuário | Alta | Consolidar cobertura |
| Remoção de módulos com dependências | Regressões indiretas | Média | Ocultar antes de remover |

### 9.2 Próximos passos

A sequência recomendada é:

1. centralizar os identificadores dos modelos de IA;
2. executar regressão da extração com faturas sintéticas;
3. criar testes dos fluxos de fatura, pagamento, transações e Analytics;
4. preservar o fuzzy match até existir evidência para mudança;
5. completar a entrega de notificações por email;
6. substituir `print()` por logging estruturado;
7. mover a migração executada no boot para Alembic;
8. implementar configuração de módulos habilitados no frontend;
9. consolidar os cenários E2E da Parte 1;
10. implementar object storage para produção;
11. endurecer configuração de segurança;
12. implementar exclusão de conta e revisar política de privacidade;
13. validar a PWA existente em dispositivos reais;
14. preparar publicação Android pelo caminho definido com o cliente.

A PWA já existe no projeto; portanto, o trabalho é de verificação e distribuição, e não de construção do zero.

Para publicação, o retorno do cliente recomenda tratar Google Play e App Store como esforços diferentes: Google Play pode compor a primeira entrega, enquanto App Store deve ser planejada posteriormente devido ao maior esforço de empacotamento e requisitos nativos.

Fila distribuída, múltiplos workers e tracing completo permanecem como evoluções futuras.

---

## 10. Conclusão

O Biveto-fin já possui uma base técnica capaz de suportar o MVP.

Não se recomenda reconstruir o backend nem substituir suas principais tecnologias.

Devem ser preservados:

```text
FastAPI
SQLAlchemy
Alembic
PostgreSQL
Docker
Monólito modular
Pipeline de extração existente
```

Após a validação do cliente, o foco técnico passa a ser menos de **alterar regras existentes** e mais de **tornar seguro o lançamento daquilo que já funciona**.

As prioridades são:

```text
comportamento existente
        │
        ▼
testes de caracterização
        │
        ▼
correção dos bloqueadores técnicos
        │
        ▼
redução da superfície do produto
        │
        ▼
publicação
```

A arquitetura proposta continua:

```text
React / PWA
     │
     ▼
FastAPI
     │
     ├── PostgreSQL
     ├── Object Storage
     └── Serviço de IA
```

e continua não exigindo, neste momento:

```text
microserviços
Kubernetes
Redis
Celery
fila distribuída
workers independentes
múltiplas instâncias
```

O fluxo de IA permanece protegido por uma separação determinística:

```text
Documento
   │
   ▼
Extração probabilística
   │
   ▼
Validação + revisão
   │
   ▼
Domínio financeiro determinístico
```

A principal mudança de perspectiva é que a evolução do projeto deve partir daquilo que está **versionado, funcionando e protegido por testes**, e não apenas daquilo que foi proposto documentalmente.

Para a disciplina de **Gerência de Configuração e Evolução de Software**, esse princípio é central:

> **o comportamento versionado forma a baseline; propostas alteram essa baseline somente através de mudanças rastreáveis, revisadas e testadas.**

Assim, o MVP pode ser reduzido e estabilizado sem descartar trabalho já realizado e sem introduzir complexidade arquitetural desnecessária.
