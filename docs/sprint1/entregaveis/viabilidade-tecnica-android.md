# Viabilidade Técnica — Android

**Projeto:** Biveto-fin
**Sprint:** Sprint 1 (01/09 a 07/09)
**Responsável:** Brenno Oliveira (Grupo 1 — Infraestrutura, hospedagem e DevOps)
**Escopo:** Ecossistema Android (Google Play). O desenvolvimento iOS é tratado
em documento separado pelo Julio Dourado.

## 1. Introdução

Este documento avalia a viabilidade técnica de publicar e manter o Biveto-fin
no ecossistema Android (Google Play), seguindo o mesmo modelo do documento
elaborado para iOS, de forma a padronizar a análise entre as duas
plataformas e permitir comparação direta pela equipe de produto.

Assim como no documento iOS, para um MVP simples e barato uma stack
multiplataforma (React Native, Flutter, etc.) reduz o retrabalho entre as
duas plataformas e é a opção recomendada neste estágio. A diferença central
em relação ao iOS é que, no Android, **a compilação final, a assinatura de
código e a publicação não exigem um sistema operacional específico**: o
Android Studio e as ferramentas de build (Gradle) rodam nativamente em
Windows, Linux ou macOS. Isso remove por completo a principal barreira de
infraestrutura levantada no documento iOS (necessidade de um Mac ou de uma
máquina virtual macOS na nuvem).

Por isso, os itens abaixo (custos de conta, ambiente de build, revisão da
Play Store e ciclo de testes) se aplicam ao projeto como um todo, e não
apenas a uma eventual versão nativa.

## 2. Custos de Manutenção

| Item | Custo | Observação |
| --- | --- | --- |
| Google Play Console (conta de desenvolvedor) | **US$ 25 (taxa única)** | Pago uma única vez, sem renovação anual, diferença importante em relação aos US$ 99/ano da Apple. Permite publicar um número ilimitado de apps sob a mesma conta. |
| Comissão do Google Play (compras dentro do app) | **15% a 30%** sobre a venda | 15% se aplica ao primeiro US$ 1 milhão de receita anual do desenvolvedor processada via Google Play Billing; 30% sobre o que exceder esse valor no mesmo ano. A partir de 30/06/2026, em alguns mercados (EUA, Reino Unido, EEE) o Google passou a detalhar essa taxa em duas linhas, "service fee" (10%) + "billing fee" (5%), mas o efeito líquido para o desenvolvedor permanece equivalente aos 15% anteriores. |

**Quando a comissão se aplica:** apenas sobre a venda de **bens e serviços
digitais** consumidos dentro do app (assinaturas premium, funcionalidades
extras, conteúdo digital, créditos virtuais etc.), processados
obrigatoriamente pelo Google Play Billing.

**Isenção (bens e serviços físicos ou financeiros):** assim como no modelo
"Reader App" da Apple, a comissão do Google **não se aplica** a pagamentos
referentes a bens ou serviços consumidos fora do app, incluindo pagamentos
de contas, empréstimos ou serviços financeiros que não configurem "moeda"
ou "crédito" interno do próprio app.

Como o Biveto-fin é um **app financeiro** (controle e organização de
finanças, não uma loja de bens digitais), a expectativa, assim como no
documento iOS, é de que a maior parte (ou totalidade) das operações fique
**fora do escopo de comissão do Google**. Isso deve ser confirmado assim que
o modelo de monetização do MVP for definido: um eventual plano "premium" do
próprio app (relatórios avançados, IA, sincronização) seria considerado bem
digital e estaria sujeito à comissão.

## 3. Ambiente de Desenvolvimento (Vantagem)

Diferentemente da Apple, o Google **não exige um sistema operacional
específico** para compilar, assinar e publicar um app Android. O Android
Studio (IDE oficial) e o Gradle (build system) rodam de forma nativa em:

- Windows;
- Linux;
- macOS.

Isso significa que, para o Android, **não há necessidade de contratar
máquinas virtuais em nuvem** (como AWS EC2 Mac, MacStadium ou runners macOS
pagos), nem de depender de um hardware Apple específico, o que já é uma
vantagem estrutural em relação à etapa correspondente do documento iOS.

Neste projeto, esse custo já está coberto tanto pelo **MacBook Air M4**
mencionado no documento iOS (que também compila apps Android normalmente)
quanto por qualquer máquina Windows/Linux disponível na equipe, incluindo
runners gratuitos de CI (ex.: GitHub Actions, que oferece minutos gratuitos
em runners Linux/Windows, ao contrário dos runners macOS, que são pagos).

Outro ponto favorável é o **Play App Signing**: o Google gerencia e guarda a
chave de assinatura final do app, permitindo redefinir a chave de upload em
caso de perda, um processo mais tolerante a falhas do que o modelo de
certificados da Apple.

**Impacto no orçamento:** custo de infraestrutura de build = **US$ 0**,
restando apenas a taxa única de US$ 25 do Google Play Console (item 2). Isso
reforça a previsão orçamentária consolidada pedida na issue de
infraestrutura, eliminando qualquer variável de custo recorrente em nuvem
para esta plataforma.

## 4. Burocracias e Regras da Google Play Store

O Google submete todo app a revisão automatizada e, em casos específicos,
humana, contra a
[Política do Desenvolvedor do Google Play](https://play.google/developer-content-policy/).
Os pontos abaixo são os que representam maior risco de **rejeição ou
suspensão** para um app como o Biveto-fin:

- **Funcionalidade mínima / proibição de web wrappers:** assim como a Apple,
  o Google não aceita apps que sejam essencialmente um site embrulhado em
  WebView sem funcionalidade nativa relevante. É necessário entregar
  navegação, componentes e experiência nativos (ou nativos via framework
  multiplataforma compilado).
- **Permissões restritas de SMS e Log de Chamadas:** este é o ponto de maior
  atenção para um app financeiro. Desde 2019 o Google restringe fortemente
  as permissões `READ_SMS`, `RECEIVE_SMS` e `READ_CALL_LOG`: só podem ser
  usadas por apps registrados como o **manipulador padrão** de SMS,
  telefone ou assistente do aparelho. É comum que apps financeiros queiram
  ler SMS de bancos para categorizar gastos automaticamente — essa
  funcionalidade **dificilmente seria aprovada** pelo Google a menos que o
  Biveto-fin se torne o app padrão de SMS do usuário, o que não é realista
  para o escopo do MVP. Essa funcionalidade deve ser evitada ou substituída
  por integração via API/Open Finance.
- **Seção de Segurança de Dados (Data Safety):** equivalente aos "Privacy
  Nutrition Labels" da Apple. Todo app deve declarar no Play Console,
  taxativamente, quais dados coleta, com quem compartilha e para qual
  finalidade, assinatura de particular importância para um app que lida
  com dados financeiros sensíveis. Declarações incorretas podem gerar
  suspensão do app.
- **Política de Serviços Financeiros:** apps que se enquadrem nessa
  categoria (especialmente os que ofereçam empréstimos pessoais) podem
  precisar preencher declarações adicionais no Play Console (ex.: prazos e
  taxas de empréstimo). Vale confirmar, junto ao Grupo 3 (regras de
  negócio), se alguma funcionalidade do MVP se enquadraria nessa política
  específica.
- **Nível de API alvo (target API level):** diferentemente da Apple, o
  Google exige que os apps mantenham o `targetSdkVersion` atualizado para
  uma versão recente do Android **todos os anos** (normalmente até agosto),
  sob pena de o app deixar de estar disponível para novas instalações. Isso
  gera uma obrigação de manutenção contínua que não tem equivalente direto
  no documento iOS.
- **Conteúdo dinâmico via backend:** é permitido atualizar conteúdo
  remotamente (textos, configurações, regras de negócio servidas pela API),
  mas, assim como na Apple, **não é permitido** usar isso para alterar a
  funcionalidade principal do app sem nova revisão, nem para contornar o
  Google Play Billing.

Como o Biveto-fin é um app de finanças pessoais com regras de negócio
consolidadas via backend (Grupo 3), vale o mesmo alerta feito no documento
iOS: mudanças de **funcionalidade** (não apenas de conteúdo/dados) via
backend podem exigir nova submissão de build.

## 5. Ciclo de Testes e Publicação

O fluxo de distribuição para Android segue quatro etapas no Play Console
(uma a mais que o iOS):

1. **Teste interno:** até **100 testadores** internos, distribuição
   praticamente imediata (em segundos), sem revisão do Google. Ideal para a
   própria equipe validar builds a cada sprint — equivalente ao TestFlight
   interno do iOS.
2. **Teste fechado (closed testing):** compartilhamento com um grupo mais
   amplo, porém controlado, via listas de e-mail. **Ponto de atenção
   crítico para o cronograma:** contas de desenvolvedor **pessoais**
   criadas depois de 13/11/2023 são **obrigadas** a rodar um teste fechado
   com no mínimo **12 testadores opt-in contínuos por 14 dias corridos**
   antes de liberar o acesso à produção. Contas do tipo **Organização**
   (empresa, exige CNPJ/D-U-N-S) estão **isentas** dessa exigência. Como o
   requisito de 14 dias corridos é bem mais restritivo que a Beta Review de
   24–48h da Apple, **recomenda-se avaliar já na Sprint 1 se o projeto
   registrará a conta como Organização**, para não travar o cronograma do
   MVP nas sprints finais.
3. **Teste aberto (open testing):** torna a versão de teste visível na
   própria Play Store, com testadores ilimitados. Só fica disponível depois
   que a conta obtém acesso à produção.
4. **Google Play Console — Publicação final:** solicitação de acesso à
   produção, avaliada pelo Google em até **7 dias** (normalmente mais
   rápido). Atualizações de produção subsequentes costumam ser revisadas em
   poucas horas a poucos dias, geralmente mais rápido que o ciclo da Apple.

**Recomendação para o cronograma:** usar o teste interno desde as primeiras
sprints (build contínuo, sem atrito), e diferente da recomendação do
documento iOS de deixar tudo para o final, **iniciar o teste fechado o
quanto antes**, já na Sprint 1 ou 2, caso a conta seja pessoal, justamente
para "correr" os 14 dias corridos em paralelo ao desenvolvimento, e não como
gargalo de última hora.

## 6. Vantagens e Desvantagens

### Vantagens

- **Menor barreira técnica de build:** compilação e assinatura funcionam em
  Windows, Linux ou macOS, sem exigir hardware Apple nem VMs pagas, a
  principal diferença estrutural em relação ao iOS (seção 3).
- **Custo de conta menor e não recorrente:** US$ 25 pagos uma única vez,
  contra US$ 99/ano da Apple.
- **Maior alcance de mercado no Brasil:** o Android é a plataforma
  dominante no país, relevante para a base de usuários potenciais do
  Biveto-fin.
- **Ciclo de revisão de atualizações mais rápido**, uma vez superada a
  etapa inicial de teste fechado.

### Desvantagens

- **Exigência de teste fechado de 14 dias corridos** para contas pessoais
  novas, pode atrasar a chegada à produção se não for planejada com
  antecedência (ou se a conta não for registrada como Organização).
- **Fragmentação de dispositivos e versões do Android:** ao contrário do
  ecossistema fechado da Apple, é preciso testar em múltiplos fabricantes,
  tamanhos de tela e versões de sistema operacional.
- **Restrições rígidas de permissões sensíveis (SMS/Log de Chamadas):**
  limitam funcionalidades comuns em apps financeiros, como categorização
  automática de gastos via leitura de SMS bancário (seção 4).
- **Manutenção obrigatória do `targetSdkVersion`:** diferente da Apple, o
  Google exige atualização anual do nível de API alvo, sob risco de o app
  ser bloqueado para novas instalações, gera trabalho recorrente de
  manutenção fora do escopo do MVP inicial.
- **Comissão de 15–30%** ainda se aplica a eventual assinatura ou recurso
  premium vendido dentro do próprio app (seção 2).

---

*Documento elaborado na Sprint 1, referente à issue
"[Sprint 1](https://github.com/AppFinanceiro-GECS/doc-biveto-fin/issues/5) Planejar infraestrutura, custos e publicação",
padronizado com o modelo do documento de viabilidade técnica iOS.*
