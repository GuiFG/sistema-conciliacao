# TASK 04: Configurar Step Functions (Orquestração)

**Sprint:** 1-2 (Infraestrutura Base)  
**Prioridade:** CRÍTICA  
**Estimativa:** 2-3 dias  
**Dependências:** TASK 02 (IAM roles), TASK 03 (Lambda trigger)

---

## Objetivo

Criar Step Functions State Machine que orquestra todo o fluxo de processamento: Validação → Conciliação → Geração de Relatórios → Atualização QuickSight → Notificação.

---

## Requisitos

- IAM Role `ReconciliationStepFunctionsRole` criada
- Glue Jobs criados (TASK 06, 07, 09)
- SNS Topic criado (TASK 12)

---

## Passo a Passo Detalhado

### 1. Criar State Machine Definition (JSON)

**Arquivo:** `step-functions-definition.json`

```json
{
  "Comment": "Pipeline de Conciliação Financeira - MVP",
  "StartAt": "ValidateCSV",
  "States": {
    "ValidateCSV": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "reconciliation-validation",
        "Arguments": {
          "--job_id": "$.job_id",
          "--bucket_name": "$.bucket_name",
          "--object_key": "$.object_key",
          "--file_hash": "$.file_hash"
        }
      },
      "ResultPath": "$.validation_result",
      "Next": "CheckValidationResult",
      "Catch": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "ResultPath": "$.error",
          "Next": "NotifyFailure"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["Glue.ConcurrentRunsExceededException", "Glue.ServiceException"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ]
    },
    "CheckValidationResult": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validation_result.JobRunState",
          "StringEquals": "SUCCEEDED",
          "Next": "ReconcileData"
        }
      ],
      "Default": "NotifyFailure"
    },
    "ReconcileData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "reconciliation-reconciliation",
        "Arguments": {
          "--job_id": "$.job_id",
          "--validated_data_path": "$.validation_result.OutputPath"
        }
      },
      "ResultPath": "$.reconciliation_result",
      "Next": "PrepareQuickSight",
      "Catch": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "ResultPath": "$.error",
          "Next": "NotifyFailure"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["Glue.ConcurrentRunsExceededException", "Glue.ServiceException"],
          "IntervalSeconds": 60,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "TimeoutSeconds": 1200
    },
    "PrepareQuickSight": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "reconciliation-quicksight-prep",
        "Arguments": {
          "--job_id": "$.job_id",
          "--divergences_path": "$.reconciliation_result.OutputPath"
        }
      },
      "ResultPath": "$.quicksight_result",
      "Next": "NotifySuccess",
      "Catch": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "ResultPath": "$.error",
          "Next": "NotifyFailure"
        }
      ],
      "Retry": [
        {
          "ErrorEquals": ["Glue.ConcurrentRunsExceededException", "Glue.ServiceException"],
          "IntervalSeconds": 30,
          "MaxAttempts": 2,
          "BackoffRate": 2.0
        }
      ]
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts",
        "Message": {
          "severity": "INFO",
          "job_id": "$.job_id",
          "status": "SUCCESS",
          "timestamp": "$$.State.EnteredTime",
          "message": "Conciliação concluída com sucesso",
          "results": {
            "validation": "$.validation_result",
            "reconciliation": "$.reconciliation_result",
            "quicksight": "$.quicksight_result"
          }
        }
      },
      "Next": "Success"
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts",
        "Message": {
          "severity": "CRITICAL",
          "job_id": "$.job_id",
          "status": "FAILED",
          "timestamp": "$$.State.EnteredTime",
          "error": "$.error",
          "message": "Falha no processamento de conciliação"
        }
      },
      "Next": "Failure"
    },
    "Success": {
      "Type": "Succeed"
    },
    "Failure": {
      "Type": "Fail",
      "Error": "PipelineFailed",
      "Cause": "Processamento de conciliação falhou"
    }
  }
}
```

---

### 2. Criar State Machine com Timeout Global

**Passos:**

2.1. Criar State Machine:
```bash
aws stepfunctions create-state-machine \
  --name reconciliation-pipeline \
  --definition file://step-functions-definition.json \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/ReconciliationStepFunctionsRole \
  --logging-configuration '{
    "level": "ALL",
    "includeExecutionData": true,
    "destinations": [
      {
        "cloudWatchLogsLogGroup": {
          "logGroupArn": "arn:aws:logs:us-east-1:ACCOUNT_ID:log-group:/aws/stepfunctions/reconciliation"
        }
      }
    ]
  }' \
  --tracing-configuration '{
    "enabled": true
  }'
```

2.2. Configurar CloudWatch Logs para Step Functions:
```bash
aws logs create-log-group \
  --log-group-name /aws/stepfunctions/reconciliation

aws logs put-retention-policy \
  --log-group-name /aws/stepfunctions/reconciliation \
  --retention-in-days 90
```

---

### 3. Implementar Timeout de 20 Minutos

**Objetivo:** Garantir que processamento não exceda 20 minutos (SLA crítico)

**Passos:**

3.1. Adicionar estado de timeout no início da State Machine:
```json
{
  "StartAt": "StartTimer",
  "States": {
    "StartTimer": {
      "Type": "Pass",
      "Parameters": {
        "start_time": "$$.State.EnteredTime",
        "timeout_seconds": 1200
      },
      "Next": "ValidateCSV"
    },
    "ValidateCSV": {
      ...
      "Next": "CheckTimeout"
    },
    "CheckTimeout": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.elapsed_seconds",
          "NumericGreaterThan": 1200,
          "Next": "TimeoutFailure"
        }
      ],
      "Default": "ReconcileData"
    },
    "TimeoutFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "reconciliation-timeout-handler",
        "Payload": {
          "job_id": "$.job_id",
          "reason": "Timeout de 20 minutos excedido"
        }
      },
      "Next": "NotifyFailure"
    }
  }
}
```

3.2. Alternativa mais simples: Configurar timeout na execução:
```bash
# Ao iniciar execução, definir timeout
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline \
  --name JOB-20260216-001234 \
  --input '{"job_id": "JOB-20260216-001234", ...}' \
  --timeout-seconds 1200
```

---

### 4. Adicionar Checkpoint em Caso de Falha

**Passos:**

4.1. Criar Lambda para salvar checkpoint:
```python
import json
import boto3
from datetime import datetime

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    """
    Salva checkpoint em S3 em caso de falha/timeout
    """
    job_id = event['job_id']
    checkpoint_data = {
        'job_id': job_id,
        'timestamp': datetime.utcnow().isoformat(),
        'status': 'FAILED',
        'state': event.get('state', 'UNKNOWN'),
        'error': event.get('error', {}),
        'progress': event.get('progress', {})
    }
    
    checkpoint_key = f'{job_id}/checkpoint.json'
    
    s3_client.put_object(
        Bucket='reconciliation-checkpoints',
        Key=checkpoint_key,
        Body=json.dumps(checkpoint_data, indent=2),
        ContentType='application/json'
    )
    
    return {
        'statusCode': 200,
        'checkpoint_location': f's3://reconciliation-checkpoints/{checkpoint_key}'
    }
```

4.2. Adicionar estado de checkpoint antes de falhas:
```json
{
  "SaveCheckpoint": {
    "Type": "Task",
    "Resource": "arn:aws:states:::lambda:invoke",
    "Parameters": {
      "FunctionName": "reconciliation-checkpoint",
      "Payload": {
        "job_id": "$.job_id",
        "state": "$$.State.Name",
        "error": "$.error"
      }
    },
    "Next": "NotifyFailure"
  }
}
```

---

### 5. Configurar Métricas e Alertas

**Passos:**

5.1. Adicionar CloudWatch Alarms:
```bash
# Alerta para execuções falhadas
aws cloudwatch put-metric-alarm \
  --alarm-name reconciliation-pipeline-failures \
  --alarm-description "Alerta quando pipeline de conciliação falha" \
  --metric-name ExecutionsFailed \
  --namespace AWS/States \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts

# Alerta para execuções com duração > 18 minutos
aws cloudwatch put-metric-alarm \
  --alarm-name reconciliation-pipeline-slow \
  --alarm-description "Alerta quando pipeline demora mais de 18 minutos" \
  --metric-name ExecutionTime \
  --namespace AWS/States \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1080 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

---

## Validação

### Checklist de Validação:

- [ ] State Machine criada com definição correta
- [ ] IAM Role anexada corretamente
- [ ] CloudWatch Logs configurado
- [ ] X-Ray Tracing habilitado
- [ ] Timeout de 20 minutos configurado
- [ ] Retry policies configuradas
- [ ] Error handling configurado
- [ ] Teste manual: Iniciar execução via Lambda trigger
- [ ] Verificar execução completa no console Step Functions
- [ ] Verificar logs no CloudWatch
- [ ] Verificar métricas publicadas

### Comandos de Validação:

```bash
# Verificar State Machine criada
aws stepfunctions describe-state-machine \
  --state-machine-arn arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline

# Listar execuções recentes
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline \
  --max-results 10

# Ver detalhes de execução específica
aws stepfunctions describe-execution \
  --execution-arn <EXECUTION_ARN>

# Testar execução manual (simular input da Lambda)
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline \
  --name TEST-$(date +%Y%m%d-%H%M%S) \
  --input '{
    "job_id": "TEST-001",
    "bucket_name": "reconciliation-ingress",
    "object_key": "2026-02-16/test.csv",
    "file_hash": "test-hash"
  }'

# Ver logs
aws logs tail /aws/stepfunctions/reconciliation --follow
```

---

## Notas Técnicas

- **Timeout Global:** Configurar timeout de 20 minutos (1200 segundos) para garantir SLA
- **Retry Strategy:** Retries apenas para erros transitórios (Glue service errors)
- **Error Handling:** Todos os estados devem ter catch blocks para tratamento de erros
- **Idempotência:** Job ID único garante que execuções não sejam duplicadas
- **Logging:** Habilitar logging completo para debugging

---

## Próximas Tasks

- **TASK 06:** Implementar Glue Job de Validação (será invocado pelo Step Functions)
- **TASK 07:** Implementar Glue Job de Conciliação (será invocado pelo Step Functions)
- **TASK 09:** Implementar Glue Job de Preparação QuickSight (será invocado pelo Step Functions)
- **TASK 12:** Configurar SNS e Notificações (será usado pelo Step Functions)

---

## Referências

- Documento Técnico - Seção 3.2 (Componentes da Arquitetura)
- Documento Técnico - Seção 2.6 (Estratégia de Retry e Timeout)
- AWS Step Functions Documentation: https://docs.aws.amazon.com/step-functions/
