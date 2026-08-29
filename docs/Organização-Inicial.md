# Projeto Biveto-fin

## Resumo do projeto

O Biveto-fin é uma aplicação financeira em reformulação. O objetivo inicial é
definir um MVP mais simples e barato, com regras de negócio claras, interface
padronizada, infraestrutura viável e estratégia de publicação definida.

## Requisitos já implementados

- Levantar as funcionalidades existentes.
- Registrar cada requisito no formato **verbo + substantivo**.
- Classificar o que será mantido, alterado ou removido.

## Requisitos novos

- Consultar o cliente e/ou o Thiago sobre as necessidades da nova versão.
- Priorizar os requisitos do MVP.
- Definir critérios de aceite para os requisitos aprovados.

## Viabilidade técnica

A análise será realizada após a definição dos novos requisitos e considerará
arquitetura, integrações, segurança, infraestrutura, publicação e capacidade da
equipe.

## Custos de operação

Levantar e comparar os custos de:

- Gemini;
- VPS;
- domínio do site;
- armazenamento de objetos (*object storage*);
- publicação e manutenção da aplicação.

## Metodologia de desenvolvimento

- **Scrum**, com sprints de 7 dias.
- Planejamento e revisão ao início e ao final de cada sprint.
- Daily assíncrona pelo WhatsApp, informando andamento, próxima atividade e
  impedimentos.
- Decisões, tarefas e documentos registrados no GitHub.

## Equipe e cargos

A equipe possui **11 integrantes**, organizados em **quatro duplas e um trio**.

| Grupo | Tamanho | Frente principal | Integrantes |
| --- | ---: | --- | --- |
| Grupo 1 | 2 | Infraestrutura, hospedagem e DevOps | Julio + 1 a definir |
| Grupo 2 | 2 | Backend, dados e IA | 2 a definir |
| Grupo 3 | 2 | Backend, pagamentos e regras de negócio | 2 a definir |
| Grupo 4 | 2 | Produto, requisitos e frontend | Rafael e Guilherme |
| Grupo 5 | 3 | UX/UI, prototipação e frontend | João, Gabriel e Ana Luiza |

**Prazo:** definir os integrantes restantes até amanhã.

## Cronograma

O período de **01/09 a 12/11** possui dez sprints completas e uma sprint final
reduzida.

| Sprint | Período | Objetivo |
| --- | --- | --- |
| Sprint 1 | 01/09 a 07/09 | Reformulação e planejamento |
| Sprint 2 | 08/09 a 14/09 | A definir |
| Sprint 3 | 15/09 a 21/09 | A definir |
| Sprint 4 | 22/09 a 28/09 | A definir |
| Sprint 5 | 29/09 a 05/10 | A definir |
| Sprint 6 | 06/10 a 12/10 | A definir |
| Sprint 7 | 13/10 a 19/10 | A definir |
| Sprint 8 | 20/10 a 26/10 | A definir |
| Sprint 9 | 27/10 a 02/11 | A definir |
| Sprint 10 | 03/11 a 09/11 | A definir |
| Sprint 11 (reduzida) | 10/11 a 12/11 | Consolidação e encerramento |

## Sprint 1 — Reformulação e planejamento

**Objetivo:** consolidar a proposta do MVP e preparar os insumos técnicos,
visuais e financeiros para as próximas sprints.

| Responsável | Atividades | Entregas |
| --- | --- | --- |
| Grupo 1 | Pesquisar VPS, domínio, object storage, publicação e estratégia de implantação; apoiar a escolha entre iOS e Android; consolidar os custos levantados com o Grupo 2 | Comparativo de custos, riscos e estratégia de infraestrutura/publicação; previsão orçamentária e OMD/documento equivalente |
| Grupo 2 | Avaliar backend, dados, segurança, integração e uso do Gemini | Diagnóstico técnico, proposta inicial de arquitetura e custo do Gemini |
| Grupo 3 | Revisar os fluxos financeiros e definir uma regra de negócio única com o Grupo 4 | Regra consolidada e [mapa de lógica](https://github.com/AppFinanceiro-GECS/biveto-fin/blob/main/docs/business/LOGIC_MAP.md) atualizado |
| Grupo 4 | Reformular a proposta; levantar requisitos atuais e novos; definir as funcionalidades prioritárias | Resumo do produto, requisitos priorizados e critérios de aceite iniciais |
| Grupo 5 | Revisar telas; criar o layout padrão; organizar o protótipo e o manual da marca | Fluxo principal, protótipo consolidado e manual da marca |

Cada grupo deverá organizar e documentar suas próprias decisões e entregas no
GitHub.

### Critérios de conclusão

- proposta e requisitos do MVP priorizados;
- regra de negócio única documentada;
- protótipo e identidade visual organizados;
- viabilidade e plataforma inicial avaliadas;
- custos e orçamento preliminar registrados;
- documentação consolidada no GitHub.

## Diretrizes do MVP

- Utilizar uma **VPS** para hospedagem.
- Definir e validar uma **única regra de negócio**, evitando variações
  desnecessárias.
- Priorizar uma solução simples, barata e suficiente para validar o produto.

## Fluxo de trabalho

O GitHub será a fonte oficial do projeto:

1. O quadro Kanban geral acompanhará as entregas e dependências do projeto.
2. Cada área poderá manter um quadro próprio para suas tarefas internas.
3. Toda tarefa deverá possuir responsável, descrição e critério de conclusão.
4. Código, documentos, protótipos e decisões deverão estar vinculados aos
   repositórios ou às tarefas correspondentes.
5. O WhatsApp será usado para a daily assíncrona; decisões importantes deverão ser
   registradas também no GitHub.
