# MVP do Biveto

## Sumário

1. [Resumo do produto](#1-resumo-do-produto)
2. [Princípios do MVP](#2-princípios-do-mvp)
3. [Requisitos atuais](#3-o-que-já-existe-e-o-que-fazer-com-cada-parte)
4. [Módulos fora do MVP](#4-módulos-fora-do-mvp-ocultar-não-apagar)
5. [Requisitos novos](#5-requisitos-novos)
6. [Partes de entrega](#6-partes-de-entrega)
7. [Critérios de aceite](#7-critérios-de-aceite)
8. [Dívidas técnicas no caminho do lançamento](#8-dívidas-técnicas-no-caminho-do-lançamento)
9. [Proposta de trabalho](#9-proposta-de-trabalho)
10. [Onde não mexer neste ciclo](#10-onde-não-mexer-neste-ciclo)
11. [Como trabalhar](#11-como-trabalhar)

---

## 1. Resumo do produto

O Biveto é um app de finanças pessoais para o público brasileiro. O usuário sobe a fatura do cartão (foto ou PDF), a IA lê e separa cada compra — inclusive parceladas e assinaturas recorrentes — e o app organiza tudo em contas, cartões, faturas e um resumo mensal de gastos. Existe também suporte a uso em família, com contas e transações compartilhadas.

O código vai além disso: 29 módulos no backend e 39 rotas no frontend (`frontend/src/App.tsx`), incluindo metas, dívidas, orçamento, assistente de IA, gamificação, lista de mercado e um servidor MCP. Parte desses módulos está fora do fluxo principal, e é aí que o MVP corta.

---

## 2. Princípios do MVP

**01 · Medir o custo de IA antes de cortar**
O padrão de extração já é Gemini Flash desde 4/4/2026 (commit `ef63404`, registrado como ADR-004), e o classificador de documentos já usa Flash-Lite. Não existe medição de custo por funcionalidade no projeto: não há contagem de tokens por upload nem por conversa do assistente. Portanto "extração é a parte mais cara" é hipótese, não dado. O assistente (`financial_agent_service.py`, 3.540 linhas) faz várias chamadas ao mesmo modelo por conversa e é o candidato mais provável a maior custo por usuário — e ele já sai do MVP na Parte 3, o que provavelmente reduz mais custo do que qualquer troca de modelo.

**02 · Só entra o que sustenta a promessa central**
"A fatura chegou, o app já entende." A promessa inclui o que acontece **depois** da leitura: ser lembrado do vencimento. Por isso notificação de vencimento entra na Parte 1 (ver princípio 03), e não sai como "extra".

**03 · Nenhuma tela promete o que o backend não cumpre — e o caminho barato costuma ser acabar, não esconder**
O agendador roda (ver seção 1). O que falta em notificações é o envio por email, que é um job do APScheduler sobre uma tabela que já está populada. Esconder a tela custa mais (mexer em rotas, explicar ao usuário) do que terminar a entrega. O princípio continua valendo para o que realmente não existe.

**04 · Publicável em loja é prioridade de lançamento — uma loja de cada vez**
Google Play aceita o PWA empacotado como Trusted Web Activity (Bubblewrap): taxa única de US$ 25, sem reescrever nada. App Store **não** aceita site empacotado — a diretriz 4.2 rejeita app que é só um site numa casca, exigindo projeto Capacitor com recurso nativo real, Mac para compilar e conta a US$ 99/ano. São esforços de ordens de grandeza diferentes: Play no MVP, App Store na Parte 2.

**05 · Cortar é adiar — e para isso precisa existir o mecanismo**
O frontend tem 39 rotas em `src/App.tsx` e **nenhum mecanismo de flag de funcionalidade**. Hoje "ocultar" só é possível editando rotas e menu na mão, ou seja: ocultar vira apagar, e apagar não é adiar. A primeira tarefa de frontend do MVP é criar esse mecanismo (item 4 da proposta de trabalho).

---

## 3. O que já existe, e o que fazer com cada parte

| Requisito | Módulo | Regra (`main`) | Decisão | Motivo e fonte |
|---|---|---|---|---|
| Autenticar usuário | auth | RN013 | **manter** | Registro por convite é a ADR-005. JWT com refresh já existe e o interceptor em `src/services/api.ts` renova a sessão. Sem trabalho novo além de teste. |
| Registrar transação | transactions | a confirmar | **manter** | Núcleo do produto. Ressalva: `transaction_service.py` tem 1.995 linhas e **nenhum** teste automatizado (dívida #2). Manter sem testar é manter risco. |
| Extrair fatura via IA | documents | a confirmar | **manter** | Corrigido: a v1 dizia "alterar" para reduzir custo. É a parte mais trabalhada do sistema — 76 commits entre jan e ago/2026, classificador de tipo de documento, prompts versionados por banco (`prompts/banks/`), roteamento do Itaú por PDF nativo (`12a80e9`), validador de soma, provider alternativo (Mistral OCR). O padrão já é Flash (ADR-004). Único trabalho legítimo aqui é a migração de identificador de modelo — item 1 da proposta. |
| Confirmar itens extraídos | transactions | RN005 | **manter** | Confirmação manual antes de virar transação: correto e mantido. **A remoção da tolerância de 5% não procede** — ver nota abaixo da tabela. |
| Gerenciar parcelamento | installments | RN005, RN012 | **manter** | Regra madura e protegida por `tests/test_installment_fuzzy_match.py`. |
| Calcular período da fatura | invoices | RN003 | **manter** | Compra após o fechamento cai na fatura seguinte. Sem teste hoje (dívidas #2 e #3). |
| Pagar fatura | invoices | RN006, RN010 | **manter** | Liquidez restrita e status parcial já implementados. A lógica vive em `credit_cards/routers/invoices.py`, 1.243 linhas dentro de um router (dívida #3), sem um teste sequer. |
| Compartilhar com a família | household | a confirmar | **manter** | Corrigido: a v1 se contradizia (tabela dizia "manter", Parte 2 dizia "logo em seguida"). Como já funciona, fica na **Parte 1 sem trabalho novo**. |
| Ver resumo mensal | analytics | RN008 | **manter** | Pagamento de fatura já é excluído do total de despesas do mês. |
| Gerar transação recorrente | recurring | a confirmar | **manter, entrega na Parte 2** | Corrigido: não há o que "validar no agendador". O job está registrado e ativo (plano de ação 3.5, concluído em março) e o bug que o impedia de rodar foi corrigido em agosto (`fa813cc`). A limitação real está na ADR-003: só funciona com uma réplica da API. O que falta é confirmar que o job gera as recorrentes corretas em um ciclo de uso real — teste do job + homologação rodando um mês. |
| Notificar vencimento | notifications | a confirmar | **manter e terminar** | O agendador já gera vencimento de fatura, orçamento estourado e fatura que não chegou (`alert_engine`). Falta a entrega por email (dívida #7) — item 3 da proposta. |

---

## 4. Módulos fora do MVP (ocultar, não apagar)

| Módulo | Decisão | Motivo e fonte |
|---|---|---|
| grocery | **ocultar** | Cinco tabelas (`backend/app/models/grocery.py`), **Não apagar:** o requisito futuro N07 é exatamente este módulo, e a lista inteligente já existe em `grocery/services/smart_list_service.py`. Já levantado no PLANO_DE_ACAO, item 4.1. |
| gamification | **ocultar** | Sem dado de engajamento não há como saber se vale manter (PLANO_DE_ACAO, item 4.2). |
| simulators | **ocultar** | Sem persistência e fora da promessa central. |
| calendar | **ocultar** | Corrigido: `calendar/services/cash_calendar_service.py` existe, com visões diária, semanal e mensal, saldo corrente e projeção. |
| benefit_cards | **ocultar só no frontend** | Corrigido: a conta de VR/VA com operadora, dia de recarga e valor esperado (`app/models/benefit_card.py`), e o campo `linked_income_source_id` **liga** a uma renda, não a copia. **Remover o backend quebra recibos:** `receipts/services/receipt_service.py` importa esse modelo para decidir de que conta saiu um cupom. |
| receipts | **ocultar a tela, manter o backend** | Corrigido: o módulo não "divide recibo" — ele confirma um cupom fiscal extraído e o transforma em transação com itens. Como o critério de aceite do MVP aceita cupom, **o backend continua necessário**. Unificar com `documents` é trabalho da Parte 2 (PLANO_DE_ACAO, item 4.3). |
| automations | **ocultar** | Regras "se X então Y" definidas pelo usuário: mais complexas e menos essenciais. Parte 2. |
| mcp | **fora do app, sem remoção** | Corrigido: não é ferramenta interna de desenvolvimento. O servidor em `mcp-biveto-db/` consome a API pública com a chave `biv_` do próprio usuário (ADR-006) — é o usuário consultando as próprias finanças pelo Claude sem que o Biveto pague a conversa. Fica como está, sem tela no app. |

Ocultar significa **não registrar a rota** pelo mecanismo de módulos habilitados (item 4 da proposta), nunca apagar código.

---

## 5. Requisitos novos

| # | Requisito | Descrição corrigida | Prioridade |
|---|---|---|---|
| N01 | Publicar no Google Play | Empacotar o PWA como Trusted Web Activity (Bubblewrap), taxa única de US$ 25, exige HTTPS e `assetlinks.json` publicado no domínio. | **MVP** |
| N01b | Publicar na App Store | Exige projeto Capacitor com recurso nativo real (câmera, push ou biometria), conta a US$ 99/ano, Mac para compilar e conta de demonstração para a revisão (diretrizes 4.2 e 2.1). Site empacotado é rejeitado. | Parte 2 |
| N02 | **Medir** custo de IA por funcionalidade | Corrigido: a v1 pedia "padronizar num modelo mais barato", mas o padrão já é Flash desde abril (ADR-004) e não existe nenhuma medição no projeto. O requisito é registrar tokens de entrada e saída por chamada e só então decidir. | **MVP** |
| N03 | Verificar o PWA em aparelho real | Corrigido: **não é requisito novo.** Já existem `frontend/public/manifest.json` (ícones em 8 tamanhos, `display: standalone`, pt-BR), `sw.js` registrado no `index.html`, screenshots e as páginas `privacy-policy.html` e `support.html` que as lojas exigem. A tarefa é instalar num Android e num iPhone e registrar o que falha (cache do service worker em atualização, por exemplo). Uma tarde, não uma entrega. | **MVP** |
| N04 | Simplificar hospedagem | O deploy já é um container único que serve backend e frontend na mesma origem (ADR-007). O que falta: workflow de CI que constrói a imagem e publica num registro, e `alembic upgrade head` documentado como passo do deploy. Trocar de domínio exige refazer o `assetlinks.json` da TWA e o `id` do manifest. | **MVP** |
| N05 | Definir nova identidade | Nome e domínio próprios. Decisão do cliente, não do time técnico. | **MVP** |
| N06 | Habilitar 12 testadores no Google Play | Corrigido em três pontos: é **teste fechado** (closed testing), não interno; a regra só vale para **conta pessoal criada após 13/11/2023** (conta de organização/CNPJ está isenta e publica direto — isso decide em nome de quem a conta é aberta); e são 12 testadores com **opt-in contínuo por 14 dias**, com uso real verificado desde 2026. Doze convites sem uso não contam. | **MVP** |
| N07 | Sugerir lista de compras | Já existe em `grocery/services/smart_list_service.py` — é o módulo que o MVP oculta. Registrado para não ser reinventado. | futuro |
| N08 | Conectar Open Finance | Importação bancária automática; integração regulatória, fora de um primeiro ciclo. | futuro |
| N09 | Excluir a própria conta no app | **Novo e bloqueante para publicar.** Hoje só o administrador apaga um usuário (`DELETE /admin/users/{id}`). Apple exige exclusão de conta dentro do app desde 2022, Google desde 2023, e a LGPD exige canal para exportação e exclusão. Sem isso, nenhum recorte de funcionalidade libera a loja. | **MVP** |

---

## 6. Partes de entrega

### Parte 1 — Lançamento

- Autenticação e convite
- Contas e cartões
- Transação manual
- Upload e extração de fatura
- Parcelamento
- Cálculo e pagamento de fatura
- Resumo mensal
- **Compartilhamento familiar** (já funciona, sem trabalho novo — subiu da Parte 2)
- **Notificação de vencimento por email** (subiu da remoção: falta só a entrega)
- **Exclusão de conta pelo usuário** (bloqueante de loja)
- Publicação no Google Play via TWA

### Parte 2 — Pós-entrega imediata

- Orçamento mensal
- Metas financeiras
- Recorrentes (após um ciclo confirmado em homologação)
- App Store via Capacitor
- Automações
- Unificar recibos e documentos
- Web Push

### Parte 3 — Backlog pós-MVP

- Dívidas com estratégias
- Assistente de IA
- Lista de compras inteligente (N07)
- Open Finance (N08)
- Gamificação, cartão-benefício (tela), simuladores, calendário

---

## 7. Critérios de aceite

| Critério | Estado no código | O que falta |
|---|---|---|
| Só entra por convite | Implementado (RN013) | Já tem teste em `test_auth.py` |
| Sessão renova sem derrubar | Implementado (refresh de 7 dias) | Teste E2E do refresh |
| Aceita PDF e foto | Implementado (imagens, PDF, HEIC, planilhas) | Nada |
| Revisão antes de virar transação | Implementado (fluxo de confirmação) | Teste do fluxo |
| Reenvio detectado como duplicata | Implementado por SHA-256 do arquivo (`document_service.py:384`) | Teste |
| Compra após fechamento cai na fatura seguinte | Implementado (RN003) | Teste — hoje inexistente |
| Pagamento não aceita cartão como origem | Implementado (RN006) | Teste — hoje inexistente |
| Status da fatura em tempo real | Implementado (RN010) | Teste — hoje inexistente |
| Parcela projetada atualizada, nunca duplicada | Implementado (RN005 + fuzzy match) | Já tem teste; é o que a remoção da tolerância quebraria |
| Instalável como PWA | Implementado | Verificação em aparelho + TWA |

**Retirado do MVP:** "modelo de IA padrão é o de menor custo para o tipo de arquivo enviado" — não existe roteamento por custo no código, e criar um depende da medição do N02.

O padrão a reparar: o núcleo que queremos lançar (fatura, pagamento, status) funciona e **não tem teste**. Escrever esses testes é a melhor porta de entrada no código para o time, porque força a ler o fluxo real e produz algo que protege o lançamento.

---

## 8. Dívidas técnicas no caminho do lançamento

| # | Dívida | Por que bloqueia o lançamento |
|---|---|---|
| #2 | Sem cobertura de testes nos módulos centrais | `transactions`, `credit_cards` (faturas) e `budgets` não têm um teste sequer — são exatamente a Parte 1. Qualquer mudança, inclusive "ocultar", pode quebrar algo sem ninguém perceber antes do usuário. |
| #7 | Entrega de notificações não implementada | Está a um job de distância de funcionar. Ver item 3 da proposta. |
| #5 | Backend registra log com `print()` | Num app de loja, quando o usuário disser "minha fatura deu erro", a única pista é o stdout do container: sem nível, sem timestamp estruturado, sem identificação de usuário. |
| #4 | Migração de dados dentro do `main.py` | `migrate_credit_card_transactions` roda 30 s depois de **todo** boot e falha em silêncio. Pertence a uma migração Alembic executada uma vez. |
| #9 | Suítes E2E duplicadas e sem seletores estáveis | Duas suítes em paralelo, com seletores por texto em português que quebram a cada mudança de copy. Os critérios de aceite da seção 7 só são verificáveis com E2E confiável. |

**Segurança e conformidade:**

- `SECRET_KEY` forte é obrigatória em ambiente publicado: além de assinar os JWTs, ela deriva a chave que criptografa as senhas de PDF dos usuários (`app/core/crypto.py`).
- Exclusão de conta pelo próprio usuário não existe (N09) e bloqueia as duas lojas.
- Política de privacidade já publicada em `public/privacy-policy.html`, mas precisa de revisão antes do envio.
- **Fatura real jamais entra no repositório**, nem como fixture de teste. O gitleaks bloqueia segredos, mas não reconhece uma fatura.

---

## 9. Proposta de trabalho

### 1 · Migrar o identificador do modelo Gemini
*Backend · 1 pessoa · pré-requisito de tudo que envolve upload*

**Por quê:** o modelo padrão do código foi desligado pelo Google. O identificador usado é `gemini-2.0-flash`, e toda a família 2.0 saiu do ar em 1/6/2026. Chamadas de teste com a chave do projeto em 9/9/2026: `gemini-2.0-flash` → **HTTP 404**; `gemini-2.0-flash-lite` → **HTTP 404**; `gemini-2.5-flash` → 200; `gemini-2.5-flash-lite` → 200. Qualquer ambiente que suba com a configuração padrão recebe 404 em todo upload. O `gemini-2.5-flash` já tem desligamento anunciado para **16/10/2026** — trocar um identificador por outro resolve por cinco semanas.

**O que fazer:**
1. Substituir todas as ocorrências literais de `gemini-2.0-*` por leitura de configuração. Hoje o identificador está escrito em cerca de doze pontos: `config.py:40`, fallbacks em `document_service.py`, `llm_ocr_service.py`, `google_provider.py`, o classificador em `document_classifier.py:390` (que nem lê variável de ambiente), o assistente em `financial_agent_service.py` (linhas 201 e 3852) e a lista de mercado.
2. Criar em `config.py` três campos: `vision_model` (extração), `classifier_model` (hoje Flash-Lite) e `chat_model`. Nada mais no código conhece nome de modelo.
3. Padrão inicial: `gemini-2.5-flash-lite` para classificador, `gemini-2.5-flash` para extração e chat. Avaliar os aliases `gemini-flash-latest` e `gemini-flash-lite-latest`, que o Google aponta para a versão vigente — registrar a escolha e o trade-off (alias muda sem aviso) na **ADR-011**.
4. Montar faturas **sintéticas** em `backend/tests/fixtures/`: uma por banco com prompt (Itaú em duas colunas, Nubank, Bradesco) mais um cupom fiscal, com o JSON esperado ao lado. Dados inventados; jamais fatura real.
5. Teste marcado `@pytest.mark.live` (fora do CI, roda na máquina com chave) que extrai cada fixture e compara itens, valores e total.
6. Rodar o mesmo teste com Flash e Flash-Lite e registrar acerto e tempo. **Só então** decidir o padrão de extração, com número na mão — é isso que a v1 chamava de N02.

**Pronto quando:** `grep -rn "gemini-" backend/app` só encontra `config.py`; o teste live passa nas quatro fixtures; a ADR-011 está na pasta de onboarding com a tabela de acerto por modelo.

### 2 · Testes de característica do fluxo de fatura
*Backend · 2 pessoas · em paralelo ao item 1*

**Por quê:** o núcleo da Parte 1 não tem teste. Os critérios de aceite da seção 7 já são os casos — basta escrevê-los.

**O que fazer:** `tests/test_invoices.py` com, no mínimo: compra depois do fechamento cai na fatura seguinte; vencimento anterior ao fechamento vai para o mês seguinte; pagar com conta do tipo cartão devolve 400; pagamento parcial deixa status `partial` e rola o restante; pagamento total deixa `paid`. E `tests/test_transactions.py` com: duplicata por descrição, valor e data devolve 409; confirmação de item extraído cria a transação com a categoria sugerida; analytics do mês exclui o pagamento de fatura. Usar o `conftest.py` existente (SQLite em memória por teste, client HTTP pronto) — não precisa de infra nova.

**Pronto quando:** cada critério de aceite de "fatura" e "parcelamento" tem um teste com o nome do critério, e o CI roda todos.

### 3 · Entregar notificações por email
*Backend · 1 pessoa*

**Por quê:** transforma "tela que promete e não cumpre" em funcionalidade, com menos esforço do que escondê-la e explicar por quê.

**O que fazer:** um job em `app/core/scheduler.py` que, a cada N minutos, busca notificações não enviadas e as envia por SMTP (configuração por variável de ambiente). Marcar como enviada só após envio bem-sucedido; registrar falhas com o logger, não com `print`. Web Push fica para a Parte 2 — email cobre o vencimento de fatura, que é o caso que importa.

**Pronto quando:** um usuário com fatura vencendo em 3 dias recebe o email em homologação, e há um teste do job com envio simulado.

### 4 · Mecanismo de módulos habilitados no frontend
*Frontend · 1 pessoa*

**Por quê:** é o que torna "cortar é adiar" verdadeiro. Sem isso, ocultar vira apagar.

**O que fazer:** variável `VITE_ENABLED_MODULES` lida uma vez em `src/config/modules.ts`. Em `App.tsx`, cada rota de módulo opcional só é registrada se o módulo estiver na lista; menu lateral e atalhos do dashboard consultam a mesma lista. Configuração do MVP: desligar `grocery`, `gamification`, `simulators`, `calendar`, `automations`, `benefit_cards` e `receipts` — mantendo **notificações ligadas**. Não editar as telas que a v1 queria "simplificar".

**Pronto quando:** trocar a variável e reconstruir muda o menu sem nenhuma outra alteração de código, e o E2E da Parte 1 continua passando.

### 5 · Log estruturado e migração fora do boot
*Backend · 1 a 2 pessoas · boa porta de entrada para quem está começando*

**Por quê:** sem log não se sustenta app publicado, e a migração no boot é uma bomba silenciosa em todo deploy.

**O que fazer:** configurar `logging` em `app/main.py` com formato estruturado, nível por variável de ambiente, e trocar cada `print(` por `logger.info/warning/error`. Remover do log qualquer dado sensível (hoje o prefixo da senha de PDF vai para o stdout). Mover `migrate_credit_card_transactions` para uma migração Alembic idempotente e tirar a chamada do `lifespan`.

**Pronto quando:** `grep -rn "print(" backend/app` não retorna nada fora de scripts; `alembic upgrade head` em banco vazio deixa o schema completo; o boot não agenda migração nenhuma.

### 6 · Consolidar E2E e cobrir a Parte 1
*Frontend · 1 a 2 pessoas*

**Por quê:** os critérios de aceite são, na prática, cenários E2E, e a suíte atual é duplicada e frágil.

**O que fazer:** apagar a suíte antiga em `e2e/*.spec.ts`, manter as specs numeradas em `e2e/tests/` e reincluir a pasta no ESLint. Adicionar `data-testid` nos elementos das telas da Parte 1 (login, upload, revisão de itens, faturas, pagamento) e trocar os seletores por texto. Reduzir a matriz para 2 projetos no CI (chromium claro e mobile), mantendo os 12 para rodada manual antes de release.

**Pronto quando:** os cenários "upload → revisão → confirmação → fatura → pagamento" rodam verdes no CI em menos de 10 minutos.

### 7 · Publicar no Google Play como TWA
*Frontend + cliente · 1 pessoa · depende do domínio definitivo (N04, N05)*

**O que fazer:** verificar o PWA em aparelhos reais (instalação, ícone, atualização do service worker, câmera para foto de fatura). Gerar o projeto Android com Bubblewrap a partir do manifest, publicar `/.well-known/assetlinks.json` no domínio, assinar e subir no Play Console na trilha de teste fechado. Implementar a exclusão de conta no app (N09) e revisar a política de privacidade **antes** de enviar. Recrutar os 12 testadores e manter 14 dias de uso real antes de pedir produção.

**Pronto quando:** o app instala pela Play Store na trilha fechada e um testador completa o ciclo: entrar, subir fatura, confirmar itens, ver a fatura, pagar.

---

## 10. Onde não mexer neste ciclo

Coisas que funcionam, foram calibradas com esforço, e cuja alteração exige justificativa com dados **antes** de qualquer PR:

- Prompts de extração em `documents/prompts/`, gerais e por banco. Só mudar com uma fatura sintética que demonstre o erro e o teste live do item 1 mostrando que o acerto subiu.
- Validador de extração e roteamento do Itaú pelo PDF nativo.
- Tolerância de casamento de parcelas e o serviço de detecção de duplicatas.
- Regras de fatura: período por dia de fechamento, status, pagamento parcial.
- Autenticação: JWT, refresh, API keys, convites.
- Saldo calculado a partir das transações (ADR-009). Nunca criar coluna de saldo.
- Refatorar os arquivos grandes (`invoices.py`, `transaction_service.py`, `financial_agent_service.py`, `Upload.tsx` com 2.368 linhas) **antes** de existirem os testes dos itens 2 e 6. Refatorar sem teste é reescrever no escuro.

---


