# MVP do Biveto: o essencial para o primeiro lançamento

**Proposta de escopo · para validação do cliente**

O Biveto hoje já resolve o problema central — entender a fatura sem digitação manual — mas cresceu com módulos que não sustentam esse valor. Esta proposta corta o produto até o que garante um lançamento estável e barato nas lojas, e organiza o resto como evolução futura.



## Sumário

1. [Resumo do produto](#1-resumo-do-produto)
2. [Princípios do MVP](#2-princípios-do-mvp)
3. [Requisitos atuais](#3-o-que-já-existe-e-o-que-fazer-com-cada-parte)
4. [Requisitos novos](#4-o-que-surgiu-na-conversa-com-o-cliente)
5. [Partes de entrega](#5-três-partes-de-entrega)
6. [Critérios de aceite](#6-como-saber-que-a-onda-1-está-pronta)
7. [Impacto no frontend](#7-o-que-muda-no-frontend)
8. [Decisões pendentes](#8-o-que-precisamos-validar-com-o-cliente)

---

## 1. Resumo do produto

O Biveto é um app de finanças pessoais para o público brasileiro. O usuário sobe a fatura do cartão (foto ou PDF), a IA lê e separa cada compra — inclusive parceladas e assinaturas recorrentes — e o app organiza tudo em contas, cartões, faturas e um resumo mensal de gastos. Existe também suporte a uso em família, com contas e transações compartilhadas.

O código atual vai muito além disso: são 29 módulos no backend e mais de 30 telas no frontend, incluindo metas, dívidas, orçamento, assistente de IA, gamificação, lista de mercado e integrações para desenvolvedores (MCP). Uma auditoria técnica recente mostrou que boa parte disso está desconectada do fluxo principal ou depende de uma infraestrutura (fila de tarefas agendadas) que não está confirmada como ativa em produção — ou seja, tem tela pronta para funcionalidade que o usuário final não recebe hoje.

**Proposta:** lançar com o núcleo que já prova valor e custa pouco para manter, publicar nas lojas, e só então reintroduzir o restante com dado real de uso — não por já estar codificado.

---

## 2. Princípios do MVP

Cinco critérios usados para decidir o que fica, o que muda e o que espera — combinam o pedido do cliente na reunião inicial com o que a auditoria técnica encontrou no código.

**01 · Menor custo de IA possível**
Extração de imagem é a parte mais cara do produto. O padrão do MVP usa o modelo mais barato disponível (Gemini Flash / Flash-Lite) e reserva os outros provedores como opção de configuração, não como default.

**02 · Só entra o que sustenta a promessa central**
"A fatura chegou, o app já entende." Toda funcionalidade do MVP precisa apoiar diretamente essa frase — o resto é backlog.

**03 · Nenhuma tela promete o que o backend não cumpre**
Notificações, automações e conquistas dependem de um agendador que está em rollout gradual. Enquanto não for confirmado estável, a tela não vai ao ar.

**04 · Publicável nas lojas é prioridade de lançamento**
App Store e Google Play, com PWA como caminho mais rápido e barato de disponibilizar a versão mobile enquanto o processo de loja nativa corre em paralelo.

**05 · Cortar é adiar, não descartar**
O que sai do MVP permanece no código e na documentação como backlog validado — volta quando houver uso real que justifique o investimento.

---

## 3. O que já existe, e o que fazer com cada parte

Levantamento dos requisitos já implementados no código, classificados como **manter**, **alterar** ou **remover do MVP**. As regras de negócio (RN) referenciadas estão documentadas no mapa técnico do projeto (`docs/business/LOGIC_MAP.md`).

| Requisito | Módulo | Regra | Decisão | Motivo |
|---|---|---|---|---|
| Autenticar usuário | auth | RN013, RN014 | **manter** | Login por convite já funciona e é a base de tudo. |
| Cadastrar conta e cartão | accounts, credit_cards | — | **manter** | Pré-requisito para lançar qualquer transação. |
| Registrar transação | transactions | RN004, RN009 | **manter** | Núcleo do produto, com deduplicação já testada. |
| Extrair fatura via IA | documents | RN015 | **alterar** | Fluxo funciona; trocar o modelo padrão para reduzir custo por upload. |
| Confirmar itens extraídos | transactions | RN005 | **manter** | Etapa de revisão evita erro de leitura da IA. |
| Gerenciar parcelamento | installments | RN001, RN002, RN012 | **manter** | Muito comum em fatura brasileira; regra madura. |
| Calcular período da fatura | invoices | RN003, RN010 | **manter** | Baseado no fechamento do cartão; já testado. |
| Pagar fatura | invoices | RN006 | **manter** | Fluxo crítico de fechamento do ciclo mensal. |
| Compartilhar com a família | household | RN007 | **manter** | Diferencial pedido pelo cliente; manter escopo atual. |
| Ver resumo mensal | analytics | RN008 | **manter** | É o que dá sensação imediata de controle. |
| Gerar transação recorrente | recurring | RN011 | **alterar** | Depende do agendador; validar estabilidade antes de automatizar de verdade. |
| Planejar orçamento mensal | budgets | — | **alterar** | Manter limite por categoria + comparativo; adiar cópia entre meses. |
| Definir meta financeira | goals | — | **alterar** | Reduzir a uma metodologia (reserva de emergência) no lançamento. |
| Registrar dívida | debts | — | **alterar** | Manter cadastro simples; adiar comparativo de estratégias. |
| Consultar assistente de IA | chat | — | **alterar** | Reduzir escopo de perguntas para conter custo de token. |
| Rastrear compras de mercado | grocery | — | **remover** | Seis tabelas quase sem integração com o restante do app. |
| Conceder conquistas | gamification | — | **remover** | Depende de gatilhos manuais frágeis; sem dado de engajamento. |
| Gerenciar cartão-benefício | benefit_cards | — | **remover** | Duplica dado já coberto por fontes de renda. |
| Dividir recibo | receipts | — | **remover** | Sobrepõe com o fluxo de documentos e confunde o usuário. |
| Simular cenário financeiro | simulators | — | **remover** | Sem persistência; baixa prioridade para o lançamento. |
| Notificar vencimento/alerta | notifications, automations | — | **remover** | Agendador ainda em validação gradual — tela não pode prometer o que não roda. |
| Consultar via Claude/ChatGPT | mcp | — | **remover** | Ferramenta interna de desenvolvimento, não é o usuário final. |
| Ver calendário financeiro | calendar | — | **remover** | Sem camada de serviço própria; baixa prioridade. |

---

## 4. O que surgiu na conversa com o cliente

Registrados no formato verbo + substantivo a partir da reunião de organização inicial, com indicação se entram no MVP ou ficam para depois.

| # | Requisito | Descrição | Prioridade |
|---|---|---|---|
| N01 | Publicar aplicativo | Disponibilizar o Biveto na App Store e no Google Play — hoje o app não está publicado em nenhuma das duas. | **MVP** |
| N02 | Reduzir custo de extração | Padronizar o OCR/IA num modelo mais barato para imagem, ponto de custo mais sensível apontado pelo cliente. | **MVP** |
| N03 | Disponibilizar acesso mobile | Web app instalável (PWA) redirecionando para a experiência mobile — caminho mais rápido enquanto a loja nativa é aprovada. | **MVP** |
| N04 | Simplificar hospedagem | Pipeline de deploy mais direta e barata, com domínio novo para a marca. | **MVP** |
| N05 | Definir nova identidade | Nome e domínio próprios, necessários para publicar nas lojas sob a marca do cliente. | **MVP** |
| N06 | Habilitar testadores | Cadastrar ao menos 12 testadores internos no Google Play — exigência da própria loja para liberar a publicação. | **MVP** |
| N07 | Sugerir lista de compras | Gerar lista de mercado com base no padrão de gastos do usuário. | futuro |
| N08 | Conectar Open Finance | Importar dados bancários automaticamente via Open Finance. | futuro |

**Sobre N07 e N08:** ambos têm valor claro para o usuário, mas custo alto para o prazo da disciplina — N07 repete o padrão do módulo de mercado já hoje desconectado do core, e N08 envolve integração regulatória (Open Finance) que não cabe num primeiro ciclo. Recomendamos tratá-los como o topo do backlog pós-lançamento, não como parte do MVP.

---

## 5. Três partes de entrega

Da funcionalidade que não pode faltar no dia do lançamento até o que só volta a ser discutido com dado de uso em mãos.

### Parte 1 — Lançamento
*precisa estar pronto para publicar*

- Autenticação e convite
- Contas e cartões
- Transação manual
- Upload e extração de fatura
- Parcelamento
- Cálculo e pagamento de fatura
- Resumo mensal
- Publicação (loja + PWA)

### Parte 2 —  (Pós-Entrega) Logo em seguida
*semanas após o lançamento, sem bloquear a publicação*

- Orçamento mensal simplificado
- Compartilhamento familiar
- Recorrentes (após validar o agendador)
- Meta financeira única

### Parte 3 — (Pós-Entrega) Backlog pós-MVP
*volta com dado real de uso, não por já estar codificado*

- Dívidas com estratégias
- Assistente de IA
- Notificações e automações
- Lista de compras inteligente
- Open Finance
- Gamificação, cartão-benefício, simuladores

---

## 6. Como saber que a onda 1 está pronta

Critérios iniciais para os fluxos mais críticos do lançamento — ponto de partida para o time técnico refinar em histórias de usuário.

**Login e convite**
- Usuário só entra por convite — cadastro público continua desabilitado.
- Sessão expira e renova automaticamente sem derrubar o usuário no meio do uso.

**Upload e extração de fatura**
- Aceita PDF e foto/imagem da fatura ou cupom.
- Itens extraídos aparecem para revisão do usuário antes de virar transação — nenhum lançamento é criado sem confirmação.
- Reenviar o mesmo documento é detectado como duplicata e bloqueado.
- Modelo de IA padrão é o de menor custo disponível para o tipo de arquivo enviado.

**Fatura de cartão**
- Compra feita após o fechamento do cartão cai automaticamente na fatura do mês seguinte.
- Pagamento de fatura não aceita outro cartão de crédito como conta de origem.
- Status da fatura (aberta, fechada, paga, parcial) reflete o pagamento em tempo real.

**Parcelamento**
- Confirmar uma compra parcelada gera as parcelas futuras automaticamente.
- Quando a fatura real chega, a parcela já projetada é atualizada — nunca duplicada.

**Publicação**
- App instalável como PWA em Android e iOS.
- Google Play com pelo menos 12 testadores internos ativos, requisito para liberar a faixa de publicação.
- Nenhuma tela do app publicado leva a uma funcionalidade que não responde (ver onda 3).

---

## 7. O que muda no frontend

O frontend atual tem mais de 30 páginas. Cortar o escopo do backend para o MVP também reduz o que precisa ser mantido, testado e carregado no app.

| Mantidas | Simplificadas | Ocultadas do MVP |
|---|---|---|'
| Login / Onboarding | Formulário de transação — só campos essenciais visíveis por padrão | Notificações |
| Dashboard | Orçamento — sem cópia entre meses | Automações |
| Transações e Upload | Metas — uma metodologia só | Conquistas |
| Contas e Cartões | Dívidas — sem comparativo de estratégias | Lista de mercado |
| Faturas | | Simuladores |
| Família | | Calendário |
| | | Cartão-benefício / Recibos |

Ganho esperado: menos telas competindo por manutenção e teste, e nenhuma tela "vazia" prometendo uma função que o backend ainda não entrega — problema já identificado na análise técnica do time.

---

## 8. O que precisamos validar com o cliente

1. Publicar primeiro só como PWA (mais rápido e barato) e a loja nativa entra em paralelo, ou os dois já precisam estar prontos no dia do lançamento?
2. Confirmar reserva de emergência como a metodologia de meta financeira do lançamento — as demais (PNIF, Dave Ramsey completo) ficam para depois.
3. Open Finance é bloqueante para o cliente ou pode mesmo ficar para uma segunda fase, fora do escopo da disciplina?
4. Lista de compras inteligente: vale reinvestir nessa ideia antes de termos uso real do MVP, dado que a versão anterior (grocery) nunca engatou?
5. Qual o teto de custo mensal de IA aceitável — isso decide se o modelo padrão de OCR muda já no lançamento ou só quando o volume de usuários crescer.

---
