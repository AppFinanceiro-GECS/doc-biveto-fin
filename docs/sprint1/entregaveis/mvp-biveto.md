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
9. [Onde não mexer neste ciclo](#9-onde-não-mexer-neste-ciclo)

---

## 1. Resumo do produto

O Biveto é um app de finanças pessoais para o público brasileiro. O usuário sobe a fatura do cartão (foto ou PDF), a IA lê e separa cada compra — inclusive parceladas e assinaturas recorrentes — e o app organiza tudo em contas, cartões, faturas e um resumo mensal de gastos. Existe também suporte a uso em família, com contas e transações compartilhadas.

O código vai além disso: 29 módulos no backend e 39 rotas no frontend (`frontend/src/App.tsx`), incluindo metas, dívidas, orçamento, assistente de IA, gamificação, lista de mercado e um servidor MCP. Parte desses módulos está fora do fluxo principal, e é aí que o MVP corta.

---

## 2. Princípios do MVP

**01 · Medir o custo de IA antes de cortar**
O padrão de extração já é Gemini Flash desde 4/4/2026 (commit `ef63404`, ADR-004), e o classificador já usa Flash-Lite. Não existe medição de custo por funcionalidade: não há registro de `usage_metadata` por upload nem por conversa do assistente. Portanto "extração é a parte mais cara" é hipótese, não dado. A [análise do Grupo 2](analise-uso-custo-gemini-2.md) estima, por ordem de grandeza, US$ 3 a 10/mês para 1.000 faturas com Flash-Lite — estimativa, não medição. O assistente (`financial_agent_service.py`, 3.540 linhas) faz várias chamadas por conversa e é o candidato mais provável a maior custo por usuário; ele já sai do MVP na Parte 3.

**02 · Só entra o que sustenta a promessa central**
"A fatura chegou, o app já entende." A promessa inclui o que acontece **depois** da leitura: ser lembrado do vencimento. Por isso notificação de vencimento entra na Parte 1, e não sai como "extra".

**03 · Nenhuma tela promete o que o backend não cumpre — e o caminho barato costuma ser acabar, não esconder**
O agendador roda (seção 1). O que falta em notificações é o envio por email, que é um job do APScheduler sobre uma tabela já populada. Esconder a tela custa mais do que terminar a entrega. O princípio continua valendo para o que realmente não existe.

**04 · Publicável em loja é prioridade — uma loja de cada vez**
Com os dados do Grupo 1, a diferença entre as duas lojas ficou mais precisa:

| | Google Play | App Store |
|---|---|---|
| Conta | US$ 25, **taxa única** | US$ 99/**ano** (app sai da loja se não renovar) |
| Ambiente de build | Windows, Linux ou macOS — custo US$ 0 | Exige macOS + Xcode; **já coberto** pelo MacBook Air M4 do time — custo US$ 0 |
| Web wrapper | Proibido (funcionalidade mínima) | Proibido (diretriz 4.2) |
| Trilha de teste | Interno (100 testadores, imediato) → fechado → aberto → produção (até 7 dias) | TestFlight interno (100) → externo (10 mil, Beta Review 24–48h) → produção |
| Gargalo de cronograma | **Teste fechado: 12 testadores opt-in por 14 dias corridos** — só para conta pessoal criada após 13/11/2023; conta Organização é **isenta** | Revisão humana a cada submissão |
| Manutenção recorrente | `targetSdkVersion` atualizado **todo ano**, sob pena de bloqueio para novas instalações | Renovação anual da conta |

A barreira de hardware que justificaria adiar o iOS **não existe** neste time. O que mantém a App Store na Parte 2 é o esforço de embrulhar o PWA num projeto Capacitor com recurso nativo real (câmera, push ou biometria) e sustentar conta de demonstração para a revisão (diretriz 2.1) — não o Mac. Play no MVP, App Store na Parte 2.

**05 · Cortar é adiar — e para isso precisa existir o mecanismo**
O frontend tem 39 rotas em `src/App.tsx` e **nenhum mecanismo de flag de funcionalidade**. Hoje "ocultar" só é possível editando rotas e menu na mão: ocultar vira apagar, e apagar não é adiar. A primeira tarefa de frontend do MVP é criar esse mecanismo (item 4 da proposta de trabalho).

---

## 3. O que já existe, e o que fazer com cada parte

| Requisito | Módulo | Regra (`main`) | Decisão | Motivo e fonte |
|---|---|---|---|---|
| Autenticar usuário | auth | RN013 | **manter** | Registro por convite é a ADR-005. JWT com refresh já existe e o interceptor em `src/services/api.ts` renova a sessão. Sem trabalho novo além de teste. |
| Registrar transação | transactions | a confirmar | **manter** | Núcleo do produto. Ressalva: `transaction_service.py` tem 1.995 linhas e **nenhum** teste automatizado (dívida #2). Manter sem testar é manter risco. |
| Extrair fatura via IA | documents | a confirmar | **manter** | É a parte mais trabalhada do sistema: 76 commits entre jan e ago/2026, classificador de tipo de documento, prompts versionados por banco (`prompts/banks/`), roteamento do Itaú por PDF nativo (`12a80e9`), validador de soma, provider alternativo (Mistral OCR). O padrão já é Flash (ADR-004). O único trabalho legítimo aqui é a migração de identificador de modelo — item 1 da proposta. |
| Confirmar itens extraídos | transactions | RN005 | **manter** | Confirmação manual antes de virar transação: mantida e não está em disputa. A remoção da tolerância de 5% segue **em aberto** (seção 10). |
| Gerenciar parcelamento | installments | RN005, RN012 | **manter** | Regra protegida por `tests/test_installment_fuzzy_match.py`. |
| Calcular período da fatura | invoices | RN003 | **manter** | Compra após o fechamento cai na fatura seguinte. Sem teste hoje (dívidas #2 e #3). |
| Pagar fatura | invoices | RN006, RN010 | **manter, com refinamento** | Liquidez restrita e status parcial **já estão implementados** — os REQ-PAG-01/02 são estas regras com outro nome, não trabalho novo. O refinamento que o Grupo 2 propõe é legítimo: validar "esta conta é origem de liquidez permitida?" em vez de "não é cartão?", e completar o fluxo de rolagem do saldo parcial. A lógica vive em `credit_cards/routers/invoices.py`, 1.243 linhas dentro de um router (dívida #3), sem um teste sequer. |
| Compartilhar com a família | household | a confirmar | **manter — Parte 1** | Já funciona; entra no lançamento sem trabalho novo. Divergência com a matriz do Grupo 2, que o classificou como pós-entrega — resolvida a favor da Parte 1, porque não há esforço a alocar. |
| Ver resumo mensal | analytics | RN008 | **manter, com refinamento** | Pagamento de fatura **já é excluído** do total de despesas (REQ-PAG-03 é esta regra). O refinamento do Grupo 2 é marcar a origem da transação explicitamente, em vez de reconstruir pela entidade de fatura. |
| Gerar transação recorrente | recurring | a confirmar | **manter, entrega na Parte 2** | Não há o que "validar no agendador": o job está registrado e ativo (PLANO_DE_ACAO 3.5, concluído em março) e o bug que o impedia de rodar foi corrigido em agosto (`fa813cc`). A limitação real está na ADR-003: só funciona com uma réplica da API. Falta confirmar que o job gera as recorrentes corretas em um ciclo real — teste do job + homologação por um mês. |
| Notificar vencimento | notifications | a confirmar | **manter e terminar** | O agendador já gera vencimento de fatura, orçamento estourado e fatura que não chegou (`alert_engine`). Falta a entrega por email (dívida #7) — item 3 da proposta. Divergência com a matriz do Grupo 2, que o listou fora do lançamento: resolvida a favor de terminar, por ser mais barato que esconder. |

---

## 4. Módulos fora do MVP (ocultar, não apagar)

| Módulo | Decisão | Motivo e fonte |
|---|---|---|
| grocery | **ocultar** | Cinco tabelas (`backend/app/models/grocery.py`). **Não apagar:** o requisito futuro N07 é exatamente este módulo, e a lista inteligente já existe em `grocery/services/smart_list_service.py`. PLANO_DE_ACAO, item 4.1. |
| gamification | **ocultar** | Sem dado de engajamento não há como saber se vale manter (PLANO_DE_ACAO, item 4.2). |
| simulators | **ocultar** | Sem persistência e fora da promessa central. |
| calendar | **ocultar** | Por prioridade, apenas: `calendar/services/cash_calendar_service.py` existe, com visões diária, semanal e mensal, saldo corrente e projeção. |
| benefit_cards | **ocultar só no frontend** | É conta de VR/VA com operadora, dia de recarga e valor esperado (`app/models/benefit_card.py`); o campo `linked_income_source_id` **liga** a uma fonte de renda, não a copia. **Remover o backend quebra recibos:** `receipts/services/receipt_service.py` importa esse modelo para decidir de que conta saiu um cupom. |
| receipts | **ocultar a tela, manter o backend** | O módulo confirma um cupom fiscal extraído e o transforma em transação com itens. Como o critério de aceite aceita cupom, **o backend continua necessário**. Unificar com `documents` é trabalho da Parte 2 (PLANO_DE_ACAO, item 4.3). |
| automations | **ocultar** | Regras "se X então Y" definidas pelo usuário: mais complexas e menos essenciais. Parte 2. |
| debts, chat | **ocultar** | Fora da promessa central. O chat é ainda o maior suspeito de custo de IA por usuário (princípio 01). |
| mcp | **fora do app, sem remoção** | Não é ferramenta interna de desenvolvimento: o servidor em `mcp-biveto-db/` consome a API pública com a chave `biv_` do próprio usuário (ADR-006) — é o usuário consultando as próprias finanças pelo Claude sem que o Biveto pague a conversa. Fica como está, sem tela no app. |

Ocultar significa **não registrar a rota** pelo mecanismo de módulos habilitados (item 4 da proposta), nunca apagar código.

---

## 5. Requisitos novos

| # | Requisito | Descrição | Prioridade |
|---|---|---|---|
| N01 | Publicar no Google Play | Empacotar o PWA como Trusted Web Activity (Bubblewrap): taxa única de US$ 25, exige HTTPS e `assetlinks.json` no domínio. Usar a trilha **interna** (100 testadores, distribuição imediata, sem revisão) desde já, e abrir o **teste fechado o quanto antes** se a conta for pessoal — os 14 dias correm em paralelo ao desenvolvimento, não no fim. | **MVP** |
| N01b | Publicar na App Store | Projeto Capacitor com recurso nativo real (câmera, push ou biometria) e conta de demonstração para a revisão (diretrizes 4.2 e 2.1). Build sem custo extra: o time já tem o Mac. Site empacotado é rejeitado. | Parte 2 |
| N02 | **Medir** custo de IA por funcionalidade | Registrar `usage_metadata` (tokens de entrada e saída) por chamada, substituindo as estimativas da análise do Grupo 2 por dado medido. Só depois disso decidir modelo por custo. | **MVP** |
| N03 | Verificar o PWA em aparelho real | **Não é requisito novo.** Já existem `frontend/public/manifest.json` (ícones em 8 tamanhos, `display: standalone`, pt-BR), `sw.js` registrado no `index.html`, screenshots e as páginas `privacy-policy.html` e `support.html` que as lojas exigem. A tarefa é instalar num Android e num iPhone e registrar o que falha (cache do service worker em atualização, por exemplo). | **MVP** |
| N04 | Simplificar hospedagem | O deploy já é um container único que serve backend e frontend na mesma origem (ADR-007). Falta: workflow de CI que constrói a imagem e publica num registro, e `alembic upgrade head` documentado como passo do deploy. Trocar de domínio exige refazer o `assetlinks.json` da TWA e o `id` do manifest. | **MVP** |
| N05 | Definir nova identidade | Nome e domínio próprios. Decisão do cliente; o manual de marca do Grupo 5 já está no Figma. | **MVP** |
| N06 | Resolver a exigência de testadores do Play | Corrigido com os dados do Grupo 1: é **teste fechado**, não interno; são 12 testadores com **opt-in contínuo por 14 dias corridos** e uso real verificado; e a regra **só vale para conta pessoal criada após 13/11/2023** — conta do tipo Organização (CNPJ/D-U-N-S) é isenta e publica direto. **A decisão do tipo de conta precisa sair agora** (seção 10), porque define se existe ou não um gargalo de 14 dias no cronograma. | **MVP** |
| N07 | Sugerir lista de compras | Já existe em `grocery/services/smart_list_service.py` — é o módulo que o MVP oculta. Registrado para não ser reinventado. | futuro |
| N08 | Conectar Open Finance | Importação bancária automática. Também é o caminho correto no lugar de ler SMS de banco: o Google restringe `READ_SMS` a apps que sejam o manipulador padrão de SMS do aparelho, o que inviabiliza essa via. | futuro |
| N09 | Excluir a própria conta no app | **Bloqueante para publicar.** Hoje só o administrador apaga um usuário (`DELETE /admin/users/{id}`). Apple exige exclusão de conta dentro do app desde 2022, Google desde 2023, e a LGPD exige canal para exportação e exclusão. | **MVP** |
| N10 | Declarar Data Safety e política de serviços financeiros | O Play Console exige declaração taxativa de quais dados o app coleta, com quem compartilha e para quê — declaração incorreta gera suspensão, e o peso é maior num app financeiro. Confirmar com o Grupo 3 se alguma funcionalidade do MVP se enquadra na Política de Serviços Financeiros do Google. | **MVP** |
| N11 | Bloquear reenvio de documento duplicado | Levantado pelo Grupo 2: a verificação de hash SHA-256 existe, mas é **só informativa** — não impede o reenvio nem evita reprocessar o OCR, gerando custo de IA a cada reenvio do mesmo arquivo. Em hash igual do mesmo usuário, retornar 409 e não reprocessar. | **MVP** |

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
- Compartilhamento familiar *(já funciona, sem trabalho novo)*
- Notificação de vencimento por email *(falta só a entrega)*
- Exclusão de conta pelo usuário (N09) *(bloqueante de loja)*
- Bloqueio de reenvio duplicado (N11)
- Publicação no Google Play via TWA

Módulos envolvidos: `auth`, `admin` (escopo mínimo: convites e licenças), `accounts`, `transactions`, `categories`, `documents`, `installments`, `credit_cards`, `analytics`, `notifications`, `household`.

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
| Aceita PDF e foto | Implementado (imagens, PDF, HEIC, planilhas), limite de 10 MB (`MAX_UPLOAD_SIZE_MB`) | Nada |
| Revisão antes de virar transação | Implementado (fluxo de confirmação) | Teste do fluxo |
| Reenvio detectado como duplicata e **bloqueado** | **Parcial:** o hash SHA-256 existe (`document_service.py:384`), mas é informativo — não bloqueia nem evita reprocessar o OCR | Implementar o bloqueio (N11) e o teste |
| Compra após fechamento cai na fatura seguinte | Implementado (RN003) | Teste — hoje inexistente |
| Pagamento não aceita cartão como origem | Implementado (RN006) | Teste; e formalizar "origem de liquidez permitida" em vez de "não é cartão" |
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
| #7 | Entrega de notificações não implementada | Está a um job de distância de funcionar. Item 3 da proposta. |
| #5 | Backend registra log com `print()` | Num app de loja, quando o usuário disser "minha fatura deu erro", a única pista é o stdout do container: sem nível, sem timestamp estruturado, sem identificação de usuário. |
| #4 | Migração de dados dentro do `main.py` | `migrate_credit_card_transactions` roda 30 s depois de **todo** boot e falha em silêncio. Pertence a uma migração Alembic executada uma vez. |
| #9 | Suítes E2E duplicadas e sem seletores estáveis | Duas suítes em paralelo, com seletores por texto em português que quebram a cada mudança de copy. Os critérios de aceite da seção 7 só são verificáveis com E2E confiável. |

**Segurança e conformidade:**

- `SECRET_KEY` forte é obrigatória em ambiente publicado: além de assinar os JWTs, ela deriva a chave que criptografa as senhas de PDF dos usuários (`app/core/crypto.py`).
- Exclusão de conta pelo próprio usuário não existe (N09) e bloqueia as duas lojas.
- Declaração de Data Safety no Play Console (N10) — incorreta, gera suspensão.
- Política de privacidade já publicada em `public/privacy-policy.html`, mas precisa de revisão antes do envio.
- **Fatura real jamais entra no repositório**, nem como fixture de teste. Isso vale também para a amostra de 30–50 faturas reais proposta pelo Grupo 2: ela roda localmente, na máquina de quem tem a chave, e só o **resultado agregado** (taxa de acerto, diferença por item) é publicado. O gitleaks bloqueia segredos, mas não reconhece uma fatura.
- Produção precisa estar no **paid tier** da Gemini API: o free tier tem cota diária restrita.

---

## 9. Onde não mexer neste ciclo

Coisas que funcionam, foram calibradas com esforço, e cuja alteração exige justificativa com dados **antes** de qualquer PR:

- Prompts de extração em `documents/prompts/`, gerais e por banco. Só mudar com uma fatura sintética que demonstre o erro e o teste live do item 1 mostrando que o acerto subiu. Hoje só Nubank, Itaú e Bradesco têm prompt específico — os demais caem no genérico, com qualidade não medida.
- Validador de extração e roteamento do Itaú pelo PDF nativo.
- Tolerância de casamento de parcelas e o serviço de detecção de duplicatas (D3).
- Regras de fatura: período por dia de fechamento, status, pagamento parcial.
- Autenticação: JWT, refresh, API keys, convites.
- Saldo calculado a partir das transações (ADR-009). Nunca criar coluna de saldo.
- A arquitetura em si: o diagnóstico do Grupo 2 confirma que o monólito modular atende o MVP. Não abrir frente de microserviços nem de camada de repositório.
- Refatorar os arquivos grandes (`invoices.py`, `transaction_service.py`, `financial_agent_service.py`, `Upload.tsx` com 2.368 linhas) **antes** de existirem os testes dos itens 2 e 6. Refatorar sem teste é reescrever no escuro.

---