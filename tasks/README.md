# Tasks de Implementação - MVP 1 Sistema de Conciliação Financeira

Este diretório contém arquivos detalhados de tasks para implementação do MVP 1 do Sistema de Conciliação Financeira.

---

## Estrutura das Tasks

As tasks estão organizadas por Sprint conforme o Documento Técnico (Seção 6.1):

### Sprint 1-2: Infraestrutura Base (2 semanas)
- **TASK_01_Infraestrutura_S3.md** - Configuração de buckets S3
- **TASK_02_IAM_Roles_Politicas.md** - Configuração de IAM roles e políticas
- **TASK_03_Lambda_Trigger.md** - Implementação de Lambda de trigger
- **TASK_04_Step_Functions.md** - Configuração de Step Functions (orquestração)
- **TASK_05_CloudWatch_Logs_Metricas.md** - Setup de CloudWatch Logs e métricas

### Sprint 3-4: Processamento Core (2 semanas)
- **TASK_06_Glue_Job_Validacao.md** - Implementação de Glue Job de validação e parsing
- **TASK_07_Glue_Job_Conciliacao.md** - Implementação de Glue Job de conciliação
- **TASK_08_Classificacao_Divergencias.md** - Implementação de classificação de divergências

### Sprint 5-6: Saída, QuickSight e Notificações (2 semanas)
- **TASK_09_Geracao_Parquet_QuickSight.md** - Geração de Parquet otimizado para QuickSight
- **TASK_10_Glue_Data_Catalog.md** - Configuração de Glue Data Catalog
- **TASK_11_QuickSight_Dashboard.md** - Configuração de QuickSight dataset e dashboard
- **TASK_12_SNS_Notificacoes.md** - Configuração de SNS e notificações
- **TASK_13_Idempotencia_Hash.md** - Implementação de idempotência e hash
- **TASK_14_Testes_End_to_End.md** - Testes end-to-end completos

---

## Como Usar Este Diretório

### 1. Leitura Inicial
- Comece lendo este README para entender a estrutura
- Revise o **Documento_Tecnico_Solucao.md** e **PRD_Sistema_Conciliação.md** para contexto completo

### 2. Execução das Tasks
- Execute as tasks **na ordem numérica** (TASK_01 → TASK_02 → ...)
- Cada task tem dependências claramente indicadas
- Valide cada task antes de prosseguir para a próxima

### 3. Estrutura de Cada Task
Cada arquivo de task contém:
- **Objetivo:** O que a task pretende alcançar
- **Sprint:** Em qual sprint a task deve ser executada
- **Prioridade:** CRÍTICA, ALTA, MÉDIA, BAIXA
- **Estimativa:** Tempo estimado de implementação
- **Dependências:** Tasks que devem ser concluídas antes
- **Passo a Passo Detalhado:** Instruções específicas de implementação
- **Validação:** Checklist e comandos para validar a implementação
- **Notas Técnicas:** Informações importantes e considerações
- **Próximas Tasks:** Tasks que dependem desta
- **Referências:** Documentos e links relevantes

---

## Requisitos Mínimos de Entrega (MVP 1)

Conforme Documento Técnico - Seção 6.1, o MVP 1 deve entregar:

✅ **Sistema processa CSV e gera Parquet de divergências**
- Ingestão de CSV do S3
- Validação de dados
- Conciliação com base interna (RDS)
- Identificação de divergências (Tipo 1, 2, 3)
- Geração de Parquet com divergências

✅ **Dashboard QuickSight funcional**
- Dataset conectado ao Glue Data Catalog
- Dashboard com visualizações básicas
- Filtros e métricas principais

✅ **Notificações básicas funcionando**
- Notificações de sucesso e falha
- Envio via SNS (Email + Slack)

✅ **Logs e métricas básicas**
- CloudWatch Logs configurado
- Métricas customizadas publicadas
- Dashboard CloudWatch básico

---

## Critérios de Aceitação do MVP 1

Conforme PRD - Seção 10.2:

✅ **Performance:**
- Sistema processa 1M linhas em ≤ 20 minutos (P95)
- Tempo de ingestão ≤ 3 minutos
- Tempo de conciliação ≤ 15 minutos
- Tempo de geração de relatórios ≤ 2 minutos

✅ **Funcionalidade:**
- Todas as validações implementadas (PRD - Seção 2.2)
- Todos os tipos de divergência identificados (PRD - Seção 3.2)
- Classificação e severidade corretas
- Dados disponíveis no QuickSight

✅ **Observabilidade:**
- Logs estruturados (JSON)
- Métricas customizadas publicadas
- Alertas configurados

---

## Dependências Entre Tasks

```
TASK_01 (S3)
    ↓
TASK_02 (IAM) → TASK_03 (Lambda) → TASK_04 (Step Functions)
    ↓                                    ↓
TASK_05 (CloudWatch)              TASK_06 (Glue Validação)
                                        ↓
                                  TASK_07 (Glue Conciliação)
                                        ↓
                                  TASK_08 (Classificação)
                                        ↓
                                  TASK_09 (Parquet QuickSight)
                                        ↓
                                  TASK_10 (Data Catalog) → TASK_11 (QuickSight)
                                        ↓
                                  TASK_12 (SNS)
                                        ↓
                                  TASK_13 (Idempotência)
                                        ↓
                                  TASK_14 (Testes E2E)
```

---

## Ambiente e Pré-requisitos

### AWS Account
- Conta AWS configurada
- Permissões adequadas para criar recursos
- Região: `us-east-1` (ou conforme necessário)

### Conhecimentos Necessários
- AWS Services: S3, IAM, Lambda, Step Functions, Glue, QuickSight, SNS, CloudWatch
- Python (para scripts Lambda e Glue)
- PySpark/Spark (para Glue Jobs)
- SQL (para queries RDS)

### Infraestrutura Existente
- RDS com base interna (PostgreSQL ou MySQL)
- Índice único em `ordem_id` no RDS (validar antes de TASK_07)
- Acesso à base interna do Glue (Security Groups configurados)

---

## Próximos Passos Após MVP 1

Conforme Documento Técnico - Seção 6.2, após MVP 1:

**Fase 2: Produção (2-3 semanas)**
- Observabilidade completa (X-Ray, métricas avançadas)
- Segurança completa (criptografia, mascaramento)
- Checkpoint em caso de timeout/falha
- Otimizações de performance
- Runbook detalhado

---

## Suporte e Referências

### Documentos Principais
- `PRD_Sistema_Conciliação.md` - Requisitos de negócio
- `Documento_Tecnico_Solucao.md` - Arquitetura técnica e decisões

### Documentação AWS
- AWS Glue: https://docs.aws.amazon.com/glue/
- AWS Step Functions: https://docs.aws.amazon.com/step-functions/
- Amazon QuickSight: https://docs.aws.amazon.com/quicksight/
- AWS SNS: https://docs.aws.amazon.com/sns/

---

## Notas Importantes

1. **SLA Crítico:** Sistema deve processar 1M linhas em ≤ 20 minutos
2. **Chave de Correspondência:** Usar apenas `ordem_id` (conforme decisão técnica)
3. **Tolerância:** Diferença absoluta = 0 (sem tolerância, mas reportar diferenças ≤ R$ 0,01 como LOW)
4. **Arquitetura:** AWS Glue (Serverless Batch) com Spark
5. **Base Interna:** RDS (PostgreSQL/MySQL) - já existente
6. **Visualização:** Amazon QuickSight

---

**Última atualização:** 16 de Fevereiro de 2026  
**Versão:** 1.0
