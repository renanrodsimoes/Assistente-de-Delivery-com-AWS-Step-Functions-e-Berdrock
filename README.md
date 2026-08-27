# MyDeliveryAgent

Assistente virtual de delivery construído com **AWS Step Functions** e **Amazon Bedrock**, desenvolvido como projeto do desafio prático da DIO *"Criando um Assistente de Delivery com AWS Step Functions e Bedrock"*.

O agente conduz o cliente por um fluxo conversacional completo: recomendação de itens, confirmação do pedido e dos dados de entrega/pagamento, validação condicional, confirmação final com status do pedido, e notificação automática.

## Arquitetura

O fluxo é orquestrado inteiramente por uma **state machine do Step Functions** (linguagem JSONata), que chama o **Amazon Bedrock** em múltiplas etapas mantendo o histórico da conversa entre elas, e finaliza publicando uma notificação via **Amazon SNS**.

```
Início
  │
  ▼
[Recomendar Itens para o Jantar]  ── Bedrock (Nova Micro)
  │   sugere pratos e itens complementares
  ▼
[Confirmar Dados e Pedido]        ── Bedrock (Nova Micro)
  │   confirma itens escolhidos, endereço e pagamento
  ▼
[Pedido Completo?]                ── Choice
  │                       │
  │ sim                   │ não
  ▼                       ▼
[Confirmar Pedido e Status]   [Solicitar Informações Faltantes]
  │   Bedrock (Nova Micro)         (encerra o fluxo)
  ▼
[Notificar Cliente]           ── Amazon SNS (publish)
  │
  ▼
 Fim

  Qualquer falha nas etapas acima → [Falha no Processamento]
  (após as tentativas de retry configuradas)
```

### Etapas da state machine

| Etapa | Tipo | O que faz |
|---|---|---|
| `Recomendar Itens para o Jantar` | Task (Bedrock InvokeModel) | Recebe o interesse do cliente (ex: jantar italiano com massas) e sugere pratos/itens que combinam |
| `Confirmar Dados e Pedido` | Task (Bedrock InvokeModel) | Confirma os itens escolhidos e verifica se endereço de entrega e forma de pagamento foram informados |
| `Pedido Completo?` | Choice | Lê o marcador `PEDIDO_COMPLETO` / `PEDIDO_INCOMPLETO` na resposta do modelo e decide o próximo passo |
| `Solicitar Informações Faltantes` | Succeed | Encerra o fluxo indicando que ainda faltam dados do pedido |
| `Confirmar Pedido e Status` | Task (Bedrock InvokeModel) | Gera a mensagem de confirmação do pedido com estimativa de entrega |
| `Notificar Cliente` | Task (SNS Publish) | Publica a confirmação do pedido em um tópico SNS |
| `Falha no Processamento` | Fail | Estado de erro, acionado quando alguma etapa falha mesmo após as tentativas de retry |

Cada etapa de Bedrock possui **Retry** (com backoff exponencial, para erros de *throttling* e timeout) e **Catch** (redirecionando qualquer falha não recuperável para `Falha no Processamento`), garantindo resiliência no pipeline.

## Modelo utilizado

- **Modelo:** `amazon.nova-micro-v1:0` (Amazon Nova Micro)
- **Região:** `us-east-1` (US East — N. Virginia)

> **Por que Nova Micro em vez de Claude/Anthropic?**
> Contas AWS no **Free Plan** (modelo de conta gratuita adotado pela AWS a partir de julho/2025) bloqueiam assinaturas via **AWS Marketplace** — e é assim que os modelos da Anthropic (Claude) são distribuídos dentro do Bedrock. O Amazon Nova, por ser um modelo nativo da própria AWS, não passa pelo Marketplace e funciona normalmente em contas gratuitas, sem necessidade de cartão de crédito ou upgrade de plano.

## Pré-requisitos

- Conta AWS com acesso ao **Amazon Bedrock** na região `us-east-1`
- Permissão de IAM para criar/editar state machines no **Step Functions**
- Um tópico no **Amazon SNS** para receber as notificações de pedido confirmado

## Como implantar

1. No console do **Step Functions** (região `us-east-1`), crie uma nova state machine em branco.
2. Na aba **Código**, cole o conteúdo de [`state_machine.json`](./state_machine.json).
3. Crie um tópico no **Amazon SNS** (ex: `MyDeliveryAgent-Notificacoes`) e, opcionalmente, inscreva um e-mail para receber as notificações (é preciso confirmar a inscrição pelo link enviado por e-mail).
4. Substitua o valor de `TopicArn` no JSON pelo ARN real do tópico criado.
5. Garanta que a *execution role* da state machine tenha permissão para `bedrock:InvokeModel` e `sns:Publish`.
6. Salve e execute a state machine com o input `{}`.

## Testando

Na aba **Entrada e saída de execução** de cada execução, é possível acompanhar:
- as recomendações geradas na 1ª etapa;
- a confirmação dos dados do pedido na 2ª etapa;
- o caminho escolhido pelo `Choice` (`PEDIDO_COMPLETO` ou `PEDIDO_INCOMPLETO`);
- a mensagem final de status do pedido (`resposta_final`) e o histórico completo da conversa (`historico_completo`).

## Possíveis evoluções

- Tornar o fluxo verdadeiramente interativo (entrada real do cliente a cada etapa, em vez de mensagens de exemplo fixas), por exemplo acoplando uma função Lambda com Function URL como camada de chat.
- Adicionar uma etapa real de integração com um serviço de pagamento.
- Persistir o histórico de pedidos em um banco de dados (ex: DynamoDB).
- Usar Amazon CloudWatch Logs / X-Ray para observabilidade da execução.

## Tecnologias

- AWS Step Functions (Standard Workflow, linguagem JSONata)
- Amazon Bedrock — Amazon Nova Micro
- Amazon SNS
- AWS IAM
