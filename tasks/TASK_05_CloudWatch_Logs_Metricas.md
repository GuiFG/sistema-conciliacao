# TASK 05: Setup de CloudWatch Logs e Métricas Básicas

**Sprint:** 1-2 (Infraestrutura Base)  
**Prioridade:** ALTA  
**Estimativa:** 1 dia  
**Dependências:** Nenhuma (pode ser feito em paralelo)

---

## Objetivo

Configurar CloudWatch Logs e métricas básicas para monitoramento do sistema de conciliação.

---

## Requisitos

- AWS Account configurada
- Permissões IAM para criar Log Groups e métricas

---

## Passo a Passo Detalhado

### 1. Criar Log Groups

**Passos:**

1.1. Criar Log Group para Lambda:
```bash
aws logs create-log-group \
  --log-group-name /aws/lambda/reconciliation-trigger

aws logs put-retention-policy \
  --log-group-name /aws/lambda/reconciliation-trigger \
  --retention-in-days 90
```

1.2. Criar Log Group para Glue Jobs:
```bash
aws logs create-log-group \
  --log-group-name /aws-glue/reconciliation-validation

aws logs create-log-group \
  --log-group-name /aws-glue/reconciliation-reconciliation

aws logs create-log-group \
  --log-group-name /aws-glue/reconciliation-quicksight-prep

# Configurar retenção
aws logs put-retention-policy \
  --log-group-name /aws-glue/reconciliation-validation \
  --retention-in-days 90

aws logs put-retention-policy \
  --log-group-name /aws-glue/reconciliation-reconciliation \
  --retention-in-days 90

aws logs put-retention-policy \
  --log-group-name /aws-glue/reconciliation-quicksight-prep \
  --retention-in-days 90
```

1.3. Criar Log Group para Step Functions:
```bash
aws logs create-log-group \
  --log-group-name /aws/stepfunctions/reconciliation

aws logs put-retention-policy \
  --log-group-name /aws/stepfunctions/reconciliation \
  --retention-in-days 90
```

---

### 2. Configurar Métricas Customizadas

**Passos:**

2.1. Criar script Python para publicar métricas (`publish_metrics.py`):
```python
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def publish_reconciliation_metric(metric_name, value, unit='Count', dimensions=None):
    """
    Publica métrica customizada no CloudWatch
    """
    try:
        cloudwatch.put_metric_data(
            Namespace='Reconciliation',
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': value,
                    'Unit': unit,
                    'Timestamp': datetime.utcnow(),
                    'Dimensions': dimensions or []
                }
            ]
        )
    except Exception as e:
        print(f'Erro ao publicar métrica: {str(e)}')

# Exemplos de uso:
# publish_reconciliation_metric('TotalLinesProcessed', 1000000, dimensions=[
#     {'Name': 'JobId', 'Value': 'JOB-20260216-001234'}
# ])
# 
# publish_reconciliation_metric('ProcessingTime', 1200, 'Seconds', dimensions=[
#     {'Name': 'Stage', 'Value': 'Validation'}
# ])
```

2.2. Métricas a serem publicadas:

**Métricas de Ingestão:**
- `TotalLines` (Count)
- `ValidLines` (Count)
- `InvalidLines` (Count)
- `IngestionTime` (Seconds)

**Métricas de Conciliação:**
- `DivergencesFound` (Count) - por tipo
- `ReconciliationTime` (Seconds)
- `Throughput` (Count/Second)

**Métricas de Resultado:**
- `TotalDivergences` (Count)
- `DivergencesByType` (Count) - TIPO_1, TIPO_2, TIPO_3
- `ReportGenerationTime` (Seconds)

---

### 3. Configurar Dashboards CloudWatch

**Passos:**

3.1. Criar dashboard básico (`dashboard.json`):
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["Reconciliation", "TotalLinesProcessed"],
          [".", "ValidLines"],
          [".", "InvalidLines"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Linhas Processadas"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["Reconciliation", "ProcessingTime", {"stat": "Average"}],
          [".", ".", {"stat": "Maximum"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Tempo de Processamento"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["Reconciliation", "DivergencesFound", {"dimensions": {"Type": "TIPO_1"}}],
          [".", ".", {"dimensions": {"Type": "TIPO_2"}}],
          [".", ".", {"dimensions": {"Type": "TIPO_3"}}]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Divergências por Tipo"
      }
    }
  ]
}
```

3.2. Criar dashboard:
```bash
aws cloudwatch put-dashboard \
  --dashboard-name Reconciliation-MVP \
  --dashboard-body file://dashboard.json
```

---

### 4. Configurar Alertas Básicos

**Passos:**

4.1. Alerta para processamento > 18 minutos:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name reconciliation-processing-slow \
  --alarm-description "Alerta quando processamento demora mais de 18 minutos" \
  --metric-name ProcessingTime \
  --namespace Reconciliation \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1080 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

4.2. Alerta para taxa de erro > 5%:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name reconciliation-error-rate-high \
  --alarm-description "Alerta quando taxa de erro excede 5%" \
  --metric-name ErrorRate \
  --namespace Reconciliation \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

4.3. Alerta para divergências > 10%:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name reconciliation-divergence-rate-high \
  --alarm-description "Alerta quando taxa de divergências excede 10%" \
  --metric-name DivergenceRate \
  --namespace Reconciliation \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts
```

---

## Validação

### Checklist de Validação:

- [ ] Log Groups criados para todos os componentes
- [ ] Retenção configurada (90 dias)
- [ ] Dashboard CloudWatch criado
- [ ] Alertas configurados
- [ ] Teste: Publicar métrica customizada
- [ ] Verificar métricas no console CloudWatch
- [ ] Verificar logs após execução de teste

### Comandos de Validação:

```bash
# Listar Log Groups
aws logs describe-log-groups --query 'logGroups[?contains(logGroupName, `reconciliation`)]'

# Verificar retenção
aws logs describe-log-groups \
  --log-group-name-prefix /aws/lambda/reconciliation \
  --query 'logGroups[*].[logGroupName,retentionInDays]'

# Listar métricas customizadas
aws cloudwatch list-metrics \
  --namespace Reconciliation

# Verificar dashboard
aws cloudwatch get-dashboard --dashboard-name Reconciliation-MVP

# Listar alarmes
aws cloudwatch describe-alarms \
  --alarm-name-prefix reconciliation
```

---

## Notas Técnicas

- **Retenção:** 90 dias conforme PRD (Seção 6.5)
- **Namespace:** Usar namespace único "Reconciliation" para todas as métricas
- **Dimensões:** Incluir JobId, Stage, Type para facilitar filtragem
- **Custos:** Monitorar custos de CloudWatch Logs e métricas

---

## Próximas Tasks

- **TASK 06:** Implementar Glue Job de Validação (publicará métricas)
- **TASK 07:** Implementar Glue Job de Conciliação (publicará métricas)

---

## Referências

- Documento Técnico - Seção 10.1 (Logs Estruturados)
- Documento Técnico - Seção 10.2 (Métricas Customizadas)
- AWS CloudWatch Documentation: https://docs.aws.amazon.com/cloudwatch/
