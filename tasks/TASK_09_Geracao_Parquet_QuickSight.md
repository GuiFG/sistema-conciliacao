# TASK 09: Implementar Geração de Parquet Otimizado para QuickSight

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** ALTA  
**Estimativa:** 2-3 dias  
**Dependências:** TASK 07 (Glue Job Conciliação)

---

## Objetivo

Implementar Glue Job que prepara dados de divergências em formato Parquet otimizado para consumo pelo Amazon QuickSight, incluindo otimização de schema e particionamento.

---

## Requisitos

- Glue Job de Conciliação funcionando (TASK 07)
- Bucket S3 de output configurado
- Glue Data Catalog configurado (TASK 10)

---

## Passo a Passo Detalhado

### 1. Criar Script do Glue Job

**Arquivo:** `glue_quicksight_prep_job.py`

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
from datetime import datetime
import json

# Inicializar contexto
args = getResolvedOptions(sys.argv, [
    'JOB_NAME',
    'job_id',
    'divergences_path'
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Parâmetros
job_id = args['job_id']
divergences_path = args['divergences_path']

# CloudWatch para métricas
cloudwatch = boto3.client('cloudwatch')

print(f"Lendo divergências de: {divergences_path}")

# Ler Parquet de divergências
df_divergences = spark.read.parquet(divergences_path)

# Otimizar schema para QuickSight
# Garantir tipos corretos e nullable apropriado
df_optimized = df_divergences.select(
    F.col("fornecedor_id").cast(StringType()).alias("fornecedor_id"),
    F.col("ordem_id").cast(StringType()).alias("ordem_id"),
    F.col("nome").cast(StringType()).alias("nome"),
    F.col("ativo_id").cast(StringType()).alias("ativo_id"),
    F.col("data").cast(DateType()).alias("data"),
    F.col("valor_csv").cast(DecimalType(10, 2)).alias("valor_csv"),
    F.col("valor_base").cast(DecimalType(10, 2)).alias("valor_base"),
    F.col("tipo_divergencia").cast(StringType()).alias("tipo_divergencia"),
    F.col("detalhes").cast(StringType()).alias("detalhes"),
    F.col("severidade").cast(StringType()).alias("severidade"),
    F.col("hash_linha").cast(StringType()).alias("hash_linha"),
    F.col("job_id").cast(StringType()).alias("job_id"),
    F.col("timestamp").cast(TimestampType()).alias("timestamp"),
    # Campos calculados adicionais
    F.abs(F.col("valor_csv") - F.col("valor_base")).alias("diferenca_valor"),
    F.when(
        F.col("valor_base") > 0,
        (F.abs(F.col("valor_csv") - F.col("valor_base")) / F.col("valor_base")) * 100
    ).otherwise(F.lit(None)).cast(DecimalType(10, 2)).alias("percentual_divergencia"),
    # Prioridade para ordenação
    F.when(F.col("tipo_divergencia") == "TIPO_2_AUSENTE_CSV", 1)
     .when(F.col("tipo_divergencia") == "TIPO_1_AUSENTE_BASE", 2)
     .when((F.col("tipo_divergencia") == "TIPO_3_VALOR") & (F.col("severidade") == "HIGH"), 3)
     .when((F.col("tipo_divergencia") == "TIPO_3_VALOR") & (F.col("severidade") == "MEDIUM"), 4)
     .when(F.col("tipo_divergencia") == "TIPO_3_DATA", 5)
     .when((F.col("tipo_divergencia") == "TIPO_3_VALOR") & (F.col("severidade") == "LOW"), 6)
     .otherwise(99).alias("priority")
)

# Ordenar por prioridade
df_optimized = df_optimized.orderBy("priority", "fornecedor_id", "ordem_id")

# Extrair data para particionamento
date_str = datetime.utcnow().strftime('%Y-%m-%d')
df_optimized = df_optimized.withColumn("partition_date", F.lit(date_str))

# Salvar em Parquet otimizado para QuickSight
output_path = f"s3://reconciliation-output/{date_str}/divergencias.parquet"

print(f"Salvando dados otimizados em: {output_path}")

# Configurações de otimização
df_optimized.write \
    .mode("overwrite") \
    .option("compression", "snappy") \
    .option("parquet.block.size", "134217728") \
    .option("parquet.page.size", "1048576") \
    .partitionBy("fornecedor_id", "data") \
    .parquet(output_path)

# Coletar estatísticas
total_divergences = df_optimized.count()
tipo1_count = df_optimized.filter(F.col("tipo_divergencia") == "TIPO_1_AUSENTE_BASE").count()
tipo2_count = df_optimized.filter(F.col("tipo_divergencia") == "TIPO_2_AUSENTE_CSV").count()
tipo3_valor_count = df_optimized.filter(F.col("tipo_divergencia") == "TIPO_3_VALOR").count()
tipo3_data_count = df_optimized.filter(F.col("tipo_divergencia") == "TIPO_3_DATA").count()

# Publicar métricas
try:
    cloudwatch.put_metric_data(
        Namespace='Reconciliation',
        MetricData=[
            {
                'MetricName': 'QuickSightPrepCompleted',
                'Value': 1,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id}
                ]
            },
            {
                'MetricName': 'TotalDivergencesOutput',
                'Value': total_divergences,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id}
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
    "event": "quicksight_prep_completed",
    "output_path": output_path,
    "metrics": {
        "total_divergences": total_divergences,
        "tipo1": tipo1_count,
        "tipo2": tipo2_count,
        "tipo3_valor": tipo3_valor_count,
        "tipo3_data": tipo3_data_count
    }
}

print(json.dumps(log_data))

# Finalizar job
job.commit()

print(f"Preparação QuickSight concluída. Total de divergências: {total_divergences}")
```

---

### 2. Criar Glue Job

**Passos:**

2.1. Upload do script:
```bash
aws s3 cp glue_quicksight_prep_job.py s3://reconciliation-scripts/glue/reconciliation-quicksight-prep.py
```

2.2. Criar Glue Job:
```bash
aws glue create-job \
  --name reconciliation-quicksight-prep \
  --role ReconciliationGlueRole \
  --command '{
    "Name": "glueetl",
    "ScriptLocation": "s3://reconciliation-scripts/glue/reconciliation-quicksight-prep.py",
    "PythonVersion": "3"
  }' \
  --default-arguments '{
    "--TempDir": "s3://reconciliation-intermediate/temp/",
    "--enable-metrics": "",
    "--enable-spark-ui": "true"
  }' \
  --max-retries 2 \
  --timeout 10 \
  --allocated-capacity 5 \
  --glue-version "4.0"
```

---

### 3. Otimizar Schema para QuickSight

**Passos:**

3.1. Garantir tipos de dados corretos:
- `valor_csv` e `valor_base`: DecimalType(10, 2) para precisão monetária
- `data`: DateType() para filtros temporais eficientes
- `timestamp`: TimestampType() para ordenação temporal

3.2. Configurar compressão:
- Usar Snappy para balance entre compressão e velocidade de leitura

3.3. Configurar tamanho de blocos:
- `parquet.block.size`: 128MB (padrão otimizado)
- `parquet.page.size`: 1MB (padrão)

---

### 4. Configurar Particionamento

**Passos:**

4.1. Particionar por:
- `fornecedor_id`: Para filtros por fornecedor
- `data`: Para filtros temporais e queries eficientes

4.2. Benefícios:
- Reduz I/O ao ler apenas partições relevantes
- Melhora performance de queries no QuickSight
- Facilita manutenção de dados históricos

---

## Validação

### Checklist de Validação:

- [ ] Script Glue criado
- [ ] Glue Job criado no AWS
- [ ] Parquet gerado no S3 output
- [ ] Schema validado (tipos corretos)
- [ ] Particionamento funcionando
- [ ] Compressão configurada (Snappy)
- [ ] Teste: Ler Parquet gerado
- [ ] Validar que dados estão completos
- [ ] Verificar métricas no CloudWatch

### Comandos de Validação:

```bash
# Verificar Parquet gerado
aws s3 ls s3://reconciliation-output/2026-02-16/divergencias.parquet/ --recursive

# Validar schema (usando Spark ou Glue)
# Ler Parquet e verificar schema
```

---

## Notas Técnicas

- **DPU:** Configurar 5-8 DPU (conforme Documento Técnico - Seção 5.3)
- **Compressão:** Snappy é ideal para QuickSight (balance entre tamanho e velocidade)
- **Particionamento:** Particionar por `fornecedor_id` e `data` para queries eficientes
- **Schema:** Garantir tipos corretos para evitar erros no QuickSight

---

## Próximas Tasks

- **TASK 10:** Configurar Glue Data Catalog (catalogar Parquet gerado)
- **TASK 11:** Configurar QuickSight Dataset e Dashboard (consumir Parquet)

---

## Referências

- Documento Técnico - Seção 4.1 (Configuração do QuickSight)
- Documento Técnico - Seção 4.2 (Estrutura de Dados para QuickSight)
- AWS Glue Data Catalog: https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html
