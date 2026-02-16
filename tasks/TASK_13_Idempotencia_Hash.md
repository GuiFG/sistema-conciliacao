# TASK 13: Implementar Idempotência e Hash

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** MÉDIA  
**Estimativa:** 1-2 dias  
**Dependências:** TASK 06 (Glue Job Validação)

---

## Objetivo

Garantir idempotência do sistema através de hash de arquivos e linhas, evitando processamento duplicado e garantindo rastreabilidade completa.

---

## Requisitos

- Sistema de processamento funcionando
- Hash já parcialmente implementado nas tasks anteriores

---

## Passo a Passo Detalhado

### 1. Validar Implementação de Hash de Arquivo

**Já implementado na TASK 03 (Lambda Trigger):**

- Hash SHA-256 do arquivo CSV completo
- Armazenado no input do Step Functions
- Usado para verificar se arquivo já foi processado

**Validação:**

```python
# Verificar se arquivo já foi processado
def check_file_already_processed(file_hash):
    """
    Verifica se arquivo com mesmo hash já foi processado
    """
    # Consultar DynamoDB ou S3 para verificar hash
    # Retornar True se já processado, False caso contrário
    pass
```

---

### 2. Validar Implementação de Hash de Linha

**Já implementado na TASK 06 (Glue Job Validação):**

- Hash SHA-256 de cada linha processada
- Calculado com base em todos os campos: `fornecedor_id|ordem_id|nome|ativo_id|data|valor`
- Armazenado no Parquet de saída

**Validação:**

```python
# Verificar cálculo de hash
def calculate_row_hash(row):
    """
    Calcula hash SHA-256 da linha
    """
    import hashlib
    row_str = f"{row.fornecedor_id}|{row.ordem_id}|{row.nome}|{row.ativo_id}|{row.data}|{row.valor}"
    return hashlib.sha256(row_str.encode()).hexdigest()
```

---

### 3. Implementar Verificação de Duplicação de Arquivo

**Passos:**

3.1. Criar tabela DynamoDB para rastrear arquivos processados:
```bash
aws dynamodb create-table \
  --table-name reconciliation-processed-files \
  --attribute-definitions \
    AttributeName=file_hash,AttributeType=S \
  --key-schema \
    AttributeName=file_hash,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

3.2. Adicionar verificação na Lambda Trigger:
```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('reconciliation-processed-files')

def check_file_already_processed(file_hash):
    """
    Verifica se arquivo já foi processado
    """
    try:
        response = table.get_item(Key={'file_hash': file_hash})
        if 'Item' in response:
            return True, response['Item']
        return False, None
    except Exception as e:
        print(f"Erro ao verificar arquivo: {str(e)}")
        return False, None

def mark_file_as_processed(file_hash, job_id, timestamp):
    """
    Marca arquivo como processado
    """
    try:
        table.put_item(
            Item={
                'file_hash': file_hash,
                'job_id': job_id,
                'timestamp': timestamp,
                'status': 'PROCESSED'
            }
        )
    except Exception as e:
        print(f"Erro ao marcar arquivo: {str(e)}")

# No lambda_handler, antes de iniciar processamento:
already_processed, item = check_file_already_processed(file_hash)

if already_processed:
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Arquivo já foi processado anteriormente',
            'previous_job_id': item['job_id'],
            'previous_timestamp': item['timestamp']
        })
    }
```

---

### 4. Implementar Verificação de Duplicação de Linha

**Passos:**

4.1. Adicionar verificação no Glue Job de Conciliação:
```python
# Verificar se linha já foi processada em execução anterior
# Usar hash_linha para identificar duplicatas entre execuções

# Opção 1: Verificar no DynamoDB (para linhas críticas)
# Opção 2: Verificar no Parquet de saída anterior (mais simples)

def check_line_already_processed(hash_linha, previous_output_path):
    """
    Verifica se linha já foi processada em execução anterior
    """
    try:
        df_previous = spark.read.parquet(previous_output_path)
        if df_previous.filter(F.col("hash_linha") == hash_linha).count() > 0:
            return True
        return False
    except:
        return False
```

---

### 5. Implementar Rastreabilidade Completa

**Passos:**

5.1. Garantir que hash_linha está presente em todos os registros:
- CSV validado: ✅ (TASK 06)
- Divergências: ✅ (TASK 07)
- Parquet QuickSight: ✅ (TASK 09)

5.2. Adicionar hash_linha aos logs estruturados:
```python
log_data = {
    "timestamp": datetime.utcnow().isoformat(),
    "level": "INFO",
    "job_id": job_id,
    "event": "line_processed",
    "hash_linha": hash_linha,
    "ordem_id": ordem_id
}
```

---

## Validação

### Checklist de Validação:

- [ ] Hash de arquivo calculado corretamente
- [ ] Hash de linha calculado corretamente
- [ ] Verificação de duplicação de arquivo funcionando
- [ ] Arquivos duplicados não são reprocessados
- [ ] Hash_linha presente em todos os registros de saída
- [ ] Rastreabilidade completa garantida
- [ ] Teste: Reprocessar mesmo arquivo (deve ser ignorado)
- [ ] Teste: Verificar hash em logs e saídas

### Testes:

1. **Teste de Idempotência de Arquivo:**
   - Upload mesmo arquivo CSV duas vezes
   - Verificar que segunda execução é ignorada ou retorna informação de duplicação

2. **Teste de Hash de Linha:**
   - Processar CSV com linhas conhecidas
   - Verificar que hash_linha está presente e correto
   - Comparar hash entre execuções (deve ser igual)

3. **Teste de Rastreabilidade:**
   - Processar arquivo
   - Verificar que hash_linha pode ser usado para rastrear linha específica
   - Verificar que hash aparece em logs e saídas

---

## Notas Técnicas

- **Hash de Arquivo:** Usar SHA-256 para garantir unicidade
- **Hash de Linha:** Usar SHA-256 com todos os campos concatenados
- **Idempotência:** Garantir que reprocessamento não cria duplicatas
- **Rastreabilidade:** Hash permite rastrear linha específica em auditoria

---

## Próximas Tasks

- **TASK 14:** Testes End-to-End (validar idempotência)

---

## Referências

- PRD - Seção 6.4 (Rastreabilidade de Linha)
- Documento Técnico - Seção 10.5 (Compliance e Auditoria)
