# Análise de Uso e Custo do Gemini — Biveto MVP

**Sprint:** 1 — Diagnóstico de Backend, Dados e IA
**Data:** 09/09/2026 (revisado a partir do retorno do cliente)
**Escopo:** avaliar o modelo de IA usado na extração de fatura (OCR), sua viabilidade técnica e seu custo no contexto do MVP

---

### Nota de revisão

Esta versão incorpora o retorno do cliente sobre o MVP enviado pelo Grupo 3. O cliente é quem decide o rumo do projeto, então os pontos abaixo substituem o que estava escrito antes, não se somam a ele.

- **Como era:** a troca de modelo era apresentada como uma medida de redução de custo — "trocar para o Flash porque é o mais barato".
- **O que mudou:** custo não é o motivo da troca, porque ainda não foi medido. O motivo real é que o identificador de modelo em uso está morto (desligado pela Google). A seção 3 foi reescrita para tratar isso como migração técnica de identificador — modelo morto para modelo vivo —, não como otimização de custo. Custo entra depois, com medição.
- **Como era:** a remoção da tolerância de 5% no match (RN004/antiga RN011) era tratada como decisão já fechada pelo Grupo 3.
- **O que mudou:** o cliente pediu que essa remoção não seja considerada definitiva sem casos reais de fatura em que os 5% causaram um casamento errado, e sem os testes de fuzzy match correspondentes ajustados. A seção 6 foi reescrita para refletir isso como decisão em aberto.

---

## 1. Achado urgente

O modelo configurado como padrão no projeto para extração de fatura, `gemini-2.0-flash`, foi desligado pela Google em **1º de junho de 2026**. A partir dessa data, chamadas para esse modelo (e para `gemini-2.0-flash-lite`) passam a retornar erro, sem período de graça e sem distinção entre contas free e pagas. A informação está confirmada na página oficial de deprecations da Gemini API e foi corroborada por cobertura independente sobre o corte.

Isso significa que, se essa configuração não foi alterada desde então, **o fluxo central do produto — "a fatura chegou, o app já entende" — pode estar quebrado em produção neste momento**, de forma silenciosa.

Vale marcar com clareza: este é um problema de **disponibilidade**, não de custo. O modelo em uso já pertencia à linha mais barata (Flash) antes mesmo desta análise — a decisão de usar Flash em vez de Pro já estava tomada. O que está quebrado é a versão específica dentro dessa linha, não a escolha da linha em si.

---

## 2. Estado atual no código

O modelo antigo aparece em mais de um ponto do projeto, não apenas na extração principal:

- **OCR de fatura (fluxo principal):** `gemini-2.0-flash`, configurado via variável de ambiente `VISION_MODEL`.
- **Classificador de tipo de documento:** `gemini-2.0-flash-lite`, hardcoded (não parametrizado por env).
- **Chat financeiro:** aponta para a mesma geração 2.0 Flash-Lite. Este módulo já está classificado como fora do escopo do MVP (Parte 3 — backlog, conforme o documento do Grupo 4). Se as rotas dele ainda estiverem ativas e acessíveis, o ponto relevante não é migrar o modelo que ele usa — é confirmar que ele está de fato desligado (ver seção 4).
- **Troca de provedor:** existe parametrização via `VISION_PROVIDER` / `VISION_MODEL` permitindo Google ou Mistral; as opções OpenAI e Anthropic estão previstas no código mas não funcionam hoje por falta de chave configurada em Settings.

A migração do modelo, portanto, não é uma alteração de uma linha só — precisa cobrir o fluxo principal e o classificador, saindo do hardcode para entrar na mesma configuração via ambiente.

---

## 3. Migração de modelo — correção de disponibilidade, não decisão de custo

O modelo em uso está desligado; é preciso migrar para um identificador vivo. A escolha de para qual modelo migrar não deve ser guiada por suposição de custo — isso exigiria uma medição que ainda não existe —, mas por continuidade do que já estava decidido (linha Flash, já mais barata que a linha Pro) e por validação de que a extração continua funcionando como antes.

**Modelo indicado para a migração: `gemini-2.5-flash-lite`.** Não porque seja "o mais barato disponível" — esse argumento pertence à etapa de medição de custo, não a esta migração —, mas porque mantém a mesma classe de modelo já decidida e é a versão estável mais recente dessa classe.

A tarefa correta é uma **migração de identificador**, não uma reavaliação de custo:

1. Centralizar todas as ocorrências do nome do modelo numa única configuração — hoje estão espalhadas entre `VISION_MODEL` (fluxo principal), o classificador (hardcoded) e o chat.
2. Apontar essa configuração para um modelo vivo (`gemini-2.5-flash-lite`).
3. Provar com faturas de teste que nada regrediu — comparar a extração antes e depois da troca usando um conjunto de faturas conhecidas.
4. Só depois disso, com telemetria de custo real (seção 4), avaliar se há motivo para trocar de modelo por razão de custo. Não antes.

Referência de preço, para uso na etapa de medição — não como critério da migração em si:

| Modelo | Entrada (por milhão de tokens) | Saída (por milhão de tokens) | Situação |
|---|---|---|---|
| gemini-2.5-flash-lite | US$ 0,10 | US$ 0,40 | Estável — indicado para a migração |
| gemini-2.5-flash | US$ 0,30 | US$ 2,50 | Estável, mais caro — fallback de qualidade se necessário |
| gemini-2.0-flash / flash-lite | — | — | **Desligado desde 01/06/2026 — não usar** |

**Ressalva mantida:** o próprio `gemini-2.5-flash` já tem desligamento anunciado para **16 de outubro de 2026**, com `gemini-3.5-flash` como sucessor indicado pela Google. Isso reforça dois pontos: o modelo precisa continuar parametrizado por configuração em todos os pontos de uso, e o acompanhamento da página de deprecations da Gemini API precisa virar processo recorrente — não uma correção pontual feita uma vez e esquecida.

---

## 4. Estimativa de custo — e a alavanca real de redução

**Método:** não existe hoje, no código, nenhum registro (`usage_metadata`) de tokens consumidos por upload. Os números abaixo são estimativas de ordem de grandeza, baseadas em suposições razoáveis de tamanho médio de fatura — **não em medição real**. Nenhuma decisão de custo deveria se apoiar só nesses números.

| Cenário | Entrada estimada | Saída estimada | Custo estimado (2.5 Flash-Lite) | Custo estimado (2.5 Flash) |
|---|---|---|---|---|
| PDF de 5 páginas, modo texto (`GOOGLE_PDF_MODE=text`) | ~15–40 mil tokens | ~2–8 mil tokens | US$ 0,003–0,007 | US$ 0,01–0,03 |
| PDF de 5 páginas, renderizado como imagens | ~5 imagens + prompt | ~2–8 mil tokens | US$ 0,005–0,02 | 2–3x maior |
| Foto única (comprovante) | ~1–3 mil tokens + 1 imagem | ~1–4 mil tokens | Muito abaixo de US$ 0,01 | Muito abaixo de US$ 0,02 |

Projeção grosseira para 1.000 faturas/mês, predominantemente PDFs de ~5 páginas em modo texto: US$ 3 a US$ 10/mês com Flash-Lite, US$ 10 a US$ 30/mês com Flash.

---

## 5. Limitações práticas

- **Tamanho de upload:** limite atual de 10 MB (`MAX_UPLOAD_SIZE_MB`).
- **Formatos aceitos:** JPG, PNG e PDF (além de planilhas para outros fluxos).
- **Janela de contexto:** ~1 milhão de tokens de entrada no gemini-2.5-flash-lite — folgada para o caso de uso.
- **Rate limits:** o free tier tem cota diária restrita; produção precisa estar necessariamente no paid tier.
- **Páginas por PDF:** não há limite superior configurado no conversor — uma fatura muito longa gera custo e latência variáveis sem teto definido.
- **Cobertura de prompt por banco (RN014):** hoje só três instituições têm prompt de OCR específico (Nubank, Itaú, Bradesco). Faturas de outros bancos caem no prompt genérico, com qualidade de extração não medida.

---

## 6. Precisão de extração — e a decisão em aberto sobre a tolerância de 5%

O Grupo 3 havia proposto remover a tolerância de 5% que antes existia na conciliação de parcelas e recorrentes (antiga RN011), substituindo-a pela exigência de match exato (RN004 atualizada). O cliente, ao revisar essa proposta, **não aceitou a remoção como decisão fechada**: pediu que ela só avance se vier acompanhada de casos reais de fatura em que a tolerância de 5% tenha causado um casamento incorreto, e com os testes de fuzzy match ajustados de acordo. A manutenção da confirmação manual como camada de segurança está correta e não está em disputa — o que está em aberto é especificamente a remoção da tolerância automática.

Para este documento, o ponto prático é que a mesma amostra de faturas reais proposta abaixo serve a dois propósitos ao mesmo tempo: medir a precisão do modelo de extração, e reunir os casos concretos que a decisão sobre a tolerância de 5% está pedindo. Não é necessário rodar dois testes separados.

**Próximo passo proposto:** rodar uma amostra de 30 a 50 faturas reais (priorizando os três bancos já cobertos por prompt específico, mais 2 ou 3 exemplos sem overlay) contra o `gemini-2.5-flash-lite`, registrando, para cada item extraído: se bateu exatamente com o valor e a data reais, e — para os casos que não bateram — se a diferença estaria dentro da antiga tolerância de 5%/15 dias. Isso gera, ao mesmo tempo, a métrica de precisão do modelo e a evidência concreta que falta para decidir sobre a tolerância.

Esse teste também é o momento certo para instrumentar o registro real de tokens consumidos por upload, substituindo as estimativas da seção 4 por dados medidos.

---

## Resumo das ações recomendadas

1. Migrar o identificador de modelo morto (`gemini-2.0-flash` / `flash-lite`) para um modelo vivo (`gemini-2.5-flash-lite`), centralizando a configuração em todos os pontos de uso. Tratar como correção de disponibilidade, não como decisão de custo.
2. Validar a migração com faturas de teste, comparando extração antes e depois, antes de considerar a troca concluída.
3. Instrumentar telemetria de tokens por upload — pré-requisito para qualquer decisão de custo futura, incluindo eventual troca de modelo por esse motivo.
4. Rodar a amostra de 30–50 faturas reais, usada tanto para medir precisão de extração quanto para embasar (ou não) a decisão sobre remover a tolerância de 5% do match — decisão que segue em aberto até haver esses casos.
5. Manter acompanhamento periódico da página de deprecations da Gemini API, dado o histórico recente de ciclos curtos de desligamento de modelo.
