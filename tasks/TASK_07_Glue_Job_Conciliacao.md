# TASK 07: Implementar Glue Job de Conciliação

**Sprint:** 3-4 (Processamento Core)  
**Prioridade:** CRÍTICA  
**Estimativa:** 4-5 dias  
**Dependências:** TASK 06 (Glue Job Validação), RDS configurado

---

## Objetivo

Implementar AWS Glue Job que realiza a conciliação entre o CSV validado e a base interna (RDS), identificando divergências Tipo 1, Tipo 2 e Tipo 3.

---

## Requisitos

- Glue Job de Validação funcionando (TASK 06)
- RDS acessível do Glue (Security Groups configurados)
- Índice único em `ordem_id` no RDS
- Credenciais RDS configuradas (secrets manager ou parâmetros)

---

## Passo a Passo Detalhado

### 1. Criar Script do Glue Job

**Arquivo:** `glue_reconciliation_job.py`

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
    'validated_data_path'
])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Parâmetros
job_id = args['job_id']
validated_data_path = args['validated_data_path']

# CloudWatch para métricas
cloudwatch = boto3.client('cloudwatch')

# Configurações RDS (obter de Secrets Manager ou variáveis de ambiente)
RDS_HOST = "your-rds-endpoint.rds.amazonaws.com"
RDS_PORT = "5432"
RDS_DATABASE = "your_database"
RDS_TABLE = "transactions"  # Tabela da base interna
RDS_USER = "reconciliation_glue"
RDS_PASSWORD = "SECURE_PASSWORD"  # Obter de Secrets Manager

# URL de conexão JDBC
jdbc_url = f"jdbc:postgresql://{RDS_HOST}:{RDS_PORT}/{RDS_DATABASE}"
connection_properties = {
    "user": RDS_USER,
    "password": RDS_PASSWORD,
    "driver": "org.postgresql.Driver"
}

print(f"Lendo dados validados de: {validated_data_path}")

# Ler Parquet validado
df_csv = spark.read.parquet(validated_data_path)

# Filtrar apenas linhas válidas e não duplicadas
df_csv = df_csv.filter(
    (F.col("is_valid") == True) & 
    (F.col("is_duplicate") == False)
)

print(f"Total de linhas válidas do CSV: {df_csv.count()}")

# Ler base interna do RDS
print("Lendo base interna do RDS...")

# Estratégia 1: Ler toda a tabela (se pequena)
# df_base = spark.read.jdbc(
#     url=jdbc_url,
#     table=RDS_TABLE,
#     properties=connection_properties
# )

# Estratégia 2: Ler apenas registros relevantes (recomendado)
# Filtrar por data e fornecedores presentes no CSV
dates = [row['data'] for row in df_csv.select("data").distinct().collect()]
fornecedores = [row['fornecedor_id'] for row in df_csv.select("fornecedor_id").distinct().collect()]

# Construir query SQL com filtros
query = f"""
(SELECT ordem_id, fornecedor_id, nome, ativo_id, data, valor
 FROM {RDS_TABLE}
 WHERE data IN ({','.join([f"'{d}'" for d in dates])})
   AND fornecedor_id IN ({','.join([f"'{f}'" for f in fornecedores])})
) AS filtered_base
"""

df_base = spark.read.jdbc(
    url=jdbc_url,
    table=query,
    properties=connection_properties
)

print(f"Total de linhas da base interna: {df_base.count()}")

# Adicionar colunas de metadados à base interna
df_base = df_base.withColumn("job_id", F.lit(job_id)) \
                 .withColumn("timestamp", F.current_timestamp())

# Renomear colunas da base para evitar conflito
df_base_renamed = df_base.select(
    F.col("ordem_id").alias("ordem_id_base"),
    F.col("fornecedor_id").alias("fornecedor_id_base"),
    F.col("nome").alias("nome_base"),
    F.col("ativo_id").alias("ativo_id_base"),
    F.col("data").alias("data_base"),
    F.col("valor").alias("valor_base"),
    F.col("job_id").alias("job_id_base"),
    F.col("timestamp").alias("timestamp_base")
)

# Renomear colunas do CSV
df_csv_renamed = df_csv.select(
    F.col("ordem_id").alias("ordem_id_csv"),
    F.col("fornecedor_id").alias("fornecedor_id_csv"),
    F.col("nome").alias("nome_csv"),
    F.col("ativo_id").alias("ativo_id_csv"),
    F.col("data").alias("data_csv"),
    F.col("valor").alias("valor_csv"),
    F.col("hash_linha").alias("hash_linha_csv"),
    F.col("job_id").alias("job_id_csv"),
    F.col("timestamp").alias("timestamp_csv")
)

# JOIN por ordem_id (chave única de correspondência)
df_joined = df_csv_renamed.join(
    df_base_renamed,
    df_csv_renamed.ordem_id_csv == df_base_renamed.ordem_id_base,
    "full_outer"
)

# Classificar divergências

# TIPO 1: Presente no CSV, Ausente na Base
df_tipo1 = df_joined.filter(
    F.col("ordem_id_csv").isNotNull() & 
    F.col("ordem_id_base").isNull()
).select(
    F.col("fornecedor_id_csv").alias("fornecedor_id"),
    F.col("ordem_id_csv").alias("ordem_id"),
    F.col("nome_csv").alias("nome"),
    F.col("ativo_id_csv").alias("ativo_id"),
    F.col("data_csv").alias("data"),
    F.col("valor_csv").alias("valor_csv"),
    F.lit(None).cast(DecimalType(10, 2)).alias("valor_base"),
    F.lit("TIPO_1_AUSENTE_BASE").alias("tipo_divergencia"),
    F.lit("Transação presente no CSV mas não encontrada na base interna").alias("detalhes"),
    F.lit("HIGH").alias("severidade"),
    F.col("hash_linha_csv").alias("hash_linha"),
    F.col("job_id_csv").alias("job_id"),
    F.col("timestamp_csv").alias("timestamp")
)

# TIPO 2: Presente na Base, Ausente no CSV
df_tipo2 = df_joined.filter(
    F.col("ordem_id_base").isNotNull() & 
    F.col("ordem_id_csv").isNull()
).select(
    F.col("fornecedor_id_base").alias("fornecedor_id"),
    F.col("ordem_id_base").alias("ordem_id"),
    F.col("nome_base").alias("nome"),
    F.col("ativo_id_base").alias("ativo_id"),
    F.col("data_base").alias("data"),
    F.lit(None).cast(DecimalType(10, 2)).alias("valor_csv"),
    F.col("valor_base").alias("valor_base"),
    F.lit("TIPO_2_AUSENTE_CSV").alias("tipo_divergencia"),
    F.lit("Transação presente na base interna mas não encontrada no CSV").alias("detalhes"),
    F.lit("HIGH").alias("severidade"),
    F.lit(None).cast(StringType()).alias("hash_linha"),
    F.col("job_id_base").alias("job_id"),
    F.col("timestamp_base").alias("timestamp")
)

# TIPO 3: Presente em Ambos, mas com Campos Divergentes
df_tipo3 = df_joined.filter(
    F.col("ordem_id_csv").isNotNull() & 
    F.col("ordem_id_base").isNotNull()
)

# Comparar valores (diferença absoluta = 0, sem tolerância)
df_tipo3_valor = df_tipo3.filter(
    F.abs(F.col("valor_csv") - F.col("valor_base")) > 0
).select(
    F.col("fornecedor_id_csv").alias("fornecedor_id"),
    F.col("ordem_id_csv").alias("ordem_id"),
    F.col("nome_csv").alias("nome"),
    F.col("ativo_id_csv").alias("ativo_id"),
    F.col("data_csv").alias("data"),
    F.col("valor_csv").alias("valor_csv"),
    F.col("valor_base").alias("valor_base"),
    F.lit("TIPO_3_VALOR").alias("tipo_divergencia"),
    (F.lit("Diferença de R$ ") + 
     F.format_number(F.abs(F.col("valor_csv") - F.col("valor_base")), 2)).alias("detalhes"),
    F.when(
        F.abs(F.col("valor_csv") - F.col("valor_base")) > 100,
        F.lit("HIGH")
    ).otherwise(
        F.when(
            F.abs(F.col("valor_csv") - F.col("valor_base")) > 0.01,
            F.lit("MEDIUM")
        ).otherwise(F.lit("LOW"))
    ).alias("severidade"),
    F.col("hash_linha_csv").alias("hash_linha"),
    F.col("job_id_csv").alias("job_id"),
    F.col("timestamp_csv").alias("timestamp")
)

# Comparar datas (diferença de data)
df_tipo3_data = df_tipo3.filter(
    F.col("data_csv") != F.col("data_base")
).select(
    F.col("fornecedor_id_csv").alias("fornecedor_id"),
    F.col("ordem_id_csv").alias("ordem_id"),
    F.col("nome_csv").alias("nome"),
    F.col("ativo_id_csv").alias("ativo_id"),
    F.col("data_csv").alias("data"),
    F.col("valor_csv").alias("valor_csv"),
    F.col("valor_base").alias("valor_base"),
    F.lit("TIPO_3_DATA").alias("tipo_divergencia"),
    (F.lit("Diferença de data. CSV: ") + 
     F.col("data_csv").cast(StringType()) + 
     F.lit(", Base: ") + 
     F.col("data_base").cast(StringType())).alias("detalhes"),
    F.lit("MEDIUM").alias("severidade"),
    F.col("hash_linha_csv").alias("hash_linha"),
    F.col("job_id_csv").alias("job_id"),
    F.col("timestamp_csv").alias("timestamp")
)

# Unir todas as divergências
df_divergences = df_tipo1.union(df_tipo2).union(df_tipo3_valor).union(df_tipo3_data)

# Ordenar por prioridade (severidade e tipo)
from pyspark.sql.window import Window

df_divergences = df_divergences.withColumn(
    "priority",
    F.when(F.col("severidade") == "HIGH", 1)
     .when(F.col("severidade") == "MEDIUM", 2)
     .otherwise(3)
).orderBy("priority", "tipo_divergencia")

# Salvar divergências em Parquet
output_path = f"s3://reconciliation-intermediate/{job_id.split('-')[1]}/divergences.parquet"

print(f"Salvando divergências em: {output_path}")

df_divergences.write \
    .mode("overwrite") \
    .partitionBy("fornecedor_id", "data") \
    .parquet(output_path)

# Coletar métricas
total_divergences = df_divergences.count()
tipo1_count = df_tipo1.count()
tipo2_count = df_tipo2.count()
tipo3_valor_count = df_tipo3_valor.count()
tipo3_data_count = df_tipo3_data.count()

# Publicar métricas no CloudWatch
try:
    cloudwatch.put_metric_data(
        Namespace='Reconciliation',
        MetricData=[
            {
                'MetricName': 'TotalDivergences',
                'Value': total_divergences,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Stage', 'Value': 'Reconciliation'}
                ]
            },
            {
                'MetricName': 'DivergencesByType',
                'Value': tipo1_count,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Type', 'Value': 'TIPO_1'}
                ]
            },
            {
                'MetricName': 'DivergencesByType',
                'Value': tipo2_count,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Type', 'Value': 'TIPO_2'}
                ]
            },
            {
                'MetricName': 'DivergencesByType',
                'Value': tipo3_valor_count,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Type', 'Value': 'TIPO_3_VALOR'}
                ]
            },
            {
                'MetricName': 'DivergencesByType',
                'Value': tipo3_data_count,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'JobId', 'Value': job_id},
                    {'Name': 'Type', 'Value': 'TIPO_3_DATA'}
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
    "event": "reconciliation_completed",
    "metrics": {
        "total_divergences": total_divergences,
        "tipo1": tipo1_count,
        "tipo2": tipo2_count,
        "tipo3_valor": tipo3_valor_count,
        "tipo3_data": tipo3_data_count
    },
    "output_path": output_path
}

print(json.dumps(log_data))

# Finalizar job
job.commit()

print(f"Conciliação concluída. Total de divergências: {total_divergences}")
```

---

### 2. Configurar Conexão RDS

**Passos:**

2.1. Obter credenciais do Secrets Manager (recomendado):
```python
import boto3
import json

secrets_client = boto3.client('secretsmanager')
secret = secrets_client.get_secret_value(SecretId='reconciliation-rds-credentials')
credentials = json.loads(secret['SecretString'])

RDS_USER = credentials['username']
RDS_PASSWORD = credentials['password']
```

2.2. Configurar JDBC Driver no Glue:
   - Adicionar JDBC driver (PostgreSQL ou MySQL) ao Glue Job
   - Configurar `--extra-jars` no Glue Job

---

### 3. Otimizar Performance

**Passos:**

3.1. Configurar connection pooling:
```python
connection_properties = {
    "user": RDS_USER,
    "password": RDS_PASSWORD,
    "driver": "org.postgresql.Driver",
    "numPartitions": "10",  # Número de partições para leitura paralela
    "partitionColumn": "ordem_id",  # Coluna para particionar
    "lowerBound": "1",
    "upperBound": "1000000"
}
```

3.2. Configurar Spark para otimizar joins:
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

---

## Validação

### Checklist de Validação:

- [ ] Script Glue criado
- [ ] Conexão RDS testada
- [ ] JDBC driver configurado
- [ ] Glue Job criado no AWS
- [ ] Teste com dados pequenos (100 linhas)
- [ ] Verificar divergências Tipo 1 identificadas
- [ ] Verificar divergências Tipo 2 identificadas
- [ ] Verificar divergências Tipo 3 identificadas
- [ ] Verificar Parquet gerado
- [ ] Verificar métricas no CloudWatch
- [ ] Validar performance (deve processar 1M linhas em ≤ 15 minutos)

---

## Notas Técnicas

- **DPU:** Configurar 10-15 DPU para conciliação (conforme Documento Técnico - Seção 5.3)
- **Timeout:** Configurar timeout de 20 minutos (SLA)
- **RDS Performance:** Garantir índice único em `ordem_id` no RDS
- **Connection Pooling:** Configurar adequadamente para evitar esgotamento de conexões
- **Particionamento:** Particionar leitura do RDS por `ordem_id` ou `fornecedor_id`

---

## Próximas Tasks

- **TASK 08:** Implementar Classificação de Divergências (já incluída nesta task, mas pode ser refinada)
- **TASK 09:** Implementar Glue Job de Preparação QuickSight (lerá divergências desta task)

---

## Referências

- Documento Técnico - Seção 3.2 (Componentes da Arquitetura)
- Documento Técnico - Seção 2.1 (Chave de Correspondência)
- PRD - Seção 3.2 (Tipos de Divergência)
- AWS Glue JDBC Connection: https://docs.aws.amazon.com/glue/latest/dg/set-up-jdbc-connection.html
