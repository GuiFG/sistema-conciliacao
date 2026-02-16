# TASK 11: Configurar QuickSight Dataset e Dashboard

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** ALTA  
**Estimativa:** 3-4 dias  
**Dependências:** TASK 10 (Glue Data Catalog)

---

## Objetivo

Configurar Amazon QuickSight com dataset de divergências e criar dashboard interativo conforme especificação do Documento Técnico.

---

## Requisitos

- QuickSight account configurado
- Tabela catalogada no Glue Data Catalog (TASK 10)
- IAM Role para QuickSight configurada (TASK 02)

---

## Passo a Passo Detalhado

### 1. Configurar Permissões QuickSight

**Passos:**

1.1. Adicionar IAM Role do QuickSight ao Glue Database:
```bash
aws glue put-resource-policy \
  --policy-in-json '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "quicksight.amazonaws.com"
        },
        "Action": [
          "glue:GetDatabase",
          "glue:GetTable",
          "glue:GetPartitions"
        ],
        "Resource": [
          "arn:aws:glue:us-east-1:ACCOUNT_ID:database/reconciliation",
          "arn:aws:glue:us-east-1:ACCOUNT_ID:table/reconciliation/divergences"
        ]
      }
    ]
  }'
```

1.2. Configurar acesso QuickSight ao S3:
   - No console QuickSight, adicionar S3 bucket `reconciliation-output` como fonte de dados
   - Usar IAM Role `ReconciliationQuickSightRole`

---

### 2. Criar Dataset no QuickSight

**Passos:**

2.1. Criar dataset via console QuickSight:
   - **Fonte:** AWS Glue Data Catalog
   - **Database:** `reconciliation`
   - **Table:** `divergences`
   - **Name:** `Reconciliation Divergences`

2.2. Configurar refresh automático:
   - **Schedule:** Após cada execução (via API ou manual)
   - **Incremental:** Sim (apenas novos dados)

---

### 3. Criar Dashboard: Visão Geral de Divergências

**Conforme Documento Técnico - Seção 4.3:**

**3.1. Métricas Principais (KPIs):**

- **Total de Divergências:**
  - Tipo: Número
  - Fórmula: `COUNT(ordem_id)`
  - Formato: Número inteiro

- **Taxa de Divergência:**
  - Tipo: Percentual
  - Fórmula: `(COUNT(ordem_id) / [Total de transações]) * 100`
  - Nota: Total de transações vem de métrica separada ou parâmetro

- **Total por Tipo:**
  - Tipo: Tabela
  - Colunas: `tipo_divergencia`, `COUNT(ordem_id)`
  - Agrupado por: `tipo_divergencia`

**3.2. Gráficos:**

- **Distribuição por Tipo (Pizza):**
  - Tipo: Gráfico de Pizza
  - Dimensão: `tipo_divergencia`
  - Medida: `COUNT(ordem_id)`

- **Distribuição por Severidade (Barras):**
  - Tipo: Gráfico de Barras
  - Dimensão: `severidade`
  - Medida: `COUNT(ordem_id)`
  - Ordenação: HIGH, MEDIUM, LOW

- **Top 10 Fornecedores (Barras Horizontais):**
  - Tipo: Gráfico de Barras Horizontais
  - Dimensão: `fornecedor_id`
  - Medida: `COUNT(ordem_id)`
  - Limite: Top 10

- **Tendência Histórica (Linha):**
  - Tipo: Gráfico de Linha
  - Dimensão: `data`
  - Medida: `COUNT(ordem_id)`
  - Período: Últimos 30 dias

**3.3. Tabela de Divergências:**

- **Colunas:**
  - `fornecedor_id`
  - `ordem_id`
  - `tipo_divergencia`
  - `detalhes`
  - `severidade`
  - `data`
  - `valor_csv`
  - `valor_base`
  - `diferenca_valor`

- **Filtros:**
  - Tipo de divergência (dropdown)
  - Severidade (checkbox)
  - Fornecedor (dropdown)
  - Data (range picker)

- **Paginação:** 100 registros por página

- **Export:** Botão para exportar para CSV

**3.4. Filtros Globais:**

- **Período:** Date range picker (data inicial e final)
- **Fornecedor:** Dropdown com lista de fornecedores
- **Tipo de Divergência:** Checkbox múltipla seleção
- **Severidade:** Checkbox múltipla seleção

---

### 4. Configurar Refresh Automático

**Passos:**

4.1. Criar Lambda para trigger de refresh:
```python
import boto3

quicksight = boto3.client('quicksight')

def lambda_handler(event, context):
    """
    Trigger refresh do dataset QuickSight após processamento
    """
    dataset_id = 'YOUR_DATASET_ID'
    aws_account_id = 'ACCOUNT_ID'
    
    try:
        response = quicksight.create_ingestion(
            DataSetId=dataset_id,
            IngestionId=f'ingestion-{datetime.utcnow().strftime("%Y%m%d-%H%M%S")}',
            AwsAccountId=aws_account_id,
            IngestionType='INCREMENTAL_REFRESH'
        )
        return {
            'statusCode': 200,
            'ingestion_id': response['IngestionId']
        }
    except Exception as e:
        print(f'Erro ao criar ingestion: {str(e)}')
        return {
            'statusCode': 500,
            'error': str(e)
        }
```

4.2. Adicionar ao Step Functions (após preparação QuickSight):
```json
{
  "RefreshQuickSight": {
    "Type": "Task",
    "Resource": "arn:aws:states:::lambda:invoke",
    "Parameters": {
      "FunctionName": "reconciliation-quicksight-refresh",
      "Payload": {
        "job_id": "$.job_id"
      }
    },
    "Next": "NotifySuccess"
  }
}
```

---

### 5. Configurar Row-Level Security (Opcional)

**Se necessário restringir acesso por fornecedor:**

```python
# Criar regra RLS
quicksight.create_data_source(
    AwsAccountId='ACCOUNT_ID',
    DataSourceId='reconciliation-rls',
    Name='Reconciliation RLS',
    Type='S3',
    DataSourceParameters={
        'S3Parameters': {
            'ManifestFileLocation': {
                'Bucket': 'reconciliation-output',
                'Key': 'rls-manifest.json'
            }
        }
    }
)
```

---

## Validação

### Checklist de Validação:

- [ ] QuickSight account configurado
- [ ] Permissões IAM configuradas
- [ ] Dataset criado e conectado ao Glue Data Catalog
- [ ] Dashboard criado com todas as visualizações
- [ ] Filtros funcionando corretamente
- [ ] Refresh automático configurado
- [ ] Teste: Visualizar dados após processamento
- [ ] Teste: Exportar dados para CSV
- [ ] Validar performance (dashboard carrega em tempo razoável)

### Testes Manuais:

1. Acessar dashboard no QuickSight
2. Verificar que métricas estão corretas
3. Testar filtros (tipo, severidade, fornecedor, data)
4. Verificar gráficos renderizam corretamente
5. Testar export para CSV
6. Verificar refresh automático após novo processamento

---

## Notas Técnicas

- **Performance:** Otimizar queries usando partições (`fornecedor_id`, `data`)
- **Refresh:** Configurar refresh incremental para melhor performance
- **Custos:** QuickSight cobra por usuário e por SPICE capacity (se usado)
- **Limites:** QuickSight tem limites de dados (verificar documentação)

---

## Próximas Tasks

- **TASK 12:** Configurar SNS e Notificações (incluir link para dashboard)

---

## Referências

- Documento Técnico - Seção 4.3 (Dashboard: Visão Geral de Divergências)
- Amazon QuickSight Documentation: https://docs.aws.amazon.com/quicksight/
