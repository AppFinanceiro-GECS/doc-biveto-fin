# Proposta de Trabalho — Itens de Backend (Biveto MVP)

**Origem:** proposta de trabalho enviada pelo cliente, filtrada apenas para os itens de responsabilidade de backend.
**Ordenação:** mantida a ordem por risco definida pelo cliente na proposta original — o que pode impedir o lançamento vem primeiro.
**Nota:** os quatro primeiros itens vêm diretamente da proposta do cliente, sem alteração de conteúdo. A última seção reúne itens que discutimos na sprint anterior e que se encaixam no mesmo escopo de backend, mas que **não constam na proposta do cliente** — marcados como sugestão para validar prioridade com ele, não como tarefa já aprovada.

---

## 1. Migrar o identificador do modelo Gemini

**Time:** Backend · 1 pessoa
**Depende de:** nada — é pré-requisito de tudo que envolve upload

**Por quê**
O modelo padrão do código foi desligado pelo Google. Sem essa correção, nenhum teste de extração roda e nenhum critério de aceite de upload pode ser verificado.

**O que fazer**
- Substituir todas as ocorrências literais de `gemini-2.0-*` por leituras de configuração.
- Criar em `config.py` três campos: `vision_model` (extração), `classifier_model` (classificador, hoje Flash-Lite) e `chat_model` (assistente). Nenhum outro ponto do código deve conhecer o nome de um modelo diretamente.
- Padrão inicial: `gemini-2.5-flash-lite` para o classificador, `gemini-2.5-flash` para extração e chat. Considerar os aliases `gemini-flash-latest` e `gemini-flash-lite-latest`, que o Google aponta para a versão vigente — isso evita repetir esse mesmo problema em outubro. Registrar a escolha e o trade-off (o alias muda sem aviso prévio) num ADR-011.
- Montar um conjunto de faturas sintéticas em `backend/tests/fixtures/` — uma por banco com prompt (Itaú em duas colunas, Nubank, Bradesco) mais um cupom fiscal — com o JSON esperado ao lado. Usar dados inventados; nunca fatura real.
- Escrever um teste marcado como `@pytest.mark.live` (não roda no CI, roda na máquina com chave configurada) que extrai cada fixture e compara itens, valores e total com o esperado.
- Rodar o mesmo teste com Flash e com Flash-Lite, registrando a diferença de acerto e de tempo entre os dois. Só depois desse número em mãos decidir o modelo padrão de extração — isso é o que o documento original do Grupo 4 chamava de N02.

**Está pronto quando**
`grep -rn "gemini-" backend/app` só encontra `config.py`; o teste live passa nas quatro fixtures; o ADR-011 está registrado com a tabela de acerto por modelo.

---

## 2. Testes de característica do fluxo de fatura

**Time:** Backend · 2 pessoas
**Depende de:** nada — pode começar em paralelo ao item 1

**Por quê**
O núcleo da Parte 1 não tem teste automatizado hoje. Os critérios de aceite já documentados são, na prática, os próprios casos de teste — falta escrevê-los.

**O que fazer**
- Criar `tests/test_invoices.py` cobrindo, no mínimo: compra feita depois do fechamento cai na fatura seguinte; vencimento anterior ao fechamento vai para o mês seguinte; pagar com conta do tipo cartão devolve erro 400; pagamento parcial deixa status `partial` e rola o restante; pagamento total deixa status `paid`.
- Criar `tests/test_transactions.py` cobrindo: duplicata por descrição, valor e data devolve 409; confirmação de item extraído cria a transação com a categoria sugerida; analytics do mês exclui o pagamento de fatura.
- Usar o `conftest.py` já existente (SQLite em memória por teste, client HTTP já configurado) — não é preciso infraestrutura nova.

**Está pronto quando**
Cada critério de aceite das seções "Fatura de cartão" e "Parcelamento" do documento original tem um teste com o nome do critério, e o CI roda todos.

---

## 3. Entregar notificações por email

**Time:** Backend · 1 pessoa

**Por quê**
Transforma uma "tela que promete e não cumpre" em funcionalidade real, com menos esforço do que esconder a tela e explicar por que ela não funciona.

**O que fazer**
- Criar um job em `app/core/scheduler.py` que, a cada N minutos, busca notificações não enviadas e as envia por SMTP (configuração via variável de ambiente; qualquer provedor com SMTP serve).
- Marcar como enviada só depois do envio bem-sucedido; registrar falhas com o logger, não com `print`.
- Push (Web Push) fica para a Parte 2; email cobre o vencimento de fatura, que é o caso que importa para o lançamento.

**Está pronto quando**
Um usuário com fatura vencendo em 3 dias recebe o email em ambiente de homologação, e existe um teste do job com o envio simulado.

---

## 4. Log estruturado e migração fora do boot

**Time:** Backend · 1 a 2 pessoas · indicado para quem está começando no projeto

**Por quê**
Sem log estruturado, um app publicado não se sustenta em operação. A migração rodando no boot é uma bomba silenciosa em todo deploy.

**O que fazer**
- Configurar logging em `app/main.py` com formato JSON ou chave=valor, nível controlado por variável de ambiente, e trocar cada `print(` por `logger.info`, `warning` ou `error` conforme o caso. Remover do log qualquer dado sensível — hoje o prefixo da senha de PDF vai para o stdout.
- Mover `migrate_credit_card_transactions` para uma migração Alembic idempotente e remover a chamada do lifespan.

**Está pronto quando**
`grep -rn "print(" backend/app` não retorna nada fora de scripts; `alembic upgrade head` em banco vazio deixa o schema completo; o boot não agenda nenhuma migração.

---

## Itens adicionais sugeridos (não constam na proposta do cliente)

Os itens abaixo surgiram do diagnóstico e do registro de riscos da sprint anterior, têm o mesmo perfil de trabalho de backend dos itens acima, e ainda estão sem solução na proposta que o cliente enviou. Ficam aqui como sugestão para incluir na priorização — não como tarefa já validada por ele.


### Bloquear reenvio de documento duplicado

**Time:** Backend · 1 pessoa · esforço pequeno

**Por quê**
É um critério de aceite já documentado para o MVP ("reenviar o mesmo documento é detectado como duplicata e bloqueado") que hoje falha na prática: existe verificação de hash SHA-256 por usuário, mas ela é só informativa — não impede o reenvio nem evita reprocessar o OCR, o que gera custo desnecessário de IA a cada reenvio do mesmo arquivo.

**O que fazer**
- Em hash igual do mesmo usuário, retornar um erro explícito (ex: HTTP 409) em vez de log silencioso.
- Não reprocessar o OCR nesse caso.

**Está pronto quando**
Reenviar o mesmo arquivo retorna erro ao usuário e não gera uma nova chamada ao Gemini.
