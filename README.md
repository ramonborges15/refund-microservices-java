# refund-microservices-java
Projeto de estudo de um Sistema de Solicitação e Processamento de Reembolsos Empresariais, desenvolvido em microserviços com Spring Boot, PostgreSQL e mensageria (RabbitMQ).

## Documento de Requisitos do Produto (PRD) – Sistema de Reembolso Empresarial

## Introdução

Este PRD descreve um sistema educacional simplificado de **Solicitação e Processamento de Reembolsos Empresariais**, desenvolvido em até uma semana. A arquitetura será orientada a microserviços independentes, sem dependências externas ou autenticação. Cada serviço será implementado com Spring Boot, PostgreSQL e comunicação assíncrona via mensageria (Kafka ou RabbitMQ). O fluxo de transação utilizará o padrão SAGA para garantir consistência eventual dos dados. Em sistemas distribuídos, o padrão SAGA permite manter a consistência dos dados entre múltiplos serviços sem transações distribuídas complexas, executando transações locais sequenciais e, em caso de falha, aplicando ações compensatórias para desfazer operações anteriores.

## Objetivos do Produto

* Permitir que funcionários submetam solicitações de reembolso (valor, descrição, ID) via API REST.
* Processar essas solicitações por uma sequência de serviços (financeiro, aprovação, pagamento) de forma assíncrona.
* Simular regras financeiras e gerenciais básicas (verificar orçamento fictício, aplicar limite de aprovação).
* Registrar o pagamento do reembolso em caso de aprovação final.
* Demonstrar a orquestração de transações distribuídas usando mensageria e compensações automáticas (padrão SAGA).

## Escopo

O sistema atenderá um caso de uso básico de reembolso corporativo, com os seguintes limites:

* **Sem terceiros ou autenticação**: todas as regras internas são fictícias, sem integração real com serviços externos ou componentes de segurança.
* **Desenvolvimento rápido**: foco em APIs e lógica de negócio simples, permitindo implementação em até uma semana.
* **Serviços back-end somente**: não há interface de usuário complexa, apenas APIs REST para testes via Postman/Swagger.
* **Dados simulados**: orçamentos e aprovações são calculados por regras internas simples; não há conexão com sistemas financeiros reais.

## Arquitetura e Tecnologias

O sistema seguirá uma **arquitetura de microserviços**, onde cada serviço encapsula seu próprio domínio de dados em um banco dedicado. Essa abordagem isola falhas (um serviço em pane não derruba os demais) e permite escalabilidade independente. Cada serviço conterá suas camadas típicas (controller, service, repository) organizadas por domínio, facilitando manutenção e entendimento do código.

A comunicação entre serviços será feita de forma **assíncrona por mensageria** (Kafka ou RabbitMQ). Cada transação local (ex.: criar requisição, validar, aprovar, pagar) encerrará atualizando seu banco e publicando um evento na fila para acionar o próximo passo. Não usamos transações distribuídas (2PC); em vez disso, adotamos o padrão **SAGA** para consistência eventual. O SAGA coordena cada etapa como uma transação local. Se uma etapa falhar, são executadas transações compensatórias que desfazem as alterações feitas anteriormente.

O framework principal será **Spring Boot** (uma aplicação independente por serviço), com **PostgreSQL** para persistência local e APIs **RESTful** para exposição dos endpoints. Não haverá autenticação/autorização. Todo o sistema deverá rodar localmente (por exemplo via Docker) e ser testável via ferramentas como Postman ou Swagger.

## Serviços Principais

Os serviços do sistema serão divididos pelos domínios principais do fluxo de reembolso. Cada serviço terá responsabilidade clara e independente:

| **Serviço**                | **Função Principal**                                                                                                                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *Solicitação de Reembolso* | Recebe requisições de reembolso (valor, descrição, ID do funcionário), armazena no banco com status inicial e publica evento de nova solicitação.                                                                          |
| *Validação Financeira*     | Consome evento de nova solicitação, verifica disponibilidade de orçamento fictício e atualiza o status (aprovado ou rejeitado), publicando evento de resultado da validação.                                               |
| *Aprovação Gerencial*      | Consome evento de validação aprovada, aplica regra simples de aprovação (ex: valor abaixo de certo limite) e atualiza o status (aprovado ou rejeitado), publicando evento de decisão gerencial.                            |
| *Pagamento*                | Consome evento de aprovação gerencial aprovada, simula o pagamento do reembolso (por exemplo, gravando dados em tabela de pagamentos) e publica evento de pagamento realizado.                                             |
| *Orquestração SAGA*        | Controla o fluxo geral entre os serviços acima via mensageria. Atua como orquestrador central, enviando comandos e monitorando transações; em caso de falhas, aciona transações compensatórias para manter a consistência. |

### Fluxo de Processamento de Reembolsos

O fluxo típico de processamento ocorrerá nas seguintes etapas:

1. **Cadastro da Solicitação:** o funcionário submete via API os dados da solicitação (valor, descrição, ID). O Serviço de Solicitação registra a entrada como *PENDENTE* e publica um evento (ex.: `SolicitacaoCriada`).
2. **Validação Financeira:** o Serviço de Validação Financeira consome `SolicitacaoCriada`. Ele verifica, de forma simulada, se há orçamento disponível para cobrir o valor (por exemplo, comparando contra um limite fixo ou acumulado fictício). Se não houver orçamento suficiente, atualiza o status para *FINANCEIRO\_REJEITADO* e finaliza o fluxo; caso contrário, atualiza para *FINANCEIRO\_APROVADO* e publica evento (ex.: `ValidacaoFinanceiraConcluida`).
3. **Aprovação Gerencial:** o Serviço de Aprovação Gerencial consome `ValidacaoFinanceiraConcluida`. Ele aplica regra simples (por exemplo, se valor ≤ R\$1.000, aprova automaticamente; caso contrário, rejeita). Atualiza o status para *GERENTE\_APROVOU* ou *GERENTE\_REJEITOU* e publica evento (ex.: `AprovacaoGerencialConcluida`).
4. **Pagamento:** o Serviço de Pagamento consome `AprovacaoGerencialConcluida` apenas se a decisão for de aprovação. Ele registra o pagamento (simulado) no banco de dados, atualiza o status para *PAGO* e emite evento (ex.: `PagamentoRealizado`). Após isso, o processo de reembolso se encerra com sucesso.
5. **Tratamento de Falhas (Compensações):** se ocorrer erro ou rejeição em qualquer etapa, o orquestrador SAGA coordena a execução de transações compensatórias. Por exemplo, se o pagamento falhar, o orquestrador enviaria comandos para desfazer aprovações anteriores. Isso garante que o estado final do sistema seja consistente mesmo diante de falhas.

## Requisitos Funcionais

O sistema deve implementar os seguintes requisitos funcionais:

* **Cadastro de Reembolso:** o sistema deve permitir criar uma nova solicitação de reembolso informando valor, descrição e ID do solicitante. Ao criar, registra a solicitação (status inicial *PENDENTE*) e dispara o evento que inicia o fluxo de SAGA.
* **Validação Automática:** ao receber a notificação de nova solicitação, o sistema financeiro deve checar automaticamente um orçamento fictício. Se não houver verba disponível, a solicitação deve ser rejeitada; caso contrário, marcada como aprovada nesse estágio, permitindo avançar para a próxima fase.
* **Aprovação Gerencial:** o sistema deve aplicar regras simples de aprovação. Por exemplo, se o valor do reembolso for inferior a um limite estabelecido (ex: R\$ 1.000), aprova automaticamente; se ultrapassar, rejeita ou aguarda decisão. Essa decisão atualiza o status da solicitação e é comunicada aos demais serviços via evento.
* **Simulação de Pagamento:** após aprovação gerencial, o sistema deve simular o pagamento do reembolso. Isso envolve registrar o pagamento no banco (por exemplo, criando um registro em tabela de pagamentos) e atualizar o status do pedido para *PAGO*. Um evento de conclusão de pagamento deve ser publicado.
* **Transações Compensatórias:** se qualquer serviço falhar (por exemplo, erro inesperado ou rejeição) durante o fluxo de processamento, o padrão SAGA deve acionar transações de compensação que revertam as etapas já concluídas. Isso garante que, em caso de falha, nenhuma operação parcial reste sem controle (por exemplo, um reembolso não pago não deve ficar marcado como aprovado).

## Requisitos Não Funcionais

O sistema deve atender aos requisitos não funcionais abaixo:

* **Execução Local e Independente:** todo o sistema deve ser executável localmente (pode-se usar contêineres Docker para banco e broker) e não deve depender de serviços externos. Não haverá autenticação/autorização, simplificando o desenvolvimento e testes.
* **APIs REST e Testabilidade:** cada serviço deve expor APIs REST para as operações principais (criar, consultar status etc.). Documentação Swagger e/ou coleção Postman deve ser fornecida para permitir testes fáceis das APIs.
* **Organização por Domínio:** o código deve ser estruturado por serviços e domínio (cada microserviço em projeto separado, com suas entidades, repositórios e controladores específicos), seguindo boas práticas de design. Essa organização facilita a manutenção e a compreensão do código.
* **Mensageria e Resiliência:** o uso de mensageria deve permitir que serviços reajam a eventos de forma assíncrona e tolerem falhas transitórias (p. ex. reprocessando mensagens). A arquitetura SAGA deve oferecer tolerância a falhas por meio de lógica compensatória e garantir consistência eventual dos dados.
* **Desempenho Moderado:** não se espera grande carga ou resposta em tempo real; entretanto, cada serviço deve responder de forma razoavelmente rápida às chamadas REST e processar eventos de mensageria sem bloqueios desnecessários.

Cada requisito aqui descrito orienta a implementação de forma a atingir os objetivos educacionais do sistema, privilegiando clareza de arquitetura e fluxos funcionais demonstrativos.
