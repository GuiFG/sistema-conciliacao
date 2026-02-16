# Documento Técnico - Sistema de Conciliação Financeira

**Versão:** 2.0  
**Data:** 16 de Fevereiro de 2026  
**Status:** Pronto para Implementação  
**Autor:** Agente Técnico

---

## 1. Visão Geral Executiva

### 1.1 Contexto do Problema

A empresa recebe diariamente (D-1) um arquivo CSV do fornecedor contendo aproximadamente **1.000.000 de linhas** com transações financeiras. Os campos são: `fornecedor_id`, `ordem_id`, `nome`, `ativo_id`, `data`, `valor`. A empresa possui uma base interna com os mesmos campos e precisa de um sistema de conciliação que:

- Processe 1M linhas em **≤ 20 minutos** (SLA crítico)
- Identifique 3 tipos de divergências
- Notifique falhas em tempo real
- Entregue resultados estruturados para a área de negócio
- Execute na **AWS**

### 1.2 Objetivo do Documento

Este documento apresenta a arquitetura técnica completa da solução de conciliação financeira, incluindo:
- **Decisões técnicas definidas** e justificativas
- **Arquitetura da solução** (AWS Glue + RDS + QuickSight)
- **Plano de implementação** por fases
- **Plano de testes de carga** para validar SLA de 20 minutos
- **Runbook e playbooks** de falha
- **Especificações técnicas** de saída e visualização

---

## 2. Decisões Técnicas Definidas

**STATUS:** ✅ Todas as decisões críticas foram definidas pela área de negócio e stakeholders. As decisões estão documentadas abaixo com justificativas técnicas.

---

### 2.1 Chave de Correspondência (Matching Key)

**✅ DECISÃO DEFINIDA:** Usar apenas `ordem_id` como chave única de correspondência.

**Justificativa da Decisão:**
- `ordem_id` é único globalmente (confirmado pela área de negócio)
- Simplifica lógica de matching
- Performance excelente com índice simples no RDS
- Alinhado com PRD que define `ordem_id` como chave única

**Implementação Técnica:**
- Criar índice único em `ordem_id` na tabela do RDS (se não existir)
- Usar `ordem_id` para join entre CSV e base interna no Glue Job
- Hash de linha completo será calculado para auditoria (não para matching)

**Impacto na Performance:**
- ✅ Lookup mais rápido (índice simples vs. composto)
- ✅ Menos I/O no RDS
- ✅ Queries mais eficientes no Spark

**⚠️ VALIDAÇÃO NECESSÁRIA:** Confirmar que não há `ordem_id` duplicados na base interna (constraint de unicidade).

---

### 2.2 Tolerância em Valores Numéricos

**✅ DECISÃO DEFINIDA:** Diferença absoluta = 0 (exata, sem tolerância).

**Justificativa da Decisão:**
- Regra simples e clara
- Zero tolerância a erros (requisito de negócio)
- Facilita implementação (comparação direta)

**Implementação Técnica:**
- Comparação exata: `valor_csv == valor_base`
- Valores devem ser normalizados para 2 casas decimais antes da comparação
- Qualquer diferença, mesmo de R$ 0,01, será marcada como divergência Tipo 3

**Impacto Operacional:**
- ⚠️ Pode gerar mais divergências por arredondamentos
- ⚠️ Necessário garantir que ambas as fontes usam mesmo método de arredondamento
- ✅ Regra clara e objetiva para área de negócio

**Recomendação Técnica:**
- Implementar normalização de valores (round para 2 casas decimais) antes da comparação
- Documentar método de arredondamento usado (banker's rounding recomendado)
- Monitorar taxa de divergências Tipo 3 para identificar problemas sistemáticos de arredondamento

---

### 2.3 Estratégia de Processamento

**✅ DECISÃO DEFINIDA:** AWS Glue (Serverless Batch com Spark).

**Justificativa da Decisão:**
- Baixo overhead operacional (serverless)
- Escalabilidade automática
- Integração nativa com S3 e outros serviços AWS
- Suporte a Spark para processamento distribuído
- Paga apenas pelo uso (DPU)

**Arquitetura Escolhida:**
- **Componente Principal:** AWS Glue Jobs (Spark)
- **Orquestração:** Step Functions
- **Armazenamento:** S3 (ingress, intermediário, output)
- **Base de Dados:** RDS (leitura via JDBC no Glue)

**Configuração Recomendada:**
- **DPU:** 10-15 DPU (cada DPU = 4 vCPU, 16GB RAM)
- **Workers:** 10-15 executors Spark
- **Timeout:** 20 minutos (margem para SLA de 20 min)

**Custo Estimado:** $150-300/mês (baseado em 20min/dia de execução)

---

### 2.4 Armazenamento e Consulta dos Dados de Referência (Base Interna)

**✅ DECISÃO DEFINIDA:** RDS (PostgreSQL/MySQL) - Base interna já existente.

**Justificativa da Decisão:**
- Base interna já está em RDS (infraestrutura existente)
- Evita migração de dados
- ACID completo garante consistência
- Suporte a joins complexos se necessário

**Implementação Técnica:**
- **Conexão:** AWS Glue conecta ao RDS via JDBC
- **Estratégia de Leitura:** 
  - Opção 1: Leitura direta do RDS durante conciliação (join distribuído)
  - Opção 2: Exportar snapshot diário para S3 Parquet e ler do S3 (recomendado para performance)
- **Índice Crítico:** Criar índice único em `ordem_id` (se não existir)
- **Otimizações:**
  - Usar connection pooling no Glue
  - Particionar leitura por `fornecedor_id` ou range de `ordem_id`
  - Considerar read replica se latência for problema

**Performance Esperada:**
- ⚠️ RDS pode ser gargalo se não otimizado
- ✅ Exportar snapshot para S3 reduz carga no RDS e melhora performance
- ✅ Índice em `ordem_id` é crítico para performance

**Recomendação Técnica:**
- **Fase MVP:** Leitura direta do RDS (mais simples)
- **Fase Produção:** Implementar export diário para S3 Parquet (melhor performance)
- Monitorar latência de queries e ajustar estratégia conforme necessário

---

### 2.5 Como Entregar Divergências para Negócio

**✅ DECISÃO DEFINIDA:** Amazon QuickSight (Dashboard interativo).

**Justificativa da Decisão:**
- Visualização interativa para análise exploratória
- Filtros e agregações facilitam análise de divergências
- Melhor UX para área de negócio
- Integração nativa com S3 e outros serviços AWS

**Implementação Técnica:**
- **Fonte de Dados:** S3 Parquet com divergências (gerado pelo Glue Job)
- **Glue Data Catalog:** Tabela catalogada para QuickSight
- **Atualização:** Refresh automático após cada execução de conciliação
- **Visualizações:**
  - Tabela de divergências com filtros
  - Gráficos por tipo de divergência
  - Gráficos por fornecedor
  - Tendências históricas
  - Métricas de conciliação (taxa de divergência, tempo de processamento)

**Custo Estimado:**
- **QuickSight:** $5-18/usuário/mês (conforme tipo de licença)
- **S3 Storage:** ~$0.10/mês (dados de divergências)
- **Total:** Depende do número de usuários

**Tempo de Implementação:** 2-3 semanas (incluindo desenvolvimento de dashboards)

**Recomendação Técnica:**
- Gerar dados em formato Parquet otimizado para QuickSight
- Criar datasets no QuickSight com refresh automático
- Desenvolver dashboards interativos com filtros por tipo, severidade, fornecedor, data

---

### 2.6 Estratégia de Retry e Timeout

**✅ DECISÃO DEFINIDA:** Retries até 20 minutos após início do processamento, com checkpoint em caso de falha.

**Regra de Negócio:**
- **Timeout absoluto:** 20 minutos após início do processamento
- **Exemplo:** Se arquivo apareceu às 7h00, processamento deve parar às 7h20
- **Ação em timeout:** Abortar processamento, gerar checkpoint, enviar notificação de falha

**Implementação Técnica:**

1. **Controle de Tempo:**
   - Step Functions inicia timer no início da execução
   - Timer configurado para 20 minutos
   - Se timer expira → Abortar todos os jobs Glue em execução

2. **Retries:**
   - Retries automáticos para falhas transitórias (rede, timeout RDS)
   - Backoff exponencial: 30s, 1min, 2min, 4min
   - Máximo de 3 tentativas por etapa
   - **Importante:** Retries devem respeitar o limite de 20 minutos total

3. **Checkpoint em Falha:**
   - **Quando:** Timeout de 20 minutos OU falha crítica após retries
   - **O que salvar:**
     - Estado do processamento (linhas processadas, linhas pendentes)
     - Localização dos dados intermediários (S3 Parquet)
     - Logs e métricas até o ponto de falha
   - **Onde:** S3 (`s3://reconciliation-checkpoints/{job_id}/`)
   - **Formato:** JSON com estado completo + referências S3

4. **Notificação de Falha:**
   - Enviar imediatamente após timeout/falha
   - Incluir:
     - Job ID
     - Timestamp de início e fim
     - Motivo da falha
     - Localização do checkpoint
     - Link para logs detalhados
   - Canais: Slack + Email (CRITICAL)

**Estrutura do Checkpoint:**
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

**Reprocessamento Manual:**
- Operador pode usar checkpoint para reprocessar apenas linhas pendentes
- Script de reprocessamento lerá checkpoint e continuará de onde parou
- Evita reprocessar linhas já processadas (idempotência)

---

## 3. Arquitetura da Solução

**✅ ARQUITETURA ESCOLHIDA:** Serverless Batch com AWS Glue

### 3.1 Visão Geral

Arquitetura serverless usando AWS Glue para processamento distribuído com Spark, integração direta com RDS para leitura da base interna, e Amazon QuickSight para visualização de divergências. Step Functions orquestra o fluxo completo.

### 3.2 Componentes da Arquitetura

1. **Ingress:**
   - **S3 Bucket** (`s3://reconciliation-ingress/YYYY-MM-DD/`)
   - **EventBridge Rule** detecta upload de CSV
   - **Lambda Trigger** valida formato básico e inicia orquestração

2. **Orquestração:**
   - **Step Functions State Machine** coordena fluxo completo
   - Estados: Validação → Processamento → Conciliação → Geração de Relatórios → Atualização QuickSight → Notificação

3. **Processamento:**
   - **AWS Glue Job 1 - Validação e Parsing** (Spark):
     - Leitura do CSV do S3
     - Validação de linhas (esquema, tipos, duplicatas)
     - Particionamento por `fornecedor_id` e hash de `ordem_id`
     - Escalabilidade automática (10-15 DPU)
     - Saída: S3 Parquet intermediário

4. **Conciliação:**
   - **AWS Glue Job 2 - Conciliação** (Spark):
     - Leitura do Parquet intermediário (CSV validado)
     - **Leitura da base interna via JDBC** (RDS PostgreSQL/MySQL)
     - Join distribuído usando **`ordem_id`** como chave única
     - Classificação de divergências:
       - **Tipo 1:** Presente no CSV, ausente no RDS
       - **Tipo 2:** Presente no RDS, ausente no CSV
       - **Tipo 3:** Presente em ambos, mas com campos divergentes
     - **Comparação exata de valores** (diferença absoluta = 0)
     - Cálculo de hash de linha para auditoria
     - Saída: S3 Parquet com divergências

5. **Armazenamento de Referência:**
   - **RDS** (PostgreSQL/MySQL) - Base interna existente
   - Conexão via JDBC no Glue Job
   - **Índice crítico:** Índice único em `ordem_id` (criar se não existir)
   - **Otimização:** Considerar read replica se latência for problema

6. **Saída e Visualização:**
   - **S3 Bucket** (`s3://reconciliation-output/YYYY-MM-DD/`)
   - **Parquet otimizado** para QuickSight (com todas as divergências)
   - **Glue Data Catalog:** Tabela catalogada para QuickSight
   - **Amazon QuickSight:**
     - Datasets com refresh automático após cada execução
     - Dashboards interativos com filtros (tipo, severidade, fornecedor, data)
     - Visualizações: tabelas, gráficos, métricas

7. **Notificação:**
   - **SNS Topic** → Slack (webhook) + Email (SES)
   - Notificações de sucesso e falha
   - Inclui link para dashboard QuickSight

8. **Observabilidade:**
   - **CloudWatch Logs** (logs estruturados JSON)
   - **CloudWatch Metrics** (custom metrics)
   - **X-Ray** (tracing distribuído)

### 3.3 Fluxo de Dados

```
CSV Upload → S3 → EventBridge → Lambda (validação básica)
    ↓
Step Functions (orquestração)
    ↓
Glue Job 1: Validação e Parsing
    ↓ (S3 Parquet intermediário)
Glue Job 2: Conciliação
    ├─ Leitura Parquet (CSV validado)
    ├─ Leitura RDS via JDBC (base interna)
    ├─ Join por ordem_id
    ├─ Comparação exata (diferença = 0)
    └─ Classificação de divergências (Tipo 1, 2, 3)
    ↓ (S3 Parquet com divergências)
Glue Job 3: Preparação para QuickSight
    ├─ Otimização de schema
    ├─ Registro no Glue Data Catalog
    └─ Trigger de refresh QuickSight
    ↓
QuickSight Dashboard (atualizado automaticamente)
    ↓
SNS Notificação (sucesso/falha + link dashboard)
```

### 3.4 Paralelismo e Particionamento

- **Particionamento:** Por `fornecedor_id` (primeiro nível) e hash de `ordem_id` (segundo nível)
- **Workers:** 10-15 executors Spark (configurável via DPU)
- **Throughput estimado:** 50k-100k linhas/minuto por executor
- **Chave de Join:** `ordem_id` (índice único no RDS)
- **Estratégia de Leitura RDS:**
  - Particionar leitura por range de `ordem_id` ou `fornecedor_id`
  - Usar connection pooling para otimizar conexões
  - Considerar batch reads para reduzir latência

### 3.5 Pontos de Escalonamento

- **Auto-scaling:** AWS Glue escala automaticamente baseado em DPU configurado
- **Configuração Recomendada:** 10-15 DPU (suficiente para 1M linhas em ≤ 20 min)
- **Limites:** Máximo 100 DPU por job (headroom para crescimento)
- **Cold start:** 30s-2min (primeira execução do dia)
- **RDS Connection Pooling:** Configurar pool adequado para suportar múltiplos workers Spark

### 3.6 Custos Estimados

| Componente | Custo Mensal Estimado |
|------------|----------------------|
| AWS Glue (10-15 DPU, 20min/dia) | $200-350 |
| S3 Storage (1GB ingress + 2GB output + Parquet) | $0.20 |
| S3 Requests | $0.50 |
| Step Functions | $5 |
| Lambda (invocações) | $1 |
| SNS | $1 |
| CloudWatch Logs | $10 |
| QuickSight (5 usuários Standard) | $25-90 |
| RDS (já existente, custo não incluído) | $0 |
| **Total** | **$242-458/mês** |

**Nota:** Custo do RDS não está incluído pois é infraestrutura existente. QuickSight custo varia conforme tipo de licença (Standard vs Enterprise).

### 3.7 Prós e Contras

**Prós:**
- ✅ Baixo overhead operacional (serverless)
- ✅ Escalabilidade automática
- ✅ Paga apenas pelo uso
- ✅ Integração nativa com serviços AWS (S3, RDS, QuickSight)
- ✅ Suporte a Spark (processamento distribuído)
- ✅ Visualização interativa no QuickSight
- ✅ Aproveita infraestrutura RDS existente

**Contras:**
- ❌ Cold start pode impactar latência inicial (30s-2min)
- ❌ RDS pode ser gargalo se não otimizado (índice em `ordem_id` crítico)
- ❌ Debugging mais complexo (logs distribuídos)
- ❌ Limites de Glue (100 DPU, 48h execução)
- ❌ Custo adicional do QuickSight (licenças)

### 3.8 Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| **RDS como gargalo** | Média | Alto | Criar índice único em `ordem_id`, usar connection pooling, considerar read replica |
| **Cold start > 2min** | Média | Médio | Pre-warm com job dummy ou aceitar cold start |
| **Timeout de Glue** | Baixa | Alto | Configurar timeout adequado (30min), implementar checkpoint |
| **Latência de leitura RDS** | Média | Alto | Particionar leitura, usar batch reads, monitorar métricas |
| **QuickSight refresh lento** | Baixa | Médio | Otimizar schema Parquet, usar partições, refresh assíncrono |

---

## 4. Integração com Amazon QuickSight

### 4.1 Configuração do QuickSight

**Fonte de Dados:**
- **Tipo:** S3 (Parquet)
- **Localização:** `s3://reconciliation-output/YYYY-MM-DD/divergencias.parquet`
- **Glue Data Catalog:** Tabela `reconciliation.divergences` catalogada automaticamente

**Atualização Automática:**
- **Refresh Schedule:** Após cada execução de conciliação (via EventBridge trigger)
- **Incremental:** Sim (apenas novos dados do dia)
- **Timeout:** 5 minutos

### 4.2 Estrutura de Dados para QuickSight

**Schema Parquet:**
```python
{
  "fornecedor_id": "string",
  "ordem_id": "string",
  "nome": "string",
  "ativo_id": "string",
  "data": "date",
  "valor_csv": "decimal(10,2)",
  "valor_base": "decimal(10,2)",
  "tipo_divergencia": "string",  # TIPO_1, TIPO_2, TIPO_3_VALOR, TIPO_3_DATA
  "detalhes": "string",
  "severidade": "string",  # HIGH, MEDIUM, LOW
  "hash_linha": "string",
  "timestamp": "timestamp",
  "job_id": "string"
}
```

### 4.3 Dashboard: Visão Geral de Divergências

**Único dashboard com as seguintes visualizações:**

1. **Métricas Principais (KPIs):**
   - Total de divergências (número absoluto)
   - Taxa de divergência (% do total de transações)
   - Total por tipo de divergência (Tipo 1, Tipo 2, Tipo 3)

2. **Gráficos:**
   - Distribuição de divergências por tipo (gráfico de pizza)
   - Distribuição por severidade (HIGH, MEDIUM, LOW) - gráfico de barras
   - Top 10 fornecedores com mais divergências (gráfico de barras horizontais)
   - Tendência histórica (últimos 30 dias) - gráfico de linha

3. **Tabela de Divergências:**
   - Lista completa de divergências com filtros:
     - Tipo de divergência
     - Severidade
     - Fornecedor
     - Data
   - Colunas: fornecedor_id, ordem_id, tipo_divergencia, detalhes, severidade, data
   - Paginação: 100 registros por página
   - Export para CSV disponível

4. **Filtros Globais:**
   - Período (data inicial e final)
   - Fornecedor
   - Tipo de divergência
   - Severidade

### 4.4 Permissões e Acesso

**IAM Roles:**
- QuickSight precisa de permissão de leitura no S3 bucket `reconciliation-output`
- Usuários QuickSight com acesso apenas aos datasets de conciliação

**Segurança:**
- Row-level security (RLS) se necessário (ex.: usuário vê apenas fornecedores específicos)
- Mascaramento de dados sensíveis (`nome` parcialmente mascarado)

---

## 5. Cálculos de Performance e Throughput

### 5.1 Throughput Necessário

**Requisito:** Processar 1.000.000 linhas em ≤ 20 minutos

**Cálculo:**
\[
\text{Throughput mínimo} = \frac{1.000.000 \text{ linhas}}{20 \text{ minutos}} = 50.000 \text{ linhas/minuto} \approx 833 \text{ linhas/segundo}
\]

**Meta com margem de segurança (80% eficiência):**
\[
\text{Throughput necessário} = \frac{50.000}{0.8} = 62.500 \text{ linhas/minuto} \approx 1.042 \text{ linhas/segundo}
\]

### 5.2 Breakdown por Etapa (AWS Glue)

**Estimativas baseadas em AWS Glue com Spark:**

| Etapa | Tempo Estimado | Throughput Esperado | DPU Necessário |
|-------|---------------|-------------------|----------------|
| **Ingestão e Validação** | ≤ 3 min | 333k linhas/min | 5-8 DPU |
| **Conciliação (join com RDS)** | ≤ 15 min | 67k linhas/min | 10-15 DPU |
| **Geração Parquet para QuickSight** | ≤ 2 min | 500k linhas/min | 5-8 DPU |
| **Total** | ≤ 20 min | 50k linhas/min (média) | 10-15 DPU (máx) |

**Notas:**
- Conciliação é etapa mais lenta devido a join com RDS via JDBC
- Leitura do RDS pode ser gargalo se não otimizado (índice em `ordem_id` crítico)
- Cold start do Glue: 30s-2min (não contabilizado no tempo de processamento)

### 5.3 Dimensionamento AWS Glue

**Configuração Recomendada:**

**Glue Job 1 - Validação:**
- **DPU:** 5-8 DPU
- **Workers:** 5-8 executors Spark
- **Throughput esperado:** ~100k linhas/minuto por executor
- **Tempo estimado:** 2-3 minutos para 1M linhas

**Glue Job 2 - Conciliação:**
- **DPU:** 10-15 DPU (crítico para performance)
- **Workers:** 10-15 executors Spark
- **Throughput esperado:** ~7k linhas/minuto por executor (limitado por RDS)
- **Tempo estimado:** 12-15 minutos para 1M linhas
- **Otimizações:**
  - Particionar leitura do RDS por range de `ordem_id`
  - Usar connection pooling (máx 10 conexões por executor)
  - Batch reads do RDS (1000-5000 linhas por batch)

**Glue Job 3 - Preparação QuickSight:**
- **DPU:** 5-8 DPU
- **Workers:** 5-8 executors Spark
- **Throughput esperado:** ~200k linhas/minuto por executor
- **Tempo estimado:** 1-2 minutos

**Total de DPU:** 10-15 DPU (máximo simultâneo, jobs executam sequencialmente)

### 5.4 Estimativa de Performance Realista

**Cenário Otimista (RDS bem otimizado, índice em `ordem_id`):**
- Tempo total: 15-18 minutos
- Margem: Confortável para SLA de 20 minutos

**Cenário Realista (RDS com latência média):**
- Tempo total: 18-20 minutos
- Margem: Apertada, mas dentro do SLA

**Cenário Pessimista (RDS lento, sem índice):**
- Tempo total: > 20 minutos
- **Ação:** Otimizar RDS (criar índice) ou considerar export para S3

**Fatores que Impactam Performance:**
1. **Latência do RDS:** Principal gargalo (depende de índice, conexões, carga)
2. **Cold start do Glue:** 30s-2min (primeira execução do dia)
3. **Particionamento:** Dados bem particionados melhoram paralelismo
4. **Tamanho do CSV:** Linhas maiores aumentam I/O

---

## 6. Plano de Implementação por Fases

### 6.1 Fase 1: MVP (4-6 semanas)

**Objetivo:** Sistema funcional que processa 1M linhas em ≤ 20 minutos com funcionalidades básicas.

#### Sprint 1-2: Infraestrutura Base (2 semanas)
- [ ] Configurar S3 buckets (ingress, output, intermediário)
- [ ] Configurar IAM roles e políticas
- [ ] Implementar Lambda de trigger (validação básica)
- [ ] Configurar Step Functions (orquestração básica)
- [ ] Setup de CloudWatch Logs e métricas básicas

#### Sprint 3-4: Processamento Core (2 semanas)
- [ ] Implementar Glue Job de validação e parsing
- [ ] Implementar lógica de conciliação (join com base interna)
- [ ] Implementar classificação de divergências (Tipo 1, 2, 3)
- [ ] Testes unitários e de integração

#### Sprint 5-6: Saída, QuickSight e Notificações (2 semanas)
- [ ] Implementar geração de Parquet otimizado para QuickSight
- [ ] Configurar Glue Data Catalog
- [ ] Configurar QuickSight dataset e dashboard (visão geral)
- [ ] Configurar SNS e notificações (Slack + Email)
- [ ] Implementar idempotência (hash de linha)
- [ ] Testes end-to-end

**Entregáveis MVP:**
- ✅ Sistema processa CSV e gera Parquet de divergências
- ✅ Dashboard QuickSight funcional
- ✅ Notificações básicas funcionando
- ✅ Logs e métricas básicas

---

### 6.2 Fase 2: Produção (2-3 semanas)

**Objetivo:** Hardening, observabilidade completa, segurança, checkpoint e otimizações.

#### Sprint 7-8: Observabilidade, Segurança e Checkpoint (2 semanas)
- [ ] Implementar métricas customizadas (CloudWatch)
- [ ] Configurar alertas (SNS → Slack)
- [ ] Implementar tracing (X-Ray)
- [ ] Configurar criptografia (KMS)
- [ ] Implementar mascaramento de dados sensíveis
- [ ] Implementar checkpoint em caso de timeout/falha
- [ ] Implementar timeout de 20 minutos com abort automático
- [ ] Auditoria e compliance (retenção de logs)

#### Sprint 9: Otimizações e Runbook (1 semana)
- [ ] Otimizar particionamento e queries
- [ ] Otimizar leitura do RDS (connection pooling, batch reads)
- [ ] Validar índice em `ordem_id` no RDS
- [ ] Criar runbook detalhado
- [ ] Testes de carga (1M linhas)
- [ ] Documentação técnica completa

**Entregáveis Produção:**
- ✅ Sistema otimizado e testado
- ✅ Observabilidade completa
- ✅ Checkpoint implementado
- ✅ Runbook e documentação
- ✅ Aprovação para deploy

---

## 7. Plano de Testes de Carga

### 7.1 Objetivo

Validar que o sistema processa 1.000.000 de linhas em ≤ 20 minutos com 95% de confiança (P95 ≤ 20min).

### 7.2 Cenários de Teste

#### Cenário 1: Volume Normal (1M linhas)
- **Dataset:** 1.000.000 linhas válidas
- **Distribuição:** 10 fornecedores, datas variadas
- **Divergências:** 5% Tipo 1, 3% Tipo 2, 2% Tipo 3
- **Execuções:** 5 execuções
- **Critério:** P95 ≤ 20 minutos

#### Cenário 2: Volume de Pico (1.5M linhas)
- **Dataset:** 1.500.000 linhas válidas
- **Objetivo:** Validar escalabilidade
- **Critério:** Processamento completo sem falhas

#### Cenário 3: Alta Taxa de Divergências (1M linhas, 20% divergências)
- **Dataset:** 1.000.000 linhas com 20% de divergências
- **Objetivo:** Validar performance com muitos resultados
- **Critério:** Tempo ≤ 25 minutos (tolerância para mais processamento)

### 7.3 Geração de Dados Sintéticos

**Script Python (exemplo):**
```python
import csv
import random
from datetime import datetime, timedelta

def generate_synthetic_data(num_lines, output_file):
    fornecedores = [f"FORN-{i:05d}" for i in range(1, 11)]
    ativos = [f"ATV-{i:03d}" for i in range(1, 21)]
    
    with open(output_file, 'w', newline='') as f:
        writer = csv.writer(f, delimiter=';')
        writer.writerow(['fornecedor_id', 'ordem_id', 'nome', 'ativo_id', 'data', 'valor'])
        
        for i in range(num_lines):
            fornecedor = random.choice(fornecedores)
            ordem_id = f"ORD-{random.randint(10000, 99999)}"
            nome = f"Nome {random.randint(1, 1000)}"
            ativo = random.choice(ativos)
            data = (datetime.now() - timedelta(days=random.randint(0, 30))).strftime('%Y-%m-%d')
            valor = round(random.uniform(10.0, 10000.0), 2)
            
            writer.writerow([fornecedor, ordem_id, nome, ativo, data, valor])

generate_synthetic_data(1000000, 'test_data_1M.csv')
```

### 7.4 Métricas a Coletar

| Métrica | Como Medir | Meta |
|---------|-----------|------|
| **Tempo Total E2E** | CloudWatch Logs (timestamp início/fim) | ≤ 20 min (P95) |
| **Tempo de Ingestão** | Glue Job metrics | ≤ 3 min |
| **Tempo de Conciliação** | Glue Job metrics | ≤ 15 min |
| **Tempo de Geração de Relatórios** | Glue Job metrics | ≤ 2 min |
| **Throughput (linhas/seg)** | Linhas processadas / tempo | ≥ 833 linhas/seg |
| **CPU Utilização** | CloudWatch Metrics | < 80% (média) |
| **Memória Utilizada** | CloudWatch Metrics | < 80% (média) |
| **I/O S3** | CloudWatch Metrics | Monitorar latência |
| **Taxa de Erro** | CloudWatch Metrics | 0% |

### 7.5 Critérios de Aceitação

✅ **Sucesso:**
- P95 do tempo total ≤ 20 minutos
- Taxa de erro = 0%
- Todas as divergências detectadas corretamente
- Relatório gerado corretamente

❌ **Falha:**
- P95 > 20 minutos → Otimizar particionamento/queries
- Taxa de erro > 0% → Investigar e corrigir
- Divergências não detectadas → Revisar lógica

### 7.6 Ambiente de Teste

**Requisitos:**
- Ambiente isolado (dev/staging)
- Dataset de 1M linhas realistas
- Base interna de teste (snapshot ou mock)
- Monitoramento ativo (CloudWatch)

**Execução:**
1. Preparar dataset sintético
2. Upload para S3 (ingress)
3. Executar pipeline completo
4. Coletar métricas (CloudWatch)
5. Validar resultados
6. Repetir 5 vezes
7. Calcular estatísticas (média, P50, P95, P99)

---

## 8. Runbook e Playbooks de Falha

**Nota:** Os playbooks detalhados de falha foram movidos para um documento separado: `Runbook_Playbooks_Falha.md`

Este documento contém procedimentos operacionais completos para:
- Monitoramento diário
- Falha na ingestão
- Timeout de 20 minutos
- Falha na conciliação
- Falha na entrega de resultados
- Taxa de divergências anormal

Consulte o arquivo `Runbook_Playbooks_Falha.md` para procedimentos detalhados, comandos específicos e templates de notificação.

---

## 9. Especificação de APIs e Artefatos de Saída

### 9.1 Formato de Saída: Parquet para QuickSight

**Localização:** `s3://reconciliation-output/YYYY-MM-DD/divergencias.parquet`

**Esquema Parquet (otimizado para QuickSight):**
```python
{
  "fornecedor_id": "string",
  "ordem_id": "string",  # Chave única de correspondência
  "nome": "string",
  "ativo_id": "string",
  "data": "date",
  "valor_csv": "decimal(10,2)",
  "valor_base": "decimal(10,2)",
  "tipo_divergencia": "string",  # TIPO_1, TIPO_2, TIPO_3_VALOR, TIPO_3_DATA
  "detalhes": "string",
  "severidade": "string",  # HIGH, MEDIUM, LOW
  "hash_linha": "string",
  "timestamp": "timestamp",
  "job_id": "string"
}
```

**Particionamento:**
- Por `data` (partição principal)
- Por `fornecedor_id` (partição secundária, opcional)

**Otimizações para QuickSight:**
- Formato Parquet com compressão Snappy
- Schema otimizado (tipos corretos, nullable apropriado)
- Particionamento para queries eficientes
- Glue Data Catalog atualizado automaticamente

---

---

## 10. Observabilidade e Segurança

### 10.1 Logs Estruturados

**Formato:** JSON  
**Destino:** CloudWatch Logs + S3 (retenção)

**Exemplo de Log:**
```json
{
  "timestamp": "2026-02-16T08:00:00Z",
  "level": "INFO",
  "job_id": "JOB-20260216-001234",
  "event": "ingestion_completed",
  "metrics": {
    "total_lines": 1000000,
    "valid_lines": 999500,
    "invalid_lines": 500,
    "processing_time_seconds": 180
  },
  "file_hash": "a1b2c3d4e5f6...",
  "s3_key": "s3://reconciliation-ingress/2026-02-16/input.csv"
}
```

---

### 10.2 Métricas Customizadas (CloudWatch)

**Métricas a Publicar:**

| Métrica | Unidade | Dimensões |
|---------|---------|-----------|
| `Reconciliation.TotalLines` | Count | `job_id`, `fornecedor_id` |
| `Reconciliation.Divergences` | Count | `job_id`, `tipo_divergencia` |
| `Reconciliation.ProcessingTime` | Seconds | `job_id`, `stage` |
| `Reconciliation.Throughput` | Count/Second | `job_id` |
| `Reconciliation.ErrorRate` | Percent | `job_id`, `error_type` |

**Alertas:**
- `ProcessingTime > 18 minutes` → SNS → Slack (HIGH)
- `ErrorRate > 5%` → SNS → Slack (CRITICAL)
- `Divergences > 10%` → SNS → Email (MEDIUM)

---

### 10.3 Tracing (X-Ray)

**Configuração:**
- Habilitar X-Ray em Lambda, Step Functions, Glue
- Trace completo do fluxo E2E
- Identificar gargalos de latência

**Segmentos:**
- Ingestão
- Validação
- Conciliação
- Geração de Relatórios
- Notificação

---

### 10.4 Segurança

#### 10.4.1 Criptografia

- **Em Repouso:** S3 com SSE-KMS (AWS KMS)
- **Em Trânsito:** TLS 1.3 para todas as comunicações
- **Chaves:** Rotação automática a cada 90 dias

#### 10.4.2 Controle de Acesso (IAM)

**Princípio:** Least Privilege

**Roles:**
- `ReconciliationGlueRole`: Permissões para Glue Jobs (ler S3 ingress, escrever S3 output, ler RDS)
- `ReconciliationLambdaRole`: Permissões para Lambda (invocar Step Functions, publicar SNS)
- `ReconciliationQuickSightRole`: Permissões para QuickSight (ler S3 output)

**Políticas Mínimas:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::reconciliation-ingress/*",
        "arn:aws:s3:::reconciliation-output/*"
      ]
    }
  ]
}
```

#### 10.4.3 Mascaramento de Dados Sensíveis

**Campos a Mascarar em Logs:**
- `nome`: Mostrar apenas último nome (ex.: `***Silva`)
- `valor`: Manter completo (necessário para conciliação)
- `fornecedor_id`, `ordem_id`: Manter completo (necessário para rastreabilidade)

**Implementação:** Lambda function de mascaramento antes de escrever logs

---

### 10.5 Compliance e Auditoria

- **Retenção de Logs:** 90 dias em CloudWatch, 2 anos em S3 (Glacier)
- **Retenção de Dados:** 2 anos (conforme PRD)
- **Rastreabilidade:** Hash de linha em todos os registros
- **Auditoria:** CloudTrail para todas as ações de API

---

## 11. Recomendações de IaC (Infrastructure as Code)

### 11.1 Opção A: Terraform (Recomendado)

**Estrutura:**
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── s3/
│   ├── glue/
│   ├── step-functions/
│   ├── lambda/
│   └── monitoring/
└── environments/
    ├── dev/
    └── prod/
```

**Recursos Principais:**
- S3 buckets (ingress, output, logs)
- Glue Jobs e Scripts
- Step Functions State Machine
- Lambda Functions
- IAM Roles e Policies
- CloudWatch Alarms
- SNS Topics


---

## 12. Riscos, Mitigações e Próximos Passos

### 12.1 Matriz de Riscos

| Risco | Probabilidade | Impacto | Severidade | Mitigação |
|-------|--------------|---------|------------|-----------|
| **CSV com formato incorreto** | Média | Alto | ALTA | Validação rigorosa, notificação imediata |
| **Volume > 1.5M linhas** | Baixa | Alto | MÉDIA | Auto-scaling, alertas proativos |
| **Falha na base interna** | Baixa | Crítico | ALTA | Retry automático, fallback para cache |
| **Processamento > 20 min** | Média | Alto | ALTA | Otimização, paralelismo, monitoramento |
| **Perda de dados de auditoria** | Baixa | Crítico | ALTA | Backup automático, replicação |
| **Divergências não detectadas** | Média | Alto | ALTA | Testes abrangentes, validação cruzada |
| **Ataque de segurança** | Baixa | Crítico | ALTA | Criptografia, IAM, auditoria |
| **Falha de notificação** | Baixa | Médio | MÉDIA | Múltiplos canais, retry |

---

### 12.2 Próximos Passos Críticos

1. **Responder Perguntas de Decisão (Seção 2)**
   - Chave de correspondência
   - Tolerância numérica
   - Estratégia de processamento
   - Armazenamento de base interna
   - Formato de entrega
   - Estratégia de retry

2. **Aprovação de Arquitetura**
   - Revisar arquiteturas candidatas com equipe técnica
   - Escolher arquitetura final (A, B ou C)
   - Validar custos estimados

3. **Setup de Ambiente de Teste**
   - Provisionar ambiente dev/staging
   - Preparar dataset de 1M linhas
   - Configurar base interna de teste

4. **Início do Desenvolvimento**
   - Sprint planning (Fase 1 - MVP)
   - Setup de repositório e CI/CD
   - Kickoff do desenvolvimento

---

## 13. Conclusão

Este documento técnico apresenta uma análise completa da solução de conciliação financeira, incluindo:

- ✅ **6 perguntas de decisão críticas** que precisam ser respondidas antes da implementação
- ✅ **3 arquiteturas candidatas** na AWS com trade-offs detalhados
- ✅ **Tabela comparativa** com métricas de latência, custo, complexidade
- ✅ **Cálculos de performance** demonstrando viabilidade do SLA de 20 minutos
- ✅ **Plano de implementação** por fases (MVP → Produção)
- ✅ **Plano de testes de carga** com dados sintéticos e critérios de aceitação
- ✅ **Runbook e playbooks** detalhados para tratamento de falhas
- ✅ **Especificações de APIs** e formatos de saída
- ✅ **Recomendações de observabilidade e segurança**

**Recomendação Final:** Implementar **Arquitetura A (AWS Glue Serverless)** para menor complexidade operacional, ou **Arquitetura B (EMR)** para melhor custo/performance, dependendo da prioridade da equipe.

**✅ DECISÕES DEFINIDAS:**
- Chave de correspondência: `ordem_id` (único)
- Tolerância numérica: Diferença absoluta = 0 (exata)
- Arquitetura: AWS Glue (Serverless Batch)
- Base interna: RDS (PostgreSQL/MySQL)
- Visualização: Amazon QuickSight

**⚠️ VALIDAÇÕES NECESSÁRIAS ANTES DE PRODUÇÃO:**
1. Confirmar que índice único em `ordem_id` existe no RDS
2. Validar conectividade Glue → RDS (Security Groups, VPC)
3. Configurar QuickSight datasets e dashboards
4. Testar processamento completo de 1M linhas em ambiente de teste

---

**Documento gerado em:** 16 de Fevereiro de 2026  
**Versão:** 2.0  
**Status:** Pronto para Implementação

---

## 14. Resumo das Decisões Técnicas

| Decisão | Valor Escolhido | Justificativa |
|---------|----------------|---------------|
| **Chave de Correspondência** | `ordem_id` (único) | Simplifica matching, performance excelente com índice simples |
| **Tolerância Numérica** | Diferença absoluta = 0 | Regra clara, zero tolerância a erros |
| **Arquitetura** | AWS Glue (Serverless Batch) | Baixo overhead operacional, escalabilidade automática |
| **Base Interna** | RDS (PostgreSQL/MySQL) | Infraestrutura existente, aproveita investimento atual |
| **Visualização** | Amazon QuickSight | Dashboard interativo, melhor UX para área de negócio |

**Próximos Passos:**
1. ✅ Validar índice único em `ordem_id` no RDS
2. ✅ Configurar conectividade Glue → RDS
3. ✅ Desenvolver Glue Jobs (validação, conciliação, preparação QuickSight)
4. ✅ Configurar QuickSight datasets e dashboards
5. ✅ Executar testes de carga (1M linhas)
6. ✅ Deploy em produção
