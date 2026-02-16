# TASK 14: Testes End-to-End

**Sprint:** 5-6 (Saída, QuickSight e Notificações)  
**Prioridade:** CRÍTICA  
**Estimativa:** 3-4 dias  
**Dependências:** Todas as tasks anteriores

---

## Objetivo

Executar testes end-to-end completos do sistema de conciliação, validando fluxo completo desde upload do CSV até visualização no QuickSight.

---

## Requisitos

- Todas as tasks anteriores concluídas
- Ambiente de teste configurado
- Dataset de teste preparado

---

## Passo a Passo Detalhado

### 1. Preparar Dataset de Teste

**Passos:**

1.1. Criar script para gerar CSV de teste (`generate_test_data.py`):
```python
import csv
import random
from datetime import datetime, timedelta

def generate_test_csv(num_lines, output_file, include_errors=False):
    """
    Gera CSV de teste com dados sintéticos
    """
    fornecedores = [f"FORN-{i:05d}" for i in range(1, 11)]
    ativos = [f"ATV-{i:03d}" for i in range(1, 21)]
    
    with open(output_file, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f, delimiter=';')
        
        # Cabeçalho
        writer.writerow(['fornecedor_id', 'ordem_id', 'nome', 'ativo_id', 'data', 'valor'])
        
        for i in range(num_lines):
            fornecedor = random.choice(fornecedores)
            ordem_id = f"ORD-{random.randint(10000, 99999)}"
            nome = f"Nome {random.randint(1, 1000)}"
            ativo = random.choice(ativos)
            data = (datetime.now() - timedelta(days=random.randint(0, 30))).strftime('%Y-%m-%d')
            valor = round(random.uniform(10.0, 10000.0), 2)
            
            # Incluir erros se solicitado
            if include_errors and i % 100 == 0:
                # Linha com campo vazio
                valor = ""
            elif include_errors and i % 200 == 0:
                # Linha com formato inválido
                ordem_id = "INVALID"
            
            writer.writerow([fornecedor, ordem_id, nome, ativo, data, valor])

# Gerar datasets de teste
generate_test_csv(1000, 'test_data_1k.csv', include_errors=True)
generate_test_csv(10000, 'test_data_10k.csv')
generate_test_csv(100000, 'test_data_100k.csv')
```

1.2. Upload para S3:
```bash
aws s3 cp test_data_1k.csv s3://reconciliation-ingress/2026-02-16/test_1k.csv
aws s3 cp test_data_10k.csv s3://reconciliation-ingress/2026-02-16/test_10k.csv
aws s3 cp test_data_100k.csv s3://reconciliation-ingress/2026-02-16/test_100k.csv
```

---

### 2. Cenários de Teste

**2.1. Teste 1: Fluxo Completo com Dados Válidos**

**Objetivo:** Validar que sistema processa CSV válido end-to-end

**Passos:**
1. Upload CSV válido (1.000 linhas)
2. Verificar Lambda trigger executado
3. Verificar Step Functions iniciada
4. Verificar Glue Jobs executados com sucesso
5. Verificar Parquet gerado no S3 output
6. Verificar QuickSight atualizado
7. Verificar notificações enviadas
8. Validar dados no QuickSight

**Critérios de Sucesso:**
- ✅ Todos os componentes executam sem erros
- ✅ Dados aparecem corretamente no QuickSight
- ✅ Notificações são recebidas
- ✅ Tempo total < 20 minutos

---

**2.2. Teste 2: Validação de Erros**

**Objetivo:** Validar que sistema identifica e reporta erros corretamente

**Passos:**
1. Upload CSV com erros (campos vazios, formato inválido)
2. Verificar que validação identifica erros
3. Verificar que linhas inválidas são reportadas
4. Verificar que processamento continua com linhas válidas
5. Verificar relatório de erros gerado

**Critérios de Sucesso:**
- ✅ Erros são identificados corretamente
- ✅ Linhas inválidas são reportadas
- ✅ Processamento não falha completamente

---

**2.3. Teste 3: Divergências Tipo 1, 2 e 3**

**Objetivo:** Validar que todos os tipos de divergência são identificados

**Passos:**
1. Preparar CSV com divergências conhecidas:
   - Linhas presentes apenas no CSV (Tipo 1)
   - Linhas presentes apenas na base (Tipo 2)
   - Linhas com valores diferentes (Tipo 3 VALOR)
   - Linhas com datas diferentes (Tipo 3 DATA)
2. Processar CSV
3. Verificar que todas as divergências são identificadas
4. Verificar classificação correta
5. Verificar severidade correta

**Critérios de Sucesso:**
- ✅ Todas as divergências são identificadas
- ✅ Classificação está correta
- ✅ Severidade está correta

---

**2.4. Teste 4: Idempotência**

**Objetivo:** Validar que reprocessamento não cria duplicatas

**Passos:**
1. Processar CSV válido
2. Reprocessar mesmo arquivo (mesmo hash)
3. Verificar que segunda execução é ignorada ou não cria duplicatas
4. Verificar hash de arquivo funcionando

**Critérios de Sucesso:**
- ✅ Arquivo duplicado não é reprocessado ou não cria duplicatas
- ✅ Hash de arquivo funciona corretamente

---

**2.5. Teste 5: Performance (1M linhas)**

**Objetivo:** Validar SLA de 20 minutos

**Passos:**
1. Gerar CSV com 1.000.000 linhas
2. Upload para S3
3. Iniciar processamento
4. Medir tempo de cada etapa:
   - Validação
   - Conciliação
   - Preparação QuickSight
   - Total
5. Executar 5 vezes e calcular P95

**Critérios de Sucesso:**
- ✅ Tempo total ≤ 20 minutos (P95)
- ✅ Tempo de validação ≤ 3 minutos
- ✅ Tempo de conciliação ≤ 15 minutos
- ✅ Tempo de preparação QuickSight ≤ 2 minutos

---

**2.6. Teste 6: Notificações**

**Objetivo:** Validar que notificações são enviadas corretamente

**Passos:**
1. Processar CSV com sucesso
2. Verificar notificação de sucesso (email + Slack)
3. Verificar conteúdo da notificação
4. Processar CSV com erro
5. Verificar notificação de falha (email + Slack)
6. Verificar conteúdo da notificação de falha

**Critérios de Sucesso:**
- ✅ Notificações de sucesso são enviadas
- ✅ Notificações de falha são enviadas
- ✅ Conteúdo das notificações está correto
- ✅ Links para dashboard funcionam

---

### 3. Checklist de Validação Completo

**Infraestrutura:**
- [ ] S3 buckets criados e configurados
- [ ] IAM roles criadas e configuradas
- [ ] Lambda functions criadas e funcionando
- [ ] Step Functions criada e funcionando
- [ ] Glue Jobs criados e funcionando
- [ ] QuickSight configurado
- [ ] SNS configurado

**Funcionalidade:**
- [ ] Validação de CSV funcionando
- [ ] Conciliação funcionando
- [ ] Classificação de divergências funcionando
- [ ] Geração de Parquet funcionando
- [ ] QuickSight atualizando corretamente
- [ ] Notificações sendo enviadas

**Performance:**
- [ ] Processamento de 1M linhas em ≤ 20 minutos
- [ ] Métricas sendo publicadas
- [ ] Logs sendo gerados

**Qualidade:**
- [ ] Todas as divergências são identificadas
- [ ] Classificação está correta
- [ ] Dados no QuickSight estão corretos
- [ ] Idempotência funcionando

---

### 4. Documentar Resultados

**Criar relatório de testes:**

```markdown
# Relatório de Testes End-to-End

## Data: 2026-02-16
## Ambiente: Dev/Staging

## Resumo Executivo
- Total de testes: 6
- Testes passaram: X
- Testes falharam: Y

## Detalhes dos Testes

### Teste 1: Fluxo Completo
- Status: ✅ PASSOU
- Tempo: X minutos
- Observações: ...

### Teste 2: Validação de Erros
- Status: ✅ PASSOU
- Observações: ...

...

## Métricas de Performance
- Tempo médio de processamento: X minutos
- P95: X minutos
- P99: X minutos
- Throughput: X linhas/segundo

## Problemas Identificados
1. Problema 1: Descrição
   - Impacto: Alto/Médio/Baixo
   - Solução: ...

## Próximos Passos
1. Corrigir problemas identificados
2. Executar testes novamente
3. Validar com stakeholders
```

---

## Validação

### Checklist Final:

- [ ] Todos os testes executados
- [ ] Todos os testes passaram
- [ ] Performance validada (≤ 20 minutos)
- [ ] Relatório de testes criado
- [ ] Problemas documentados
- [ ] Sistema pronto para produção (ou próximas melhorias identificadas)

---

## Notas Técnicas

- **Ambiente:** Executar testes em ambiente isolado (dev/staging)
- **Dados:** Usar dados sintéticos para testes
- **Monitoramento:** Monitorar métricas durante testes
- **Documentação:** Documentar todos os resultados

---

## Próximas Ações

Após conclusão dos testes:
1. Corrigir problemas identificados
2. Executar testes de regressão
3. Obter aprovação para produção
4. Planejar deploy em produção

---

## Referências

- Documento Técnico - Seção 7 (Plano de Testes de Carga)
- PRD - Seção 12 (Testes e Validação)
- PRD - Seção 10.2 (Teste de Performance)
