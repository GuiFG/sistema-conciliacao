# TASK 01: Configuração de Infraestrutura S3

**Sprint:** 1-2 (Infraestrutura Base)  
**Prioridade:** CRÍTICA  
**Estimativa:** 2-3 dias  
**Dependências:** Nenhuma

---

## Objetivo

Configurar os buckets S3 necessários para o sistema de conciliação financeira, incluindo buckets de ingress (entrada), output (saída) e intermediário (dados processados).

---

## Requisitos

- AWS Account configurada
- Permissões IAM para criar buckets S3
- Acesso ao AWS Console ou AWS CLI

---

## Passo a Passo Detalhado

### 1. Criar Bucket de Ingress (Entrada)

**Objetivo:** Armazenar arquivos CSV recebidos do fornecedor

**Passos:**

1.1. Criar bucket S3:
```bash
aws s3 mb s3://reconciliation-ingress --region us-east-1
```

1.2. Configurar versionamento:
```bash
aws s3api put-bucket-versioning \
  --bucket reconciliation-ingress \
  --versioning-configuration Status=Enabled
```

1.3. Configurar lifecycle policy (opcional - para limpeza automática):
```json
{
  "Rules": [
    {
      "Id": "DeleteOldCSV",
      "Status": "Enabled",
      "Expiration": {
        "Days": 90
      },
      "Prefix": ""
    }
  ]
}
```

1.4. Aplicar lifecycle policy:
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket reconciliation-ingress \
  --lifecycle-configuration file://lifecycle-ingress.json
```

1.5. Habilitar criptografia SSE-KMS:
```bash
aws s3api put-bucket-encryption \
  --bucket reconciliation-ingress \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/reconciliation-key"
      }
    }]
  }'
```

1.6. Configurar EventBridge para detectar uploads:
   - Criar regra EventBridge que detecta `s3:ObjectCreated:*` no bucket
   - Configurar target como Lambda (será criada na TASK 03)

---

### 2. Criar Bucket de Output (Saída)

**Objetivo:** Armazenar resultados finais (Parquet com divergências) para QuickSight

**Passos:**

2.1. Criar bucket S3:
```bash
aws s3 mb s3://reconciliation-output --region us-east-1
```

2.2. Configurar versionamento:
```bash
aws s3api put-bucket-versioning \
  --bucket reconciliation-output \
  --versioning-configuration Status=Enabled
```

2.3. Configurar lifecycle policy (retenção de 2 anos conforme PRD):
```json
{
  "Rules": [
    {
      "Id": "ArchiveToGlacier",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 730
      },
      "Prefix": ""
    }
  ]
}
```

2.4. Habilitar criptografia SSE-KMS:
```bash
aws s3api put-bucket-encryption \
  --bucket reconciliation-output \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/reconciliation-key"
      }
    }]
  }'
```

2.5. Configurar CORS (se necessário para acesso externo):
```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET"],
      "AllowedHeaders": ["*"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

---

### 3. Criar Bucket Intermediário

**Objetivo:** Armazenar dados intermediários durante processamento (Parquet validado)

**Passos:**

3.1. Criar bucket S3:
```bash
aws s3 mb s3://reconciliation-intermediate --region us-east-1
```

3.2. Configurar versionamento:
```bash
aws s3api put-bucket-versioning \
  --bucket reconciliation-intermediate \
  --versioning-configuration Status=Enabled
```

3.3. Configurar lifecycle policy (limpeza após 7 dias):
```json
{
  "Rules": [
    {
      "Id": "DeleteIntermediateData",
      "Status": "Enabled",
      "Expiration": {
        "Days": 7
      },
      "Prefix": ""
    }
  ]
}
```

3.4. Habilitar criptografia SSE-KMS:
```bash
aws s3api put-bucket-encryption \
  --bucket reconciliation-intermediate \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/reconciliation-key"
      }
    }]
  }'
```

---

### 4. Criar Bucket de Checkpoints (Opcional para MVP, mas recomendado)

**Objetivo:** Armazenar checkpoints em caso de falha/timeout

**Passos:**

4.1. Criar bucket S3:
```bash
aws s3 mb s3://reconciliation-checkpoints --region us-east-1
```

4.2. Configurar versionamento:
```bash
aws s3api put-bucket-versioning \
  --bucket reconciliation-checkpoints \
  --versioning-configuration Status=Enabled
```

4.3. Configurar lifecycle policy (retenção de 30 dias):
```json
{
  "Rules": [
    {
      "Id": "DeleteOldCheckpoints",
      "Status": "Enabled",
      "Expiration": {
        "Days": 30
      },
      "Prefix": ""
    }
  ]
}
```

---

### 5. Estrutura de Pastas Recomendada

**Bucket de Ingress:**
```
s3://reconciliation-ingress/
├── YYYY-MM-DD/
│   └── input.csv
```

**Bucket Intermediário:**
```
s3://reconciliation-intermediate/
├── YYYY-MM-DD/
│   ├── validated.parquet/
│   └── partial_reconciliation.parquet/
```

**Bucket de Output:**
```
s3://reconciliation-output/
├── YYYY-MM-DD/
│   └── divergencias.parquet/
```

**Bucket de Checkpoints:**
```
s3://reconciliation-checkpoints/
├── JOB-YYYYMMDD-HHMMSS/
│   └── checkpoint.json
```

---

### 6. Configurar EventBridge Rule para Ingress

**Passos:**

6.1. Criar regra EventBridge:
```bash
aws events put-rule \
  --name reconciliation-csv-uploaded \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {
        "name": ["reconciliation-ingress"]
      },
      "object": {
        "key": [{
          "suffix": ".csv"
        }]
      }
    }
  }'
```

6.2. Adicionar permissão para Lambda invocar (será configurado na TASK 03):
```bash
aws lambda add-permission \
  --function-name reconciliation-trigger \
  --statement-id allow-eventbridge \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:ACCOUNT_ID:rule/reconciliation-csv-uploaded
```

---

## Validação

### Checklist de Validação:

- [ ] Bucket `reconciliation-ingress` criado e configurado
- [ ] Bucket `reconciliation-output` criado e configurado
- [ ] Bucket `reconciliation-intermediate` criado e configurado
- [ ] Bucket `reconciliation-checkpoints` criado e configurado
- [ ] Versionamento habilitado em todos os buckets
- [ ] Criptografia SSE-KMS configurada em todos os buckets
- [ ] Lifecycle policies aplicadas
- [ ] EventBridge rule criada e configurada
- [ ] Estrutura de pastas documentada
- [ ] Teste de upload manual de CSV no bucket ingress

### Comandos de Validação:

```bash
# Listar buckets criados
aws s3 ls | grep reconciliation

# Verificar versionamento
aws s3api get-bucket-versioning --bucket reconciliation-ingress

# Verificar criptografia
aws s3api get-bucket-encryption --bucket reconciliation-ingress

# Verificar lifecycle
aws s3api get-bucket-lifecycle-configuration --bucket reconciliation-ingress

# Testar upload
echo "test" > test.csv
aws s3 cp test.csv s3://reconciliation-ingress/2026-02-16/test.csv
```

---

## Notas Técnicas

- **Região:** Usar mesma região para todos os buckets (recomendado: `us-east-1`)
- **KMS Key:** Criar chave KMS antes de configurar criptografia ou usar alias padrão
- **Custos:** Monitorar custos de storage e requests S3
- **Performance:** Considerar S3 Transfer Acceleration para uploads grandes (opcional)

---

## Próximas Tasks

- **TASK 02:** Configurar IAM Roles e Políticas (requer buckets criados)
- **TASK 03:** Implementar Lambda Trigger (requer bucket ingress e EventBridge)

---

## Referências

- Documento Técnico - Seção 3.2 (Componentes da Arquitetura)
- PRD - Seção 5.2.1 (Formatos de Entrega)
- AWS S3 Documentation: https://docs.aws.amazon.com/s3/
