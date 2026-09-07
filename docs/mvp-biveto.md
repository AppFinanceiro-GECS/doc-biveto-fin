# MVP do Biveto

**Proposta de escopo**

O Biveto hoje já resolve o problema central — entender a fatura sem digitação manual — mas cresceu com módulos que não sustentam esse valor. Esta proposta corta o produto até o que garante um lançamento estável e barato nas lojas, e organiza o resto como evolução futura.

## Sumário

1. [Resumo do produto](#1-resumo-do-produto)
2. [Princípios do MVP](#2-princípios-do-mvp)
3. [Requisitos atuais](#3-o-que-já-existe-e-o-que-fazer-com-cada-parte)
4. [Requisitos novos](#4-o-que-surgiu-na-conversa-com-o-cliente)
5. [Partes de entrega](#5-três-partes-de-entrega)
6. [Critérios de aceite](#6-como-saber-que-a-onda-1-está-pronta)
7. [Impacto no frontend](#7-o-que-muda-no-frontend)

---

## 1. Resumo do produto

O Biveto é um app de finanças pessoais para o público brasileiro. O usuário sobe a fatura do cartão (foto ou PDF), a IA lê e separa cada compra — inclusive parceladas e assinaturas recorrentes — e o app organiza tudo em contas, cartões, faturas e um resumo mensal de gastos. Existe também suporte a uso em família, com contas e transações compartilhadas.

O código atual vai muito além disso: são 29 módulos no backend e mais de 30 telas no frontend, incluindo metas, dívidas, orçamento, assistente de IA, gamificação, lista de mercado e integrações para desenvolvedores (MCP). Uma auditoria técnica recente mostrou que boa parte disso está desconectada do fluxo principal ou depende de uma infraestrutura (fila de tarefas agendadas) que não está confirmada como ativa em produção — ou seja, tem tela pronta para funcionalidade que o usuário final não recebe hoje.

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
App Store ou Google Play, com PWA como caminho mais rápido e barato de disponibilizar a versão mobile enquanto o processo de loja nativa corre em paralelo.

**05 · Cortar é adiar, não descartar**
O que sai do MVP permanece no código e na documentação como backlog validado.

---

## 3. O que já existe, e o que fazer com cada parte

Levantamento dos requisitos já implementados no código, classificados como **manter**, **alterar** ou **remover do MVP**. As regras de negócio (RN) referenciadas estão documentadas no mapa técnico do projeto, mantido pelo Grupo 3 (`docs/business/LOGIC_MAP.md`). O núcleo mantido segue a Regra Mestra definida pelo Grupo 3 (seção 2.2 do LOGIC_MAP): transação de cartão é passivo transitório até a fatura ser paga — só o pagamento da fatura mexe no saldo real das contas. Só entra nesta tabela requisito com RN documentada: sem regra associada, o item não é rastreável como regra de negócio e fica fora dela.

| Requisito | Módulo | Regra | Decisão | Motivo |
|---|---|---|---|---|
| Autenticar usuário | auth | RN012, RN013 | **manter** | Login por convite já funciona e é a base de tudo. |
| Registrar transação | transactions | RN009 | **manter** | Núcleo do produto, com normalização de merchant já testada. |
| Extrair fatura via IA | documents | RN014 | **alterar** | Fluxo funciona; trocar o modelo padrão para o Gemini Flash reduz custo por upload. |
| Confirmar itens extraídos | transactions | RN004 | **manter** | Etapa de revisão evita erro de leitura da IA — Grupo 3 removeu a tolerância de 5% no match (antiga RN011) e agora exige valor e data exatos. |
| Gerenciar parcelamento | installments | RN001, RN002 | **manter** | Muito comum em fatura brasileira; regra madura. |
| Calcular período da fatura | invoices | RN003 | **manter** | Baseado no fechamento do cartão; já testado. |
| Pagar fatura | invoices | RN005, RN007 | **manter** | Fluxo crítico de fechamento do ciclo mensal — formalizado pelo Grupo 3 como REQ-PAG-01 (só conta de liquidez) e REQ-PAG-02 (saldo parcial rola para o mês seguinte). |
| Compartilhar com a família | household | RN006 | **manter** | Diferencial pedido pelo cliente; manter escopo atual. |
| Ver resumo mensal | analytics | RN008 | **manter** | É o que dá sensação imediata de controle — formalizado como REQ-PAG-03 (pagamento de fatura não conta duas vezes no gasto do mês). |
| Gerar transação recorrente | recurring | RN010 | **alterar** | Depende do agendador; o próprio Grupo 3 classificou RN010 como "Adiada (Pós-MVP)", confirmando a decisão de validar o agendador antes de automatizar. |

**Funcionalidades removidas do MVP (sem RN associada)**

Estes módulos não têm regra de negócio no LOGIC_MAP.md — a ausência de regra é parte do próprio motivo do corte. Por isso ficam fora da tabela acima, que lista apenas requisitos com RN documentada.

- **Rastrear compras de mercado** (grocery): seis tabelas quase sem integração com o restante do app.
- **Conceder conquistas** (gamification): depende de gatilhos manuais frágeis; sem dado de engajamento.
- **Gerenciar cartão-benefício** (benefit_cards): duplica dado já coberto por fontes de renda.
- **Dividir recibo** (receipts): sobrepõe com o fluxo de documentos e confunde o usuário.
- **Simular cenário financeiro** (simulators): sem persistência; baixa prioridade para o lançamento.
- **Notificar vencimento/alerta** (notifications, automations): agendador ainda em validação gradual — tela não pode prometer o que não roda.
- **Consultar via Claude/ChatGPT** (mcp): ferramenta interna de desenvolvimento, não é o usuário final.
- **Ver calendário financeiro** (calendar): sem camada de serviço própria; baixa prioridade.

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

---

## 5. Três partes de entrega

Da funcionalidade que não pode faltar no dia do lançamento até o que só volta a ser discutido com dado de uso em mãos.

### Parte 1 — Lançamento

- Autenticação e convite
- Transação manual
- Upload e extração de fatura
- Parcelamento
- Cálculo e pagamento de fatura
- Resumo mensal
- Publicação (PWA ou App Nativo)

### Parte 2 —  (Pós-Entrega) Logo em seguida

- Orçamento mensal simplificado
- Compartilhamento familiar
- Recorrentes (após validar o agendador)
- Meta financeira única

### Parte 3 — (Pós-Entrega) Backlog pós-MVP

- Dívidas com estratégias
- Assistente de IA
- Notificações e automações
- Lista de compras inteligente
- Open Finance
- Gamificação, cartão-benefício, simuladores

---

## 6. Critérios de Aceitação

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
- App instalável como PWA ou App Android/iOS.

---

## 7. O que muda no frontend

O frontend atual tem mais de 30 páginas. Cortar o escopo do backend para o MVP também reduz o que precisa ser mantido, testado e carregado no app.

| Mantidas | Simplificadas | Ocultadas do MVP |
|---|---|---|
| Login / Onboarding | Formulário de transação — só campos essenciais visíveis por padrão | Notificações |
| Dashboard | Orçamento — sem cópia entre meses | Automações |
| Transações e Upload | Metas — uma metodologia só | Conquistas |
| Contas e Cartões | Dívidas — sem comparativo de estratégias | Lista de mercado |
| Faturas | | Simuladores |
| Família | | Calendário |
| | | Cartão-benefício / Recibos |

Ganho esperado: menos telas competindo por manutenção e teste, e nenhuma tela "vazia" prometendo uma função que o backend ainda não entrega — problema já identificado na análise técnica do time.

---


