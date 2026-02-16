# TASK 08: Implementar Classificação de Divergências

**Sprint:** 3-4 (Processamento Core)  
**Prioridade:** ALTA  
**Estimativa:** 1-2 dias  
**Dependências:** TASK 07 (Glue Job Conciliação)

---

## Objetivo

Refinar e validar a lógica de classificação de divergências conforme regras de negócio definidas no PRD, incluindo cálculo de severidade e detalhes.

---

## Requisitos

- Glue Job de Conciliação implementado (TASK 07)
- Regras de negócio validadas (PRD)

---

## Passo a Passo Detalhado

### 1. Revisar Regras de Classificação

**Conforme PRD (Seção 3.2):**

**TIPO 1: Presente no CSV, Ausente na Base**
- Severidade: HIGH
- Detalhes: "Transação presente no CSV mas não encontrada na base interna"
- Ação recomendada: Investigar se transação foi perdida no processo interno

**TIPO 2: Presente na Base, Ausente no CSV**
- Severidade: HIGH
- Detalhes: "Transação presente na base interna mas não encontrada no CSV"
- Ação recomendada: Verificar se transação foi cancelada ou há atraso no envio

**TIPO 3: Presente em Ambos, mas com Campos Divergentes**
- **TIPO_3_VALOR:** Diferença no valor
  - Severidade HIGH: Diferença > R$ 100,00
  - Severidade MEDIUM: Diferença entre R$ 0,01 e R$ 100,00
  - Severidade LOW: Diferença ≤ R$ 0,01 (dentro da tolerância, mas ainda reportado)
- **TIPO_3_DATA:** Diferença na data
  - Severidade: MEDIUM
  - Detalhes: Mostrar datas do CSV e da base

---

### 2. Implementar Função de Classificação

**Adicionar ao script de conciliação:**

```python
def classify_divergence_type(row):
    """
    Classifica tipo de divergência e calcula severidade
    """
    if row.ordem_id_csv and not row.ordem_id_base:
        return {
            'tipo_divergencia': 'TIPO_1_AUSENTE_BASE',
            'severidade': 'HIGH',
            'detalhes': 'Transação presente no CSV mas não encontrada na base interna'
        }
    elif row.ordem_id_base and not row.ordem_id_csv:
        return {
            'tipo_divergencia': 'TIPO_2_AUSENTE_CSV',
            'severidade': 'HIGH',
            'detalhes': 'Transação presente na base interna mas não encontrada no CSV'
        }
    elif row.ordem_id_csv and row.ordem_id_base:
        # Verificar divergência de valor
        if row.valor_csv and row.valor_base:
            diff_valor = abs(float(row.valor_csv) - float(row.valor_base))
            if diff_valor > 0:
                if diff_valor > 100:
                    severidade = 'HIGH'
                elif diff_valor > 0.01:
                    severidade = 'MEDIUM'
                else:
                    severidade = 'LOW'
                
                return {
                    'tipo_divergencia': 'TIPO_3_VALOR',
                    'severidade': severidade,
                    'detalhes': f'Diferença de R$ {diff_valor:.2f}'
                }
        
        # Verificar divergência de data
        if row.data_csv != row.data_base:
            return {
                'tipo_divergencia': 'TIPO_3_DATA',
                'severidade': 'MEDIUM',
                'detalhes': f'Diferença de data. CSV: {row.data_csv}, Base: {row.data_base}'
            }
    
    return None
```

---

### 3. Implementar Priorização

**Conforme PRD (Seção 5.2.2):**

**Ordem de prioridade:**
1. TIPO_2_AUSENTE_CSV (Alta prioridade)
2. TIPO_1_AUSENTE_BASE (Alta prioridade)
3. TIPO_3_VALOR com diferença > R$ 100,00 (Alta prioridade)
4. TIPO_3_VALOR com diferença entre R$ 0,01 e R$ 100,00 (Média prioridade)
5. TIPO_3_VALOR dentro da tolerância (Baixa prioridade)
6. TIPO_3_DATA (Média prioridade)

**Implementação:**

```python
def calculate_priority(row):
    """
    Calcula prioridade numérica para ordenação
    """
    tipo = row.tipo_divergencia
    severidade = row.severidade
    
    if tipo == 'TIPO_2_AUSENTE_CSV':
        return 1
    elif tipo == 'TIPO_1_AUSENTE_BASE':
        return 2
    elif tipo == 'TIPO_3_VALOR' and severidade == 'HIGH':
        return 3
    elif tipo == 'TIPO_3_VALOR' and severidade == 'MEDIUM':
        return 4
    elif tipo == 'TIPO_3_DATA':
        return 5
    elif tipo == 'TIPO_3_VALOR' and severidade == 'LOW':
        return 6
    else:
        return 99

# Aplicar prioridade
df_divergences = df_divergences.withColumn(
    "priority",
    F.udf(calculate_priority, IntegerType())(
        F.struct("tipo_divergencia", "severidade")
    )
).orderBy("priority", "fornecedor_id", "ordem_id")
```

---

### 4. Adicionar Campos de Metadados

**Adicionar campos adicionais para análise:**

```python
# Adicionar campos calculados
df_divergences = df_divergences.withColumn(
    "diferenca_valor",
    F.when(
        F.col("tipo_divergencia").contains("VALOR"),
        F.abs(F.col("valor_csv") - F.col("valor_base"))
    ).otherwise(F.lit(None))
).withColumn(
    "diferenca_valor_abs",
    F.abs(F.col("diferenca_valor"))
).withColumn(
    "percentual_divergencia",
    F.when(
        F.col("valor_base") > 0,
        (F.col("diferenca_valor_abs") / F.col("valor_base")) * 100
    ).otherwise(F.lit(None))
)
```

---

### 5. Validar Classificação

**Criar testes unitários:**

```python
def test_classify_tipo1():
    """Teste TIPO 1"""
    row = type('obj', (object,), {
        'ordem_id_csv': 'ORD-001',
        'ordem_id_base': None,
        'valor_csv': 100.00,
        'valor_base': None,
        'data_csv': '2026-02-16',
        'data_base': None
    })()
    
    result = classify_divergence_type(row)
    assert result['tipo_divergencia'] == 'TIPO_1_AUSENTE_BASE'
    assert result['severidade'] == 'HIGH'

def test_classify_tipo3_valor_high():
    """Teste TIPO 3 VALOR com severidade HIGH"""
    row = type('obj', (object,), {
        'ordem_id_csv': 'ORD-001',
        'ordem_id_base': 'ORD-001',
        'valor_csv': 200.00,
        'valor_base': 50.00,
        'data_csv': '2026-02-16',
        'data_base': '2026-02-16'
    })()
    
    result = classify_divergence_type(row)
    assert result['tipo_divergencia'] == 'TIPO_3_VALOR'
    assert result['severidade'] == 'HIGH'
```

---

## Validação

### Checklist de Validação:

- [ ] Regras de classificação implementadas corretamente
- [ ] Severidade calculada conforme regras
- [ ] Priorização implementada
- [ ] Testes unitários criados e passando
- [ ] Validar com dados reais
- [ ] Verificar que todas as divergências são classificadas
- [ ] Verificar ordenação por prioridade
- [ ] Validar detalhes das divergências

### Cenários de Teste:

1. **TIPO 1:** CSV tem ordem_id que não existe na base
2. **TIPO 2:** Base tem ordem_id que não existe no CSV
3. **TIPO 3 VALOR HIGH:** Diferença de R$ 150,00
4. **TIPO 3 VALOR MEDIUM:** Diferença de R$ 50,00
5. **TIPO 3 VALOR LOW:** Diferença de R$ 0,01
6. **TIPO 3 DATA:** Datas diferentes

---

## Notas Técnicas

- **Tolerância:** Conforme PRD, diferença absoluta = 0 (sem tolerância), mas ainda reportamos diferenças ≤ R$ 0,01 como LOW
- **Priorização:** Ordenar resultados por prioridade para facilitar análise pela área de negócio
- **Detalhes:** Incluir informações suficientes para investigação

---

## Próximas Tasks

- **TASK 09:** Implementar Glue Job de Preparação QuickSight (usará divergências classificadas)

---

## Referências

- PRD - Seção 3.2 (Tipos de Divergência)
- PRD - Seção 5.2.2 (Prioritização de Divergências)
- Documento Técnico - Seção 2.2 (Tolerância Numérica)
