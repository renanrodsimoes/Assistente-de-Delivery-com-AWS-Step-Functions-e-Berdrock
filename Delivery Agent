{
  "Comment": "Assistente de Delivery - Step Functions + Amazon Bedrock (Nova Micro): recomendacao antes do pedido, confirmacao, validacao condicional, retry/catch e notificacao via SNS",
  "QueryLanguage": "JSONata",
  "StartAt": "Recomendar Itens para o Jantar",
  "States": {
    "Recomendar Itens para o Jantar": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Arguments": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0",
        "Body": {
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "text": "O cliente disse: 'Estou pensando em fazer um jantar italiano hoje, principalmente massas. Você pode me recomendar alguns itens que combinem bem, tipo um tipo de massa, um vinho, uma entrada ou uma sobremesa?' Dê recomendações específicas e apetitosas de pratos e itens complementares para um jantar italiano, de forma cordial e objetiva."
                }
              ]
            }
          ],
          "inferenceConfig": {
            "maxTokens": 500
          }
        },
        "ContentType": "application/json",
        "Accept": "application/json"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Bedrock.ThrottlingException",
            "Bedrock.ModelTimeoutException",
            "Bedrock.ServiceUnavailableException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 4,
          "BackoffRate": 2
        },
        {
          "ErrorEquals": [
            "States.TaskFailed"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "Falha no Processamento"
        }
      ],
      "Assign": {
        "conversation_history": "{% [{\"role\": \"user\", \"content\": [{\"text\": \"O cliente disse: 'Estou pensando em fazer um jantar italiano hoje, principalmente massas. Você pode me recomendar alguns itens que combinem bem, tipo um tipo de massa, um vinho, uma entrada ou uma sobremesa?' Dê recomendações específicas e apetitosas de pratos e itens complementares para um jantar italiano, de forma cordial e objetiva.\"}]}, {\"role\": \"assistant\", \"content\": $states.result.Body.output.message.content}] %}"
      },
      "Next": "Confirmar Dados e Pedido"
    },
    "Confirmar Dados e Pedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Arguments": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0",
        "Body": {
          "messages": "{% $append($conversation_history, [{\"role\": \"user\", \"content\": [{\"text\": \"O cliente respondeu: 'Adorei as sugestões! Vou querer um Fettuccine Alfredo, uma taça de vinho branco e um tiramisu de sobremesa. Para entrega na Rua das Palmeiras, 123, apartamento 45. Vou pagar com cartão de crédito na entrega.' Confirme os itens escolhidos pelo cliente e verifique se o endereço de entrega e a forma de pagamento foram informados. Se as duas informações já foram dadas, finalize sua resposta escrevendo exatamente a palavra PEDIDO_COMPLETO em uma linha separada. Caso falte alguma informação, pergunte educadamente o que falta e finalize com a palavra PEDIDO_INCOMPLETO em uma linha separada.\"}]}]) %}",
          "inferenceConfig": {
            "maxTokens": 600
          }
        },
        "ContentType": "application/json",
        "Accept": "application/json"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Bedrock.ThrottlingException",
            "Bedrock.ModelTimeoutException",
            "Bedrock.ServiceUnavailableException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 4,
          "BackoffRate": 2
        },
        {
          "ErrorEquals": [
            "States.TaskFailed"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "Falha no Processamento"
        }
      ],
      "Assign": {
        "conversation_history": "{% $append($conversation_history, [{\"role\": \"user\", \"content\": [{\"text\": \"O cliente respondeu: 'Adorei as sugestões! Vou querer um Fettuccine Alfredo, uma taça de vinho branco e um tiramisu de sobremesa. Para entrega na Rua das Palmeiras, 123, apartamento 45. Vou pagar com cartão de crédito na entrega.' Confirme os itens escolhidos pelo cliente e verifique se o endereço de entrega e a forma de pagamento foram informados. Se as duas informações já foram dadas, finalize sua resposta escrevendo exatamente a palavra PEDIDO_COMPLETO em uma linha separada. Caso falte alguma informação, pergunte educadamente o que falta e finalize com a palavra PEDIDO_INCOMPLETO em uma linha separada.\"}]}, {\"role\": \"assistant\", \"content\": $states.result.Body.output.message.content}]) %}",
        "validacao_texto": "{% $states.result.Body.output.message.content[0].text %}"
      },
      "Next": "Pedido Completo?"
    },
    "Pedido Completo?": {
      "Type": "Choice",
      "Choices": [
        {
          "Condition": "{% $contains($validacao_texto, \"PEDIDO_COMPLETO\") %}",
          "Next": "Confirmar Pedido e Status"
        }
      ],
      "Default": "Solicitar Informacoes Faltantes"
    },
    "Solicitar Informacoes Faltantes": {
      "Type": "Succeed",
      "Comment": "Pedido ainda incompleto - em uma versão interativa real, o fluxo voltaria a perguntar ao cliente as informações faltantes."
    },
    "Confirmar Pedido e Status": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Arguments": {
        "ModelId": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0",
        "Body": {
          "messages": "{% $append($conversation_history, [{\"role\": \"user\", \"content\": [{\"text\": \"O cliente confirmou o pedido e o pagamento foi aprovado. Gere uma mensagem final informando que o pedido está sendo preparado e dê uma estimativa de tempo de entrega.\"}]}]) %}",
          "inferenceConfig": {
            "maxTokens": 500
          }
        },
        "ContentType": "application/json",
        "Accept": "application/json"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Bedrock.ThrottlingException",
            "Bedrock.ModelTimeoutException",
            "Bedrock.ServiceUnavailableException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 4,
          "BackoffRate": 2
        },
        {
          "ErrorEquals": [
            "States.TaskFailed"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "Falha no Processamento"
        }
      ],
      "Assign": {
        "resposta_final": "{% $states.result.Body.output.message.content[0].text %}"
      },
      "Next": "Notificar Cliente"
    },
    "Notificar Cliente": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Arguments": {
       "TopicArn": "arn:aws:sns:us-east-1:155534475758:MyDeliveryAgent-Notificacoes",
        "Subject": "Pedido confirmado - MyDeliveryAgent",
        "Message": "{% 'Pedido confirmado! ' & $resposta_final %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "States.TaskFailed"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "Falha no Processamento"
        }
      ],
      "Output": {
        "resposta_final": "{% $resposta_final %}",
        "historico_completo": "{% $conversation_history %}"
      },
      "End": true
    },
    "Falha no Processamento": {
      "Type": "Fail",
      "Error": "FalhaNoAssistenteDeDelivery",
      "Cause": "Ocorreu um erro ao processar o pedido após as tentativas de retry."
    }
  }
}
