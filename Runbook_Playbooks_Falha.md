# Runbook e Playbooks de Falha - Sistema de Conciliação Financeira

**Versão:** 1.0  
**Data:** 16 de Fevereiro de 2026  
**Sistema:** Sistema de Conciliação Financeira  
**Arquitetura:** AWS Glue + RDS + QuickSight

---

## 1. Runbook Geral

### 1.1 Monitoramento Diário

**Checklist:**
- [ ] Verificar execução do dia anterior (CloudWatch Dashboard)
- [ ] Validar que tempo de processamento ≤ 20 minutos
- [ ] Verificar taxa de divergências (comparar com histórico)
- [ ] Validar que notificações foram enviadas
- [ ] Verificar espaço em S3
- [ ] Verificar atualização do dashboard QuickSight

**Frequência:** Diária, após execução D-1

**Ferramentas:**
- CloudWatch Dashboard: `Reconciliation-Dashboard`
- QuickSight: Dashboard "Visão Geral de Divergências"
- S3 Console: Bucket `reconciliation-output`

---

## 2. Playbooks de Falha

### 2.1 Playbook: Falha na Ingestão

**Severidade:** CRITICAL  
**Tempo de Resposta:** ≤ 15 minutos

**Sintomas:**
- Job falha nos primeiros 3 minutos
- Erro de validação de esquema ou formato
- Lambda ou Glue Job retorna erro
- Notificação de falha recebida

**Ações:**

#### 1. Identificar Erro

```bash
# Verificar logs do Lambda trigger
aws logs tail /aws/lambda/reconciliation-trigger --follow

# Verificar logs do Glue Job de validação
aws logs tail /aws-glue/jobs/output --follow

# Verificar execução do Step Functions
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:REGION:ACCOUNT:execution:reconciliation-state-machine:EXECUTION_ID
```

#### 2. Validar CSV

- Verificar formato do CSV recebido no S3:
  ```bash
  aws s3 cp s3://reconciliation-ingress/YYYY-MM-DD/input.csv - | head -20
  ```
- Validar cabeçalho e separadores (deve ser `;`)
- Verificar encoding (deve ser UTF-8)
- Verificar se arquivo não está vazio ou corrompido

#### 3. Ações Corretivas

**Erro de formato:**
- Contatar fornecedor para correção do CSV
- Validar formato esperado: `fornecedor_id;ordem_id;nome;ativo_id;data;valor`
- Aguardar correção e reprocessar

**Erro interno:**
- Verificar permissões S3:
  ```bash
  aws iam get-role-policy --role-name ReconciliationGlueRole --policy-name s3-read
  ```
- Verificar IAM roles do Lambda e Glue
- Verificar se bucket existe e está acessível

**Timeout:**
- Verificar se CSV é muito grande (> 1.5M linhas)
- Considerar aumentar timeout do Lambda/Glue (se necessário)

#### 4. Reprocessamento

```bash
# Reprocessar após correção
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --input '{"s3_key": "YYYY-MM-DD/corrected.csv", "force_reprocess": true}'
```

**Notificação:** Slack #conciliação-alerts + Email (CRITICAL)

---

### 2.2 Playbook: Timeout de 20 Minutos

**Severidade:** CRITICAL  
**Tempo de Resposta:** Imediato (automático) + análise manual conforme necessário

**Sintomas:**
- Timer de 20 minutos expira durante processamento
- Step Functions aborta jobs Glue automaticamente
- Notificação de timeout enviada
- Checkpoint gerado automaticamente

**Ações Automáticas (Sistema):**

1. **Abortar Processamento:**
   - Step Functions cancela todos os Glue Jobs em execução
   - Parar qualquer job que ainda esteja rodando
   - Status final: `TIMEOUT`

2. **Gerar Checkpoint:**
   - Salvar estado atual do processamento em S3
   - Localização: `s3://reconciliation-checkpoints/{job_id}/checkpoint.json`
   - Incluir: linhas processadas, linhas pendentes, localização dados intermediários

3. **Enviar Notificação:**
   - Notificação CRITICAL via Slack + Email
   - Incluir: Job ID, timestamp, motivo (timeout), localização checkpoint

**Ações Manuais (Operador):**

#### 1. Verificar Checkpoint

```bash
# Localizar checkpoint
JOB_ID="JOB-20260216-001234"
aws s3 ls s3://reconciliation-checkpoints/${JOB_ID}/

# Ler conteúdo do checkpoint
aws s3 cp s3://reconciliation-checkpoints/${JOB_ID}/checkpoint.json - | jq .
```

**Estrutura esperada do checkpoint:**
```json
{
  "job_id": "JOB-20260216-001234",
  "start_time": "2026-02-16T07:00:00Z",
  "end_time": "2026-02-16T07:20:00Z",
  "status": "TIMEOUT",
  "progress": {
    "total_lines": 1000000,
    "processed_lines": 750000,
    "pending_lines": 250000,
    "last_processed_partition": "fornecedor_12345"
  },
  "checkpoint_location": "s3://reconciliation-checkpoints/JOB-20260216-001234/",
  "intermediate_data": {
    "validated_data": "s3://reconciliation-intermediate/2026-02-16/validated.parquet",
    "partial_reconciliation": "s3://reconciliation-intermediate/2026-02-16/partial.parquet"
  },
  "error_details": {
    "error_type": "TIMEOUT",
    "message": "Processamento excedeu 20 minutos",
    "stage": "reconciliation"
  }
}
```

#### 2. Analisar Causa do Timeout

**Verificar métricas CloudWatch:**
```bash
# Tempo por etapa
aws cloudwatch get-metric-statistics \
  --namespace Reconciliation \
  --metric-name ProcessingTime \
  --dimensions Name=job_id,Value=${JOB_ID} \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:20:00Z \
  --period 60 \
  --statistics Average,Maximum

# Latência do RDS
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadLatency \
  --dimensions Name=DBInstanceIdentifier,Value=reconciliation-db \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:20:00Z \
  --period 60 \
  --statistics Average,Maximum

# Throughput do Glue
aws cloudwatch get-metric-statistics \
  --namespace AWS/Glue \
  --metric-name glue.driver.ExecutorAllocationManager.executor.numberAllocatedExecutors \
  --dimensions Name=JobName,Value=reconciliation-job \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:20:00Z \
  --period 60 \
  --statistics Average,Maximum
```

**Identificar gargalo:**
- Se latência RDS alta → Problema de conectividade ou índice
- Se throughput Glue baixo → Problema de particionamento ou recursos
- Se tempo de ingestão alto → Problema de I/O S3

#### 3. Decidir Ação

**Se RDS lento:**
- Verificar índice em `ordem_id`:
  ```sql
  SELECT indexname, indexdef 
  FROM pg_indexes 
  WHERE tablename = 'transacoes' AND indexdef LIKE '%ordem_id%';
  ```
- Considerar read replica para leitura
- Verificar conexões simultâneas (connection pooling)

**Se particionamento ruim:**
- Revisar estratégia de particionamento no Glue Job
- Verificar se há partições muito grandes (skew)
- Ajustar número de partições

**Se volume maior que esperado:**
- Validar se CSV tem mais de 1M linhas
- Verificar se há linhas duplicadas
- Considerar aumentar DPU do Glue temporariamente

#### 4. Reprocessamento

**Opção 1: Reprocessar usando checkpoint (recomendado)**
```bash
# Script de reprocessamento parcial
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --input '{
    "job_id": "'${JOB_ID}'",
    "checkpoint_location": "s3://reconciliation-checkpoints/'${JOB_ID}'/checkpoint.json",
    "reprocess_from_checkpoint": true
  }'
```

**Opção 2: Reprocessar completo**
```bash
# Após correção do problema, reprocessar do início
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --input '{"s3_key": "YYYY-MM-DD/input.csv", "force_reprocess": true}'
```

**⚠️ IMPORTANTE:** Checkpoint permite reprocessamento parcial sem perder progresso.

---

### 2.3 Playbook: Falha na Conciliação

**Severidade:** HIGH  
**Tempo de Resposta:** ≤ 30 minutos

**Sintomas:**
- Job falha durante etapa de conciliação
- Erro de conexão com RDS
- Erro de memória (OOM)
- Timeout de query no RDS
- Glue Job retorna erro

**Ações:**

#### 1. Verificar Conectividade com RDS

```bash
# Testar conexão JDBC (via Lambda de teste)
aws lambda invoke \
  --function-name test-rds-connection \
  --payload '{"host": "reconciliation-db.xxx.rds.amazonaws.com", "port": 5432}' \
  response.json

# Verificar Security Groups
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=glue-sg" \
  --query 'SecurityGroups[0].IpPermissions'

# Validar credenciais no Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id reconciliation-rds-credentials \
  --query SecretString --output text | jq .

# Verificar VPC endpoints (se aplicável)
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.REGION.rds"
```

#### 2. Verificar Índice no RDS

**PostgreSQL:**
```sql
-- Verificar se índice único em ordem_id existe
SELECT 
    indexname, 
    indexdef 
FROM pg_indexes 
WHERE tablename = 'transacoes' 
  AND indexdef LIKE '%ordem_id%';

-- Se não existir, criar:
CREATE UNIQUE INDEX idx_ordem_id ON transacoes(ordem_id);

-- Verificar estatísticas do índice
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname = 'idx_ordem_id';
```

**MySQL:**
```sql
-- Verificar índices
SHOW INDEX FROM transacoes WHERE Column_name = 'ordem_id';

-- Criar índice se não existir
CREATE UNIQUE INDEX idx_ordem_id ON transacoes(ordem_id);

-- Verificar uso do índice
EXPLAIN SELECT * FROM transacoes WHERE ordem_id = 'ORD-12345';
```

#### 3. Verificar Performance do RDS

**Métricas CloudWatch:**
```bash
# CPU Utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=reconciliation-db \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:30:00Z \
  --period 60 \
  --statistics Average,Maximum

# Database Connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=reconciliation-db \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:30:00Z \
  --period 60 \
  --statistics Average,Maximum

# Read Latency
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadLatency \
  --dimensions Name=DBInstanceIdentifier,Value=reconciliation-db \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T07:30:00Z \
  --period 60 \
  --statistics Average,Maximum
```

**Verificar queries lentas:**
- PostgreSQL: `pg_stat_statements`
- MySQL: `slow_query_log`

#### 4. Ações Corretivas

**Falha de conexão:**
- Verificar Security Groups (Glue → RDS):
  ```bash
  # Adicionar regra se necessário
  aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --protocol tcp \
    --port 5432 \
    --source-group sg-glue-sg
  ```
- Validar credenciais no Secrets Manager
- Verificar se RDS está em mesma VPC ou usar VPC endpoints
- Verificar se RDS está acessível (não em manutenção)

**OOM (Out of Memory):**
- Aumentar memória do Glue Job (mais DPU):
  ```bash
  aws glue update-job \
    --job-name reconciliation-job \
    --job-update '{"NumberOfWorkers": 15, "WorkerType": "G.1X"}'
  ```
- Particionar leitura do RDS (batch reads menores)
- Reduzir número de conexões simultâneas

**Timeout de query:**
- Otimizar query (garantir uso de índice em `ordem_id`)
- Aumentar timeout do JDBC connection no Glue Job
- Considerar read replica para leitura
- Verificar se há locks bloqueando queries

**RDS sobrecarregado:**
- Escalar instância RDS temporariamente:
  ```bash
  aws rds modify-db-instance \
    --db-instance-identifier reconciliation-db \
    --db-instance-class db.r5.2xlarge \
    --apply-immediately
  ```
- Implementar export diário para S3 (fase produção)
- Reduzir carga em outras aplicações usando o mesmo RDS

#### 5. Reprocessamento

**Usar checkpoint (se disponível):**
```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --input '{
    "job_id": "'${JOB_ID}'",
    "checkpoint_location": "s3://reconciliation-checkpoints/'${JOB_ID}'/checkpoint.json",
    "reprocess_from_checkpoint": true
  }'
```

**Ou reprocessar do início após correção:**
```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --input '{"s3_key": "YYYY-MM-DD/input.csv", "force_reprocess": true}'
```

**⚠️ VALIDAÇÃO CRÍTICA:** Confirmar que índice único em `ordem_id` existe no RDS antes de produção.

---

### 2.4 Playbook: Falha na Entrega de Resultados

**Severidade:** MEDIUM  
**Tempo de Resposta:** ≤ 10 minutos

**Sintomas:**
- Processamento completo mas arquivo não disponível em S3
- Erro ao escrever arquivo de saída
- Notificação não enviada
- QuickSight não atualizado

**Ações:**

#### 1. Verificar Permissões

```bash
# Validar IAM role do Glue tem permissão de escrita no S3
aws iam get-role-policy \
  --role-name ReconciliationGlueRole \
  --policy-name s3-write

# Verificar política S3
aws iam get-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT:policy/s3-write-policy \
  --version-id v1
```

**Política esperada:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": [
        "arn:aws:s3:::reconciliation-output/*"
      ]
    }
  ]
}
```

#### 2. Verificar Espaço e Limites

```bash
# Verificar espaço disponível no S3 bucket
aws s3api list-objects-v2 \
  --bucket reconciliation-output \
  --prefix 2026-02-16/ \
  --query 'Contents[].Size' \
  --output text | awk '{sum+=$1} END {print sum/1024/1024/1024 " GB"}'

# Verificar limites de tamanho de objeto (máx 5TB)
# Verificar se há lifecycle policies deletando objetos
aws s3api get-bucket-lifecycle-configuration \
  --bucket reconciliation-output
```

#### 3. Verificar Logs do Glue Job

```bash
# Verificar logs do Glue Job de saída
aws logs tail /aws-glue/jobs/output/reconciliation-output-job \
  --since 30m \
  --filter-pattern "ERROR"
```

#### 4. Regenerar Resultados

**Se dados intermediários existem:**
```bash
# Localizar dados intermediários
aws s3 ls s3://reconciliation-intermediate/2026-02-16/

# Executar apenas Glue Job de saída
aws glue start-job-run \
  --job-name reconciliation-output-job \
  --arguments '{
    "--input_path": "s3://reconciliation-intermediate/2026-02-16/divergences.parquet",
    "--output_path": "s3://reconciliation-output/2026-02-16/divergencias.parquet"
  }'
```

**Se dados intermediários não existem:**
- Reprocessar completo (última opção)
- Verificar se checkpoint está disponível

#### 5. Verificar QuickSight

```bash
# Verificar se dataset foi atualizado
aws quicksight describe-data-set \
  --aws-account-id ACCOUNT \
  --data-set-id reconciliation-dataset

# Forçar refresh do dataset
aws quicksight create-ingestion \
  --data-set-id reconciliation-dataset \
  --ingestion-id $(date +%s) \
  --ingestion-type INCREMENTAL_REFRESH
```

---

### 2.5 Playbook: Taxa de Divergências Anormal

**Severidade:** MEDIUM  
**Tempo de Resposta:** ≤ 1 hora

**Sintomas:**
- Taxa de divergências > 10% (vs. histórico de ~5%)
- Aumento súbito em Tipo 1 ou Tipo 2
- Alertas do CloudWatch disparados
- Notificação recebida

**Ações:**

#### 1. Analisar Padrão

**Verificar no QuickSight:**
- Dashboard "Visão Geral de Divergências"
- Filtrar por data do dia
- Verificar distribuição por tipo de divergência
- Identificar fornecedores com mais divergências

**Verificar métricas CloudWatch:**
```bash
# Taxa de divergências
aws cloudwatch get-metric-statistics \
  --namespace Reconciliation \
  --metric-name DivergenceRate \
  --dimensions Name=job_id,Value=${JOB_ID} \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T08:00:00Z \
  --period 300 \
  --statistics Average,Maximum

# Divergências por tipo
aws cloudwatch get-metric-statistics \
  --namespace Reconciliation \
  --metric-name Divergences \
  --dimensions Name=job_id,Value=${JOB_ID} Name=tipo_divergencia,Value=TIPO_1 \
  --start-time 2026-02-16T07:00:00Z \
  --end-time 2026-02-16T08:00:00Z \
  --period 300 \
  --statistics Sum
```

**Comparar com histórico:**
- Comparar com média dos últimos 7 dias
- Verificar se há padrão sazonal
- Identificar se é problema pontual ou sistemático

#### 2. Investigar Causa

**Validar base interna:**
- Verificar se base interna está atualizada
- Verificar se há atraso na atualização da base
- Validar se há problemas conhecidos na base

**Verificar CSV do fornecedor:**
- Comparar volume de linhas com dias anteriores
- Verificar se há linhas duplicadas
- Validar formato e qualidade dos dados

**Validar regras de conciliação:**
- Verificar se regras não foram alteradas
- Validar chave de correspondência (`ordem_id`)
- Verificar se tolerância está correta (diferença = 0)

#### 3. Notificar Stakeholders

**Área de negócio:**
- Enviar relatório detalhado de divergências
- Incluir análise de padrões identificados
- Sugerir ações corretivas

**Fornecedor (se problema no CSV):**
- Notificar sobre problemas identificados no CSV
- Solicitar correção se necessário
- Agendar reenvio se aplicável

**Template de notificação:**
```
Assunto: Taxa de Divergências Anormal - ${DATA}

Taxa de divergências detectada: ${TAXA}%
Histórico médio: ${HISTORICO}%

Divergências por tipo:
- Tipo 1 (ausente na base): ${TIPO1}
- Tipo 2 (ausente no CSV): ${TIPO2}
- Tipo 3 (campos divergentes): ${TIPO3}

Top 3 fornecedores com mais divergências:
1. ${FORNECEDOR1}: ${QTD1}
2. ${FORNECEDOR2}: ${QTD2}
3. ${FORNECEDOR3}: ${QTD3}

Link para análise detalhada: [QuickSight Dashboard]
```

---

## 3. Contatos e Escalonamento

### 3.1 Contatos de Emergência

| Papel | Contato | Canal | Disponibilidade |
|-------|---------|-------|-----------------|
| **Equipe de Operações/SRE** | ops-conciliacao@empresa.com | Slack #ops-oncall | 24/7 |
| **Product Owner** | po-conciliacao@empresa.com | Email | Horário comercial |
| **Engenharia** | eng-conciliacao@empresa.com | Slack #eng-conciliacao | Horário comercial |
| **Área de Negócio** | negocio-conciliacao@empresa.com | Email | Horário comercial |

### 3.2 SLA de Resposta

| Severidade | Tempo de Resposta | Responsável | Escalation |
|------------|-------------------|-------------|------------|
| **CRITICAL** | ≤ 15 minutos | Equipe de Operações | Product Owner + Engenharia |
| **HIGH** | ≤ 1 hora | Equipe de Operações | Engenharia |
| **MEDIUM** | ≤ 4 horas | Equipe de Operações | - |
| **LOW** | ≤ 1 dia útil | Equipe de Operações | - |

### 3.3 Processo de Escalonamento

1. **Nível 1 (Operações):** Tentar resolver usando playbooks
2. **Nível 2 (Engenharia):** Se problema técnico complexo
3. **Nível 3 (Product Owner):** Se impacto de negócio crítico
4. **Nível 4 (Diretoria):** Se impacto financeiro significativo

---

## 4. Ferramentas e Comandos Úteis

### 4.1 CloudWatch

```bash
# Ver logs em tempo real
aws logs tail /aws-glue/jobs/output --follow

# Ver métricas customizadas
aws cloudwatch get-metric-statistics \
  --namespace Reconciliation \
  --metric-name ProcessingTime \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

### 4.2 Step Functions

```bash
# Listar execuções recentes
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:REGION:ACCOUNT:stateMachine:reconciliation-state-machine \
  --max-results 10

# Ver detalhes de execução
aws stepfunctions describe-execution \
  --execution-arn EXECUTION_ARN
```

### 4.3 Glue

```bash
# Listar jobs
aws glue get-jobs --max-results 10

# Ver detalhes do job
aws glue get-job --job-name reconciliation-job

# Ver execuções do job
aws glue get-job-runs --job-name reconciliation-job --max-results 10
```

### 4.4 S3

```bash
# Listar arquivos no bucket
aws s3 ls s3://reconciliation-output/2026-02-16/

# Baixar arquivo para análise
aws s3 cp s3://reconciliation-output/2026-02-16/divergencias.parquet ./local.parquet

# Verificar tamanho do arquivo
aws s3 ls s3://reconciliation-output/2026-02-16/divergencias.parquet --human-readable
```

---

**Documento gerado em:** 16 de Fevereiro de 2026  
**Versão:** 1.0  
**Última atualização:** 16 de Fevereiro de 2026
