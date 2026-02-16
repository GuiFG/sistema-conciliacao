# TASK 06: Implementar Glue Job de Validação e Parsing

**Sprint:** 3-4 (Processamento Core)  
**Prioridade:** CRÍTICA  
**Estimativa:** 3-4 dias  
**Dependências:** TASK 01 (S3), TASK 02 (IAM), TASK 05 (CloudWatch)

---

## Objetivo

Implementar AWS Glue Job que valida e processa o CSV de entrada, identificando linhas válidas e inválidas, e gerando Parquet intermediário para próxima etapa.

---

## Requisitos

- Buckets S3 criados
- IAM Role `ReconciliationGlueRole` configurada
- CloudWatch Logs configurado
- Conhecimento de PySpark/Spark

---

## Passo a Passo Detalhado

### 1. Criar Script do Glue Job

**Arquivo:** `glue_validation_job.py`

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.types import *
import boto3
import hashlib
from datetime import datetime

# Inicializar contexto
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    'job_id',
    'bucket_name',
    'object_key',
    'file_hash'
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Parâmetros
job_id = args['job_id']
bucket_name = args['bucket_name']
object_key = args['object_key']
file_hash = args['file_hash']

# CloudWatch para métricas
cloudwatch = boto3.client('cloudwatch')

# Definir schema esperado
schema = StructType([
    StructField("fornecedor_id", StringType(), False),
    StructField("ordem_id", StringType(), False),
    StructField("nome", StringType(), False),
    StructField("ativo_id", StringType(), False),
    StructField("data", DateType(), False),
    StructField("valor", DecimalType(10, 2), False)
])

def validate_fornecedor_id(value):
    """Valida fornecedor_id: String não vazia, máximo 50 caracteres"""
    if not value or len(value) > 50:
        return False
    return True

def validate_ordem_id(value):
    """Valida ordem_id: String não vazia, máximo 100 caracteres"""
    if not value or len(value) > 100:
        return False
    return True

def validate_nome(value):
    """Valida nome: String não vazia, máximo 200 caracteres"""
    if not value or len(value) > 200:
        return False
    return True

def validate_ativo_id(value):
    """Valida ativo_id: String não vazia, máximo 50 caracteres"""
    if not value or len(value) > 50:
        return False
    return True

def validate_data(value):
    """Valida data: Formato YYYY-MM-DD, data válida, não futura além de D+1"""
    try:
        from datetime import datetime, timedelta
        date_obj = datetime.strptime(value, '%Y-%m-%d')
        max_date = datetime.now() + timedelta(days=1)
        if date_obj > max_date:
            return False
        return True
    except:
        return False

def validate_valor(value):
    """Valida valor: Decimal positivo, máximo 2 casas decimais, valor > 0"""
    try:
        decimal_value = float(value)
        if decimal_value <= 0:
            return False
        # Verificar casas decimais
        if len(str(decimal_value).split('.')) > 1:
            decimals = len(str(decimal_value).split('.')[1])
            if decimals > 2:
                return False
        return True
    except:
        return False

def calculate_row_hash(row):
    """Calcula hash SHA-256 da linha para auditoria"""
    row_str = f"{row.fornecedor_id}|{row.ordem_id}|{row.nome}|{row.ativo_id}|{row.data}|{row.valor}"
    return hashlib.sha256(row_str.encode()).hexdigest()

# Ler CSV do S3
s3_path = f"s3://{bucket_name}/{object_key}"

print(f"Lendo CSV de: {s3_path}")

# Ler CSV com schema
df = spark.read \
    .option("header", "true") \
    .option("delimiter", ";") \
    .option("dateFormat", "yyyy-MM-dd") \
    .schema(schema) \
    .csv(s3_path)

# Adicionar colunas de validação
df = df.withColumn("is_valid_fornecedor_id", F.udf(validate_fornecedor_id, BooleanType())(F.col("fornecedor_id"))) \
       .withColumn("is_valid_ordem_id", F.udf(validate_ordem_id, BooleanType())(F.col("ordem_id"))) \
       .withColumn("is_valid_nome", F.udf(validate_nome, BooleanType())(F.col("nome"))) \
       .withColumn("is_valid_ativo_id", F.udf(validate_ativo_id, BooleanType())(F.col("ativo_id"))) \
       .withColumn("is_valid_data", F.udf(validate_data, BooleanType())(F.col("data"))) \
       .withColumn("is_valid_valor", F.udf(validate_valor, BooleanType())(F.col("valor")))

# Determinar se linha é válida
df = df.withColumn("is_valid", 
    F.col("is_valid_fornecedor_id") & 
    F.col("is_valid_ordem_id") & 
    F.col("is_valid_nome") & 
    F.col("is_valid_ativo_id") & 
    F.col("is_valid_data") & 
    F.col("is_valid_valor"))

# Adicionar hash de linha
df = df.withColumn("hash_linha", 
    F.udf(lambda r: calculate_row_hash(r), StringType())(
        F.struct("fornecedor_id", "ordem_id", "nome", "ativo_id", "data", "valor")
    ))

# Adicionar colunas de metadados
df = df.withColumn("job_id", F.lit(job_id)) \
       .withColumn("file_hash", F.lit(file_hash)) \
       .withColumn("timestamp", F.current_timestamp())

# Separar linhas válidas e inválidas
df_valid = df.filter(F.col("is_valid") == True)
df_invalid = df.filter(F.col("is_valid") == False)

# Remover colunas de validação intermediárias antes de salvar
columns_to_keep = ["fornecedor_id", "ordem_id", "nome", "ativo_id", "data", "valor", 
                   "hash_linha", "job_id", "file_hash", "timestamp"]

df_valid_clean = df_valid.select(columns_to_keep)

# Detectar duplicatas (linhas com mesmos valores em todos os campos)
df_valid_clean = df_valid_clean.withColumn("is_duplicate", 
    F.count("*").over(
        F.window(
            F.struct("fornecedor_id", "ordem_id", "nome", "ativo_id", "data", "valor"),
            rowsBetween(-sys.maxsize, sys.maxsize)
        )
    ) > 1
)

# Particionar por fornecedor_id e hash de ordem_id para balanceamento
df_valid_clean = df_valid_clean.withColumn("partition_key", 
    F.concat(F.col("fornecedor_id"), F.lit("_"), 
             F.substring(F.col("hash_linha"), 1, 4)))

# Salvar dados válidos em Parquet
output_path = f"s3://reconciliation-intermediate/{object_key.split('/')[0]}/validated.parquet"

print(f"Salvando dados válidos em: {output_path}")

df_valid_clean.write \
    .mode("overwrite") \
    .partitionBy("fornecedor_id") \
    .parquet(output_path)

# Salvar dados inválidos (para relatório de erros)
invalid_output_path = f"s3://reconciliation-intermediate/{object_key.split('/')[0]}/invalid.parquet"

df_invalid.select(columns_to_keep + ["is_valid_fornecedor_id", "is_valid_ordem_id", 
                                     "is_valid_nome", "is_valid_ativo_id", 
                                     "is_valid_data", "is_valid_valor"]).write \
    .mode("overwrite") \
    .parquet(invalid_output_path)

# Coletar métricas
total_lines = df.count()
valid_lines = df_valid.count()
invalid_lines = df_invalid.count()
duplicate_lines = df_valid_clean.filter(F.col("is_duplicate") == True).count()

# Publicar métricas no CloudWatch
try:
    cloudwatch.put_metric_data(
        Namespace='Reconciliation',
        MetricData=[
            {
                'MetricName': 'TotalLines',
                'Value': total_lines,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Stage', 'Value': 'Validation'}
                ]
            },
            {
                'MetricName': 'ValidLines',
                'Value': valid_lines,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Stage', 'Value': 'Validation'}
                ]
            },
            {
                'MetricName': 'InvalidLines',
                'Value': invalid_lines,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Stage', 'Value': 'Validation'}
                ]
            },
            {
                'MetricName': 'DuplicateLines',
                'Value': duplicate_lines,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Stage', 'Value': 'Validation'}
                ]
            }
        ]
    )
except Exception as e:
    print(f"Erro ao publicar métricas: {str(e)}")

# Log estruturado
log_data = {
    "timestamp": datetime.utcnow().isoformat(),
    "level": "INFO",
    "job_id": job_id,
    "event": "validation_completed",
    "metrics": {
        "total_lines": total_lines,
        "valid_lines": valid_lines,
        "invalid_lines": invalid_lines,
        "duplicate_lines": duplicate_lines
    },
    "output_path": output_path
}

print(json.dumps(log_data))

# Finalizar job
job.commit()

print(f"Validação concluída. Total: {total_lines}, Válidas: {valid_lines}, Inválidas: {invalid_lines}")
```

---

### 2. Criar Glue Job no AWS Console ou CLI

**Passos:**

2.1. Fazer upload do script para S3:
```bash
aws s3 cp glue_validation_job.py s3://reconciliation-scripts/glue/reconciliation-validation.py
```

2.2. Criar Glue Job:
```bash
aws glue create-job \
  --name reconciliation-validation \
  --role ReconciliationGlueRole \
  --command '{
    "Name": "glueetl",
    "ScriptLocation": "s3://reconciliation-scripts/glue/reconciliation-validation.py",
    "PythonVersion": "3"
  }' \
  --default-arguments '{
    "--TempDir": "s3://reconciliation-intermediate/temp/",
    "--enable-metrics": "",
    "--enable-spark-ui": "true",
    "--spark-event-logs-path": "s3://reconciliation-intermediate/spark-logs/"
  }' \
  --max-retries 0 \
  --timeout 10 \
  --allocated-capacity 10 \
  --glue-version "4.0"
```

---

### 3. Configurar Particionamento e Performance

**Passos:**

3.1. Otimizar configurações Spark:
```python
# Adicionar no início do script
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

3.2. Configurar número de partições:
```python
# Após ler CSV, reparticionar se necessário
num_partitions = max(df.rdd.getNumPartitions(), 100)  # Mínimo 100 partições
df = df.repartition(num_partitions)
```

---

## Validação

### Checklist de Validação:

- [ ] Script Glue criado e testado localmente (se possível)
- [ ] Glue Job criado no AWS
- [ ] IAM Role configurada corretamente
- [ ] Script uploadado para S3
- [ ] Teste com CSV pequeno (100 linhas)
- [ ] Teste com CSV médio (10.000 linhas)
- [ ] Verificar Parquet gerado no S3
- [ ] Verificar métricas no CloudWatch
- [ ] Verificar logs no CloudWatch
- [ ] Validar schema do Parquet gerado

### Comandos de Validação:

```bash
# Listar Glue Jobs
aws glue list-jobs --query 'JobNames[?contains(@, `reconciliation`)]'

# Verificar configuração do job
aws glue get-job --job-name reconciliation-validation

# Executar job manualmente (teste)
aws glue start-job-run \
  --job-name reconciliation-validation \
  --arguments '{
    "--job_id": "TEST-001",
    "--bucket_name": "reconciliation-ingress",
    "--object_key": "2026-02-16/test.csv",
    "--file_hash": "test-hash"
  }'

# Verificar execução
aws glue get-job-run \
  --job-name reconciliation-validation \
  --run-id <RUN_ID>

# Verificar Parquet gerado
aws s3 ls s3://reconciliation-intermediate/2026-02-16/validated.parquet/ --recursive

# Ler Parquet para validação (usando Spark local ou Glue)
```

---

## Notas Técnicas

- **DPU:** Configurar 5-8 DPU para validação (conforme Documento Técnico - Seção 5.3)
- **Timeout:** Configurar timeout adequado (10 minutos para validação)
- **Particionamento:** Particionar por `fornecedor_id` para melhorar performance na conciliação
- **Validação:** Implementar todas as validações conforme PRD (Seção 2.2)
- **Duplicatas:** Identificar e marcar duplicatas (primeira ocorrência mantida)

---

## Próximas Tasks

- **TASK 07:** Implementar Glue Job de Conciliação (lerá Parquet validado desta task)

---

## Referências

- Documento Técnico - Seção 3.2 (Componentes da Arquitetura)
- Documento Técnico - Seção 5.2 (Breakdown por Etapa)
- PRD - Seção 2.2 (Validações Iniciais)
- AWS Glue Documentation: https://docs.aws.amazon.com/glue/
