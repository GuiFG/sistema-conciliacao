# TASK 10: Configurar Glue Data Catalog

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** ALTA  
**Estimativa:** 1 dia  
**Dependências:** TASK 09 (Geração Parquet QuickSight)

---

## Objetivo

Configurar Glue Data Catalog para catalogar tabela de divergências, permitindo que QuickSight acesse os dados de forma estruturada.

---

## Requisitos

- Parquet gerado no S3 (TASK 09)
- Permissões IAM para Glue Data Catalog

---

## Passo a Passo Detalhado

### 1. Criar Database no Glue Data Catalog

**Passos:**

1.1. Criar database:
```bash
aws glue create-database \
  --database-input '{
    "Name": "reconciliation",
    "Description": "Database para dados de conciliação financeira"
  }'
```

---

### 2. Criar Crawler para Catalogar Parquet

**Passos:**

2.1. Criar IAM Role para Crawler:
```bash
aws iam create-role \
  --role-name ReconciliationGlueCrawlerRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "glue.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Anexar política S3
aws iam attach-role-policy \
  --role-name ReconciliationGlueCrawlerRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
```

2.2. Criar Crawler:
```bash
aws glue create-crawler \
  --name reconciliation-divergences-crawler \
  --role ReconciliationGlueCrawlerRole \
  --database-name reconciliation \
  --targets '{
    "S3Targets": [
      {
        "Path": "s3://reconciliation-output/"
      }
    ]
  }' \
  --schema-change-policy '{
    "UpdateBehavior": "UPDATE_IN_DATABASE",
    "DeleteBehavior": "LOG"
  }'
```

---

### 3. Executar Crawler Após Cada Processamento

**Passos:**

3.1. Adicionar ao Glue Job de preparação QuickSight:
```python
# No final do script glue_quicksight_prep_job.py
glue_client = boto3.client('glue')

try:
    # Executar crawler para atualizar catálogo
    glue_client.start_crawler(Name='reconciliation-divergences-crawler')
    print("Crawler iniciado para atualizar catálogo")
except Exception as e:
    print(f"Erro ao iniciar crawler: {str(e)}")
```

---

### 4. Criar Tabela Manualmente (Alternativa)

**Se preferir criar tabela manualmente:**

```python
import boto3

glue_client = boto3.client('glue')

table_input = {
    'Name': 'divergences',
    'StorageDescriptor': {
        'Columns': [
            {'Name': 'fornecedor_id', 'Type': 'string'},
            {'Name': 'ordem_id', 'Type': 'string'},
            {'Name': 'nome', 'Type': 'string'},
            {'Name': 'ativo_id', 'Type': 'string'},
            {'Name': 'data', 'Type': 'date'},
            {'Name': 'valor_csv', 'Type': 'decimal(10,2)'},
            {'Name': 'valor_base', 'Type': 'decimal(10,2)'},
            {'Name': 'tipo_divergencia', 'Type': 'string'},
            {'Name': 'detalhes', 'Type': 'string'},
            {'Name': 'severidade', 'Type': 'string'},
            {'Name': 'hash_linha', 'Type': 'string'},
            {'Name': 'job_id', 'Type': 'string'},
            {'Name': 'timestamp', 'Type': 'timestamp'},
            {'Name': 'diferenca_valor', 'Type': 'decimal(10,2)'},
            {'Name': 'percentual_divergencia', 'Type': 'decimal(10,2)'},
            {'Name': 'priority', 'Type': 'int'}
        ],
        'Location': 's3://reconciliation-output/',
        'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat',
        'OutputFormat': 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
        'SerdeInfo': {
            'SerializationLibrary': 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe',
            'Parameters': {
                'serialization.format': '1'
            }
        },
        'Compressed': True,
        'NumberOfBuckets': -1,
        'StoredAsSubDirectories': False
    },
    'PartitionKeys': [
        {'Name': 'fornecedor_id', 'Type': 'string'},
        {'Name': 'data', 'Type': 'date'}
    ],
    'TableType': 'EXTERNAL_TABLE',
    'Parameters': {
        'classification': 'parquet',
        'typeOfData': 'file'
    }
}

glue_client.create_table(
    DatabaseName='reconciliation',
    TableInput=table_input
)
```

---

## Validação

### Checklist de Validação:

- [ ] Database criado no Glue Data Catalog
- [ ] Crawler criado e configurado
- [ ] Crawler executado com sucesso
- [ ] Tabela `divergences` criada
- [ ] Schema da tabela correto
- [ ] Partições detectadas corretamente
- [ ] QuickSight pode acessar tabela (após TASK 11)

### Comandos de Validação:

```bash
# Listar databases
aws glue get-databases --query 'DatabaseList[?contains(Name, `reconciliation`)]'

# Verificar tabela criada
aws glue get-table \
  --database-name reconciliation \
  --name divergences

# Listar partições
aws glue get-partitions \
  --database-name reconciliation \
  --table-name divergences

# Executar crawler manualmente
aws glue start-crawler --name reconciliation-divergences-crawler

# Verificar status do crawler
aws glue get-crawler --name reconciliation-divergences-crawler
```

---

## Notas Técnicas

- **Crawler vs Manual:** Crawler é mais fácil, mas criação manual oferece mais controle
- **Atualização:** Executar crawler após cada processamento para manter catálogo atualizado
- **Partições:** Garantir que partições são detectadas corretamente

---

## Próximas Tasks

- **TASK 11:** Configurar QuickSight Dataset e Dashboard (usará tabela catalogada)

---

## Referências

- AWS Glue Data Catalog: https://docs.aws.amazon.com/glue/latest/dg/catalog-and-crawler.html
- Documento Técnico - Seção 4.1 (Configuração do QuickSight)
