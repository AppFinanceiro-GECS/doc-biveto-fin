# Viabilidade Técnica — iOS

**Projeto:** Biveto-fin
**Sprint:** Sprint 1 (01/09 a 07/09)
**Responsável:** Julio Dourado (Grupo 1 — Infraestrutura, hospedagem e DevOps)
**Escopo:** Ecossistema Apple (iOS). O desenvolvimento Android é tratado em
documento separado pelo Brenno.

## 1. Introdução

Este documento avalia a viabilidade técnica de publicar e manter o Biveto-fin
no ecossistema Apple (iOS). O objetivo é decidir entre uma abordagem **nativa**
(Swift/SwiftUI) e uma abordagem **multiplataforma** (ex.: React Native,
Flutter), e mapear os custos, ferramentas e regras que impactam diretamente o
cronograma do MVP.

Para um MVP simples e barato, uma stack multiplataforma reduz o retrabalho
entre iOS e Android e é a opção recomendada neste estágio. Ainda assim, mesmo
optando por multiplataforma, o processo de **compilação final, assinatura de
código e publicação** exige obrigatoriamente um ambiente macOS com Xcode — não
há como contornar essa etapa, independentemente do framework escolhido. Por
isso, os itens abaixo (custos de conta, ambiente de build, revisão da App
Store e ciclo de testes) se aplicam ao projeto como um todo, e não apenas a
uma eventual versão nativa.

## 2. Custos de Manutenção

| Item | Custo | Observação |
| --- | --- | --- |
| Apple Developer Program | **US$ 99/ano** | Obrigatório para publicar na App Store, usar TestFlight e gerar certificados de distribuição. Renovação anual — se não renovado, o app é removido da loja. |
| Comissão da Apple (In-App Purchase) | **15% a 30%** sobre a venda | 30% é a taxa padrão; 15% se aplica a pequenas empresas (faturamento anual abaixo de US$ 1 milhão, via *App Store Small Business Program*) ou a partir do segundo ano de uma assinatura recorrente do mesmo usuário. |

**Quando a comissão se aplica:** apenas sobre a venda de **bens e serviços
digitais** consumidos dentro do app (assinaturas premium, funcionalidades
extras, conteúdo digital, créditos virtuais etc.), processados
obrigatoriamente pelo mecanismo de compra da Apple (StoreKit).

**Isenção (Reader Apps / bens e serviços físicos):** a comissão **não se
aplica** quando o pagamento se refere a bens ou serviços consumidos fora do
app, incluindo:

- pagamentos de contas, empréstimos ou serviços financeiros que não sejam
  "moeda" ou "crédito" interno do próprio app;
- produtos físicos ou serviços prestados fora do aplicativo.

Como o Biveto-fin é um **app financeiro** (controle e organização de
finanças, não uma loja de bens digitais), a expectativa é de que a maior
parte (ou totalidade) das operações fique **fora do escopo de comissão da
Apple**. Isso deve ser confirmado assim que o modelo de monetização do MVP
for definido — se houver um plano de assinatura "premium" do próprio app
(ex.: relatórios avançados, IA, sincronização), essa assinatura **é**
considerada bem digital e está sujeita à comissão.

## 3. Ambiente de Desenvolvimento (Vantagem)

A Apple exige que toda compilação final e assinatura de código para
distribuição (TestFlight ou App Store) seja feita em macOS, via Xcode. Times
sem hardware Apple normalmente precisam contratar máquinas virtuais macOS na
nuvem (ex.: **AWS EC2 Mac**, MacStadium, GitHub Actions com runner macOS
pago), o que adiciona um custo recorrente relevante ao orçamento do MVP.

Neste projeto, esse custo é **eliminado**: já possuo um **MacBook Air M4**,
que atende integralmente aos requisitos da Apple para:

- desenvolvimento e testes locais em simulador iOS;
- compilação (build) do app;
- assinatura de código (*code signing*) com certificados de desenvolvedor;
- upload de builds para o TestFlight e App Store Connect via Xcode.

**Impacto no orçamento:** custo de infraestrutura de build = **US$ 0**,
restando apenas a taxa anual do Apple Developer Program (item 2). Isso
simplifica a previsão orçamentária consolidada pedida na issue de
infraestrutura, pois remove uma variável de custo recorrente em nuvem.

## 4. Burocracias e Regras da App Store

A Apple submete todo app a revisão humana e automatizada contra as
[App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/).
Os pontos abaixo são os que representam maior risco de **rejeição** para um
app como o Biveto-fin:

- **Proibição de web wrappers (guideline 4.2 "Minimum Functionality"):** o
  app não pode ser essencialmente um site embrulhado em WebView sem
  funcionalidade nativa relevante. É necessário entregar experiência,
  navegação e componentes nativos (ou nativos via framework multiplataforma
  compilado), não apenas carregar páginas web.
- **Sign in with Apple obrigatório (guideline 4.8):** se o app oferecer
  login social de terceiros (Google, Facebook, etc.), é **obrigatório**
  oferecer também "Sign in with Apple" como alternativa equivalente. Login
  apenas via e-mail/senha próprio não exige essa regra, mas qualquer login
  social de terceiros a ativa.
- **App Tracking Transparency — ATT (guideline 5.1.2):** qualquer rastreamento
  do usuário entre apps/sites de terceiros para fins de publicidade exige o
  prompt oficial de permissão da Apple (`AppTrackingTransparency` framework)
  **antes** de coletar o identificador (IDFA). Coleta de dados financeiros
  sensíveis também exige políticas de privacidade claras e finalidade
  declarada no App Store Connect (*Privacy Nutrition Labels*).
- **Conteúdo dinâmico via backend (guideline 2.3.1 / 3.1.1):** é permitido
  atualizar conteúdo remotamente (textos, configurações, regras de negócio
  servidas pela API), mas **não é permitido** usar isso para alterar a
  funcionalidade principal do app sem nova revisão, nem para contornar
  mecanismos de pagamento da Apple (ex.: liberar compras "premium" via
  backend fora do StoreKit).

Como o Biveto-fin é um app de finanças pessoais com regras de negócio
consolidadas via backend (Grupo 3), é importante que a equipe de produto
esteja ciente de que **mudanças de funcionalidade** (não apenas de conteúdo/
dados) via backend podem exigir nova submissão de build.

## 5. Ciclo de Testes e Publicação

O fluxo de distribuição para iOS segue três etapas:

1. **TestFlight — Testes internos:** até **100 testadores** internos
   (membros do time com acesso ao App Store Connect). Distribuição
   **imediata**, sem revisão da Apple. Ideal para a própria equipe validar
   builds a cada sprint.
2. **TestFlight — Testes externos:** até **10.000 testadores** via link
   público ou convite por e-mail. Exige **Beta App Review** (revisão
   simplificada, geralmente em 24–48h) antes da primeira liberação externa;
   builds subsequentes só passam por nova revisão se houver mudanças
   significativas.
3. **App Store Connect — Publicação final:** submissão do build de produção
   para **App Review** completo (guidelines da seção 4). Prazo médio de
   revisão: 24–48h, podendo se estender em caso de rejeição e reenvio.

**Recomendação para o cronograma:** usar TestFlight interno desde as
primeiras sprints (build contínuo) e reservar o TestFlight externo/App Review
para as sprints finais de validação, já que a Beta Review e a App Review
completa adicionam tempo de espera fora do controle da equipe.

## 6. Vantagens e Desvantagens

### Vantagens

- **Alta rentabilidade e poder de compra:** usuários iOS historicamente
  gastam mais em apps e assinaturas do que usuários Android.
- **Engajamento:** base de usuários mais fiel, com maior retenção e uso
  recorrente — relevante para um app financeiro de uso contínuo.
- **Segurança e confiança:** ecossistema fechado, revisão obrigatória de
  apps e infraestrutura de autenticação (Face ID/Touch ID, Keychain) trazem
  confiança adicional para um app que lida com dados financeiros sensíveis.
- **Zero custo de infraestrutura de build**, graças ao MacBook Air M4
  próprio (seção 3).

### Desvantagens

- **Alta barreira de entrada técnica:** compilação e assinatura exigem
  macOS/Xcode; a curva de aprendizado de APIs, Human Interface Guidelines e
  processo de release é maior que em outras plataformas.
- **Diretrizes rígidas de revisão:** risco real de rejeição por motivos como
  web wrapper, ausência de Sign in with Apple ou falhas de privacidade
  (seção 4), o que pode atrasar entregas caso não sejam antecipadas.
- **Custos de licenciamento recorrentes:** taxa anual de US$ 99 e comissão
  de 15–30% sobre eventuais bens digitais vendidos dentro do app (seção 2).
- **Ciclo de revisão fora do controle do time:** mesmo o TestFlight externo
  e a publicação final dependem de prazos de revisão da Apple, o que exige
  planejamento antecipado nas sprints finais.

---

*Documento elaborado na Sprint 1, referente à issue
"[Sprint 1][Grupo 1] Planejar infraestrutura, custos e publicação".*
