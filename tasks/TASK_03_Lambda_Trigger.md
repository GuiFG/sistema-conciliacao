# TASK 03: Implementar Lambda de Trigger

**Sprint:** 1-2 (Infraestrutura Base)  
**Prioridade:** CRÍTICA  
**Estimativa:** 1-2 dias  
**Dependências:** TASK 01 (S3 buckets), TASK 02 (IAM roles), TASK 04 (Step Functions - para invocar)

---

## Objetivo

Implementar função Lambda que é acionada quando um arquivo CSV é enviado ao bucket S3 de ingress. A Lambda deve validar o formato básico do arquivo e iniciar a orquestração via Step Functions.

---

## Requisitos

- Bucket S3 `reconciliation-ingress` criado
- IAM Role `ReconciliationLambdaRole` criada
- Step Functions State Machine criada (TASK 04)
- EventBridge Rule configurada (TASK 01)

---

## Passo a Passo Detalhado

### 1. Criar Função Lambda

**Passos:**

1.1. Criar arquivo de código Python (`lambda_trigger.py`):
```python
import json
import boto3
import csv
import hashlib
from datetime import datetime
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')
stepfunctions_client = boto3.client('stepfunctions')
sns_client = boto3.client('sns')

# Nome da Step Functions State Machine (será configurado na TASK 04)
STEP_FUNCTION_ARN = 'arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline'
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts'

def lambda_handler(event, context):
    """
    Lambda handler acionado por EventBridge quando CSV é enviado ao S3
    """
    try:
        # Extrair informações do evento S3
        bucket_name = event['Records'][0]['s3']['bucket']['name']
        object_key = event['Records'][0]['s3']['object']['key']
        
        # Validar que é um arquivo CSV
        if not object_key.endswith('.csv'):
            raise ValueError(f'Arquivo {object_key} não é um CSV')
        
        # Gerar Job ID único
        job_id = generate_job_id()
        
        # Validar formato básico do CSV
        validation_result = validate_csv_format(bucket_name, object_key)
        
        if not validation_result['valid']:
            # Notificar erro de validação
            send_notification(
                severity='CRITICAL',
                job_id=job_id,
                message=f'Erro na validação do CSV: {validation_result["error"]}',
                details={
                    'bucket': bucket_name,
                    'key': object_key,
                    'error': validation_result['error']
                }
            )
            return {
                'statusCode': 400,
                'body': json.dumps({
                    'error': 'CSV inválido',
                    'details': validation_result['error']
                })
            }
        
        # Calcular hash do arquivo para idempotência
        file_hash = calculate_file_hash(bucket_name, object_key)
        
        # Preparar input para Step Functions
        step_function_input = {
            'job_id': job_id,
            'bucket_name': bucket_name,
            'object_key': object_key,
            'file_hash': file_hash,
            'timestamp': datetime.utcnow().isoformat(),
            'validation_result': validation_result
        }
        
        # Iniciar execução da Step Functions
        execution_response = stepfunctions_client.start_execution(
            stateMachineArn=STEP_FUNCTION_ARN,
            name=f'{job_id}',
            input=json.dumps(step_function_input)
        )
        
        # Log de sucesso
        print(f'Step Functions execution started: {execution_response["executionArn"]}')
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'job_id': job_id,
                'execution_arn': execution_response['executionArn'],
                'message': 'Processamento iniciado com sucesso'
            })
        }
        
    except Exception as e:
        # Log de erro
        print(f'Erro no Lambda trigger: {str(e)}')
        
        # Notificar erro crítico
        send_notification(
            severity='CRITICAL',
            job_id=job_id if 'job_id' in locals() else 'UNKNOWN',
            message=f'Erro crítico no Lambda trigger: {str(e)}',
            details={
                'error': str(e),
                'event': event
            }
        )
        
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': 'Erro interno',
                'message': str(e)
            })
        }


def validate_csv_format(bucket_name, object_key):
    """
    Valida formato básico do CSV:
    - Verifica cabeçalho obrigatório
    - Verifica que arquivo não está vazio
    - Verifica formato de separador (ponto-e-vírgula)
    """
    try:
        # Baixar apenas primeiras linhas do CSV (head)
        response = s3_client.get_object(
            Bucket=bucket_name,
            Key=object_key,
            Range='bytes=0-1024'  # Primeiros 1KB
        )
        
        content = response['Body'].read().decode('utf-8')
        lines = content.split('\n')
        
        # Verificar que arquivo não está vazio
        if len(lines) < 1:
            return {
                'valid': False,
                'error': 'Arquivo CSV está vazio'
            }
        
        # Verificar cabeçalho
        header = lines[0].strip()
        expected_header = 'fornecedor_id;ordem_id;nome;ativo_id;data;valor'
        
        if header != expected_header:
            return {
                'valid': False,
                'error': f'Cabeçalho inválido. Esperado: {expected_header}, Recebido: {header}'
            }
        
        # Verificar que há pelo menos uma linha de dados
        if len(lines) < 2:
            return {
                'valid': False,
                'error': 'CSV não contém linhas de dados'
            }
        
        # Verificar formato da primeira linha de dados (se disponível)
        if len(lines) > 1:
            first_data_line = lines[1].strip()
            if not first_data_line:
                return {
                    'valid': False,
                    'error': 'Primeira linha de dados está vazia'
                }
            
            # Verificar número de campos (deve ter 6 campos separados por ;)
            fields = first_data_line.split(';')
            if len(fields) != 6:
                return {
                    'valid': False,
                    'error': f'Número incorreto de campos. Esperado: 6, Recebido: {len(fields)}'
                }
        
        return {
            'valid': True,
            'header': header,
            'estimated_lines': len(lines) - 1  # Excluir cabeçalho
        }
        
    except ClientError as e:
        return {
            'valid': False,
            'error': f'Erro ao acessar arquivo S3: {str(e)}'
        }
    except Exception as e:
        return {
            'valid': False,
            'error': f'Erro na validação: {str(e)}'
        }


def calculate_file_hash(bucket_name, object_key):
    """
    Calcula hash SHA-256 do arquivo para idempotência
    """
    try:
        response = s3_client.get_object(Bucket=bucket_name, Key=object_key)
        file_content = response['Body'].read()
        return hashlib.sha256(file_content).hexdigest()
    except Exception as e:
        print(f'Erro ao calcular hash: {str(e)}')
        return None


def generate_job_id():
    """
    Gera Job ID único no formato: JOB-YYYYMMDD-HHMMSS-RANDOM
    """
    timestamp = datetime.utcnow().strftime('%Y%m%d-%H%M%S')
    random_suffix = hashlib.md5(str(datetime.utcnow().timestamp()).encode()).hexdigest()[:6]
    return f'JOB-{timestamp}-{random_suffix}'


def send_notification(severity, job_id, message, details=None):
    """
    Envia notificação via SNS
    """
    try:
        notification = {
            'severity': severity,
            'job_id': job_id,
            'timestamp': datetime.utcnow().isoformat(),
            'message': message,
            'details': details or {}
        }
        
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f'[{severity}] Conciliação - {job_id}',
            Message=json.dumps(notification, indent=2)
        )
    except Exception as e:
        print(f'Erro ao enviar notificação: {str(e)}')
```

1.2. Criar arquivo de requirements (`requirements.txt`):
```
boto3>=1.26.0
```

1.3. Criar pacote ZIP para deploy:
```bash
zip lambda_trigger.zip lambda_trigger.py
```

1.4. Criar função Lambda via AWS CLI:
```bash
aws lambda create-function \
  --function-name reconciliation-trigger \
  --runtime python3.11 \
  --role arn:aws:iam::ACCOUNT_ID:role/ReconciliationLambdaRole \
  --handler lambda_trigger.lambda_handler \
  --zip-file fileb://lambda_trigger.zip \
  --timeout 60 \
  --memory-size 256 \
  --environment Variables='{
    "STEP_FUNCTION_ARN": "arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline",
    "SNS_TOPIC_ARN": "arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts"
  }'
```

---

### 2. Configurar EventBridge para Acionar Lambda

**Passos:**

2.1. Adicionar Lambda como target da regra EventBridge:
```bash
aws events put-targets \
  --rule reconciliation-csv-uploaded \
  --targets "Id=1,Arn=arn:aws:lambda:us-east-1:ACCOUNT_ID:function:reconciliation-trigger"
```

2.2. Adicionar permissão para EventBridge invocar Lambda:
```bash
aws lambda add-permission \
  --function-name reconciliation-trigger \
  --statement-id allow-eventbridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:ACCOUNT_ID:rule/reconciliation-csv-uploaded
```

---

### 3. Configurar CloudWatch Logs

**Passos:**

3.1. Criar Log Group:
```bash
aws logs create-log-group \
  --log-group-name /aws/lambda/reconciliation-trigger
```

3.2. Configurar retenção (90 dias):
```bash
aws logs put-retention-policy \
  --log-group-name /aws/lambda/reconciliation-trigger \
  --retention-in-days 90
```

---

### 4. Adicionar Métricas Customizadas

**Passos:**

4.1. Adicionar código para publicar métricas no CloudWatch:
```python
cloudwatch = boto3.client('cloudwatch')

def publish_metric(metric_name, value, unit='Count', dimensions=None):
    """
    Publica métrica customizada no CloudWatch
    """
    try:
        cloudwatch.put_metric_data(
            Namespace='Reconciliation/Lambda',
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': value,
                    'Unit': unit,
                    'Dimensions': dimensions or []
                }
            ]
        )
    except Exception as e:
        print(f'Erro ao publicar métrica: {str(e)}')
```

4.2. Adicionar chamadas de métricas no handler:
```python
# No início do lambda_handler, após validar CSV:
publish_metric('CSVUploaded', 1, dimensions=[
    {'Name': 'Bucket', 'Value': bucket_name}
])

# Após iniciar Step Functions:
publish_metric('PipelineStarted', 1, dimensions=[
    {'Name': 'JobId', 'Value': job_id}
])
```

---

## Validação

### Checklist de Validação:

- [ ] Função Lambda criada com código correto
- [ ] IAM Role anexada corretamente
- [ ] EventBridge configurado para acionar Lambda
- [ ] Permissões configuradas (EventBridge → Lambda)
- [ ] CloudWatch Logs configurado
- [ ] Variáveis de ambiente configuradas (STEP_FUNCTION_ARN, SNS_TOPIC_ARN)
- [ ] Teste manual: Upload CSV no bucket ingress
- [ ] Verificar logs no CloudWatch após teste
- [ ] Verificar que Step Functions foi iniciada (após TASK 04)

### Comandos de Validação:

```bash
# Verificar função Lambda criada
aws lambda get-function --function-name reconciliation-trigger

# Verificar configuração
aws lambda get-function-configuration --function-name reconciliation-trigger

# Verificar permissões
aws lambda get-policy --function-name reconciliation-trigger

# Testar função manualmente (simular evento S3)
aws lambda invoke \
  --function-name reconciliation-trigger \
  --payload file://test-event.json \
  response.json

# Ver logs recentes
aws logs tail /aws/lambda/reconciliation-trigger --follow

# Teste real: Upload CSV
echo "fornecedor_id;ordem_id;nome;ativo_id;data;valor
12345;ORD-001;Teste;ATV-001;2026-02-16;100.00" > test.csv
aws s3 cp test.csv s3://reconciliation-ingress/2026-02-16/test.csv
```

### Arquivo de Teste (`test-event.json`):
```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "reconciliation-ingress"
        },
        "object": {
          "key": "2026-02-16/test.csv"
        }
      }
    }
  ]
}
```

---

## Notas Técnicas

- **Timeout:** Configurar timeout adequado (60s é suficiente para validação básica)
- **Memória:** 256MB é suficiente para validação de cabeçalho
- **Idempotência:** Hash do arquivo garante que mesmo arquivo não seja processado duas vezes
- **Validação Básica:** Esta Lambda faz apenas validação básica. Validação completa será feita no Glue Job

---

## Próximas Tasks

- **TASK 04:** Configurar Step Functions (requer Lambda criada)
- **TASK 06:** Implementar Glue Job de Validação (validação completa)

---

## Referências

- Documento Técnico - Seção 3.2 (Componentes da Arquitetura)
- AWS Lambda Documentation: https://docs.aws.amazon.com/lambda/
