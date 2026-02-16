# TASK 12: Configurar SNS e Notificações

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** ALTA  
**Estimativa:** 2 dias  
**Dependências:** TASK 11 (QuickSight Dashboard)

---

## Objetivo

Configurar Amazon SNS para enviar notificações de sucesso e falha via Slack e Email, conforme especificação do PRD.

---

## Requisitos

- SNS Topic criado
- Webhook Slack configurado (se aplicável)
- Email configurado (SES ou SMTP)

---

## Passo a Passo Detalhado

### 1. Criar SNS Topic

**Passos:**

1.1. Criar tópico SNS:
```bash
aws sns create-topic \
  --name reconciliation-alerts \
  --attributes '{
    "DisplayName": "Conciliação Financeira - Alertas"
  }'
```

1.2. Obter ARN do tópico:
```bash
aws sns get-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

---

### 2. Configurar Subscription para Email

**Passos:**

2.1. Criar subscription para email:
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --protocol email \
  --notification-endpoint conciliacao-alerts@empresa.com
```

2.2. Confirmar subscription (verificar email e clicar no link de confirmação)

2.3. Adicionar múltiplos emails:
```bash
# Equipe de Operações
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --protocol email \
  --notification-endpoint ops-conciliacao@empresa.com

# Product Owner
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --protocol email \
  --notification-endpoint po-conciliacao@empresa.com

# Área de Negócio
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --protocol email \
  --notification-endpoint negocio-conciliacao@empresa.com
```

---

### 3. Configurar Subscription para Slack

**Passos:**

3.1. Criar Lambda para enviar para Slack:
```python
import json
import urllib3
import os

http = urllib3.PoolManager()

SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']
QUICKSIGHT_DASHBOARD_URL = os.environ['QUICKSIGHT_DASHBOARD_URL']

def lambda_handler(event, context):
    """
    Lambda que recebe mensagem SNS e envia para Slack
    """
    # Parse SNS message
    sns_message = json.loads(event['Records'][0]['Sns']['Message'])
    
    severity = sns_message.get('severity', 'INFO')
    job_id = sns_message.get('job_id', 'UNKNOWN')
    message = sns_message.get('message', '')
    status = sns_message.get('status', '')
    
    # Determinar cor do Slack baseado na severidade
    color_map = {
        'CRITICAL': '#FF0000',  # Vermelho
        'HIGH': '#FF9900',      # Laranja
        'MEDIUM': '#FFCC00',    # Amarelo
        'LOW': '#00CC00',       # Verde
        'INFO': '#0066CC'       # Azul
    }
    color = color_map.get(severity, '#808080')
    
    # Construir mensagem Slack
    slack_message = {
        "attachments": [
            {
                "color": color,
                "title": f"Conciliação Financeira - {severity}",
                "fields": [
                    {
                        "title": "Job ID",
                        "value": job_id,
                        "short": True
                    },
                    {
                        "title": "Status",
                        "value": status,
                        "short": True
                    },
                    {
                        "title": "Mensagem",
                        "value": message,
                        "short": False
                    }
                ],
                "footer": "Sistema de Conciliação Financeira",
                "ts": int(context.aws_request_id[:10])  # Timestamp aproximado
            }
        ]
    }
    
    # Adicionar link para dashboard se sucesso
    if status == 'SUCCESS' and QUICKSIGHT_DASHBOARD_URL:
        slack_message["attachments"][0]["actions"] = [
            {
                "type": "button",
                "text": "Ver Dashboard",
                "url": QUICKSIGHT_DASHBOARD_URL,
                "style": "primary"
            }
        ]
    
    # Adicionar detalhes se disponíveis
    if 'details' in sns_message:
        slack_message["attachments"][0]["fields"].append({
            "title": "Detalhes",
            "value": json.dumps(sns_message['details'], indent=2),
            "short": False
        })
    
    # Enviar para Slack
    try:
        response = http.request(
            'POST',
            SLACK_WEBHOOK_URL,
            body=json.dumps(slack_message),
            headers={'Content-Type': 'application/json'}
        )
        
        if response.status == 200:
            print(f"Mensagem enviada para Slack com sucesso")
        else:
            print(f"Erro ao enviar para Slack: {response.status}")
            
    except Exception as e:
        print(f"Erro ao enviar para Slack: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Mensagem processada')
    }
```

3.2. Criar Lambda function:
```bash
zip lambda_slack_notification.zip lambda_slack_notification.py

aws lambda create-function \
  --function-name reconciliation-slack-notification \
  --runtime python3.11 \
  --role arn:aws:iam::ACCOUNT_ID:role/ReconciliationLambdaRole \
  --handler lambda_slack_notification.lambda_handler \
  --zip-file fileb://lambda_slack_notification.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables='{
    "SLACK_WEBHOOK_URL": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
    "QUICKSIGHT_DASHBOARD_URL": "https://us-east-1.quicksight.aws.amazon.com/sn/dashboards/DASHBOARD_ID"
  }'
```

3.3. Criar subscription SNS → Lambda:
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:ACCOUNT_ID:function:reconciliation-slack-notification
```

3.4. Adicionar permissão para SNS invocar Lambda:
```bash
aws lambda add-permission \
  --function-name reconciliation-slack-notification \
  --statement-id allow-sns \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

---

### 4. Template de Notificação

**Conforme PRD - Seção 5.1.3:**

**Template de Notificação de Sucesso:**
```json
{
  "severity": "INFO",
  "job_id": "JOB-20260216-001234",
  "status": "SUCCESS",
  "timestamp": "2026-02-16T08:20:00Z",
  "message": "Conciliação concluída com sucesso",
  "results": {
    "total_lines": 1000000,
    "valid_lines": 999500,
    "invalid_lines": 500,
    "total_divergences": 1500,
    "tipo1": 500,
    "tipo2": 400,
    "tipo3_valor": 500,
    "tipo3_data": 100,
    "processing_time_seconds": 1200
  },
  "dashboard_url": "https://quicksight.aws.amazon.com/...",
  "output_location": "s3://reconciliation-output/2026-02-16/divergencias.parquet"
}
```

**Template de Notificação de Falha:**
```json
{
  "severity": "CRITICAL",
  "job_id": "JOB-20260216-001234",
  "status": "FAILED",
  "timestamp": "2026-02-16T08:15:30Z",
  "message": "Falha no processamento de conciliação",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Campo 'valor' ausente no CSV",
    "stage": "validation"
  },
  "details": {
    "bucket": "reconciliation-ingress",
    "key": "2026-02-16/input.csv",
    "error_lines": [
      "Linha 150: 12345;ORD-98765;João Silva;ATV-001;2026-02-15;",
      "Linha 151: 12345;ORD-98766;Maria Santos;ATV-002;2026-02-15;"
    ]
  },
  "logs_url": "https://logs.empresa.com/jobs/JOB-20260216-001234",
  "action_required": "Verificar formato do CSV com fornecedor e reprocessar após correção"
}
```

---

### 5. Integrar com Step Functions

**Já configurado na TASK 04, mas validar:**

- Notificação de sucesso após preparação QuickSight
- Notificação de falha em qualquer etapa
- Incluir link para dashboard QuickSight nas notificações de sucesso

---

## Validação

### Checklist de Validação:

- [ ] SNS Topic criado
- [ ] Subscriptions de email criadas e confirmadas
- [ ] Lambda para Slack criada
- [ ] Webhook Slack configurado
- [ ] Permissões configuradas (SNS → Lambda)
- [ ] Teste: Enviar notificação de sucesso
- [ ] Teste: Enviar notificação de falha
- [ ] Verificar que emails são recebidos
- [ ] Verificar que mensagens aparecem no Slack
- [ ] Validar formato das mensagens

### Comandos de Validação:

```bash
# Listar subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts

# Testar publicação manual
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts \
  --message '{
    "severity": "INFO",
    "job_id": "TEST-001",
    "status": "SUCCESS",
    "message": "Teste de notificação"
  }' \
  --subject "Teste de Notificação"

# Verificar logs da Lambda
aws logs tail /aws/lambda/reconciliation-slack-notification --follow
```

---

## Notas Técnicas

- **Webhook Slack:** Obter URL do webhook no Slack (Settings → Incoming Webhooks)
- **Email SES:** Se usar SES, verificar domínio e configurar adequadamente
- **Formato:** Usar JSON estruturado para facilitar parsing
- **Severidade:** Mapear severidade para cores/formatação no Slack

---

## Próximas Tasks

- **TASK 13:** Implementar Idempotência e Hash (já parcialmente implementado)
- **TASK 14:** Testes End-to-End

---

## Referências

- PRD - Seção 5.1 (Notificação de Falha)
- PRD - Seção 5.1.3 (Conteúdo Mínimo da Notificação)
- AWS SNS Documentation: https://docs.aws.amazon.com/sns/
