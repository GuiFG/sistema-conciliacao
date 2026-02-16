### Prompt para o Agente Técnico — Avaliação e Proposta de Solução (AWS)

**Contexto resumido**  
A empresa recebe diariamente (D-1) um CSV do fornecedor com ~1.000.000 linhas contendo: **fornecedor_id, ordem_id, nome, ativo_id, data, valor**. A empresa tem uma base interna com os mesmos campos. É necessário um sistema de conciliação que identifique 3 tipos de divergência (linha só no CSV; só na base; presente em ambos com campos diferentes), processe tudo em **≤ 20 minutos**, notifique falhas e entregue as divergências de forma eficiente para a área de negócio. A solução final deve usar **AWS**.

---

### Objetivo do agente técnico
Avaliar tecnicamente o problema, propor **uma ou mais arquiteturas candidatas** na AWS com trade‑offs claros (latência, custo, complexidade operacional, segurança, escalabilidade), levantar **perguntas de decisão** antes de definir regras, e entregar um **plano de implementação** com testes de performance, runbook e artefatos IaC sugeridos.

---

### Entregáveis esperados
1. **Documento técnico** com arquitetura proposta(s), componentes, fluxos de dados e justificativas.  
2. **Comparativo de opções** (tabela) com métricas: tempo estimado, custo mensal estimado, complexidade, pontos de falha, facilidade de operação.  
3. **Lista de decisões pendentes** com perguntas claras e opções plausíveis (cada pergunta deve ter 3+ alternativas e recomendação).  
4. **Plano de implementação** por fases (MVP → produção), incluindo IaC (CloudFormation/Terraform) e estimativa de esforço.  
5. **Plano de testes de carga** (cenários, dados sintéticos, métricas de sucesso) demonstrando processamento de 1M linhas em ≤ 20 minutos.  
6. **Runbook e playbooks de falha** (notificação, retry, rollback, reprocessamento).  
7. **Especificação de APIs/artefatos de saída** para a área de negócio (CSV, S3, API, dashboard).  
8. **Recomendações de observabilidade e segurança** (logs, métricas, tracing, IAM, criptografia).  

---

### Restrições e requisitos não-funcionais (fixos)
- **Tempo máximo fim-a-fim:** 20 minutos por execução D-1.  
- **Volume:** 1.000.000 linhas por execução.  
- **Formato de entrada:** CSV (campos listados).  
- **Idempotência:** reprocessamentos não devem duplicar resultados.  
- **Notificação:** falhas e conclusão devem notificar área responsável.  
- **Auditoria:** rastreabilidade por linha (hash), retenção configurável.  
- **Uso de AWS** como nuvem alvo.  

---

### Tabela inicial de atributos para comparar opções (comece por aqui)
| **Atributo** | **Por que importa** | **Meta/valor desejado** |
|---|---:|---|
| **Latência (E2E)** | Cumprir SLA de 20 min | ≤ 20 minutos |
| **Throughput** | Processar 1M linhas | ≥ 50k–100k linhas/min (ajustar) |
| **Custo** | OPEX mensal e custo por execução | Minimizar sem violar SLA |
| **Complexidade operacional** | Tempo de manutenção e skills | Preferir menor complexidade possível |
| **Escalabilidade** | Picos e crescimento | Linear/automática |
| **Confiabilidade** | Tolerância a falhas | Retries, DLQ, idempotência |
| **Segurança/Compliance** | Dados sensíveis | Criptografia, IAM, masking |
| **Tempo de entrega (MVP)** | Prazo de implementação | 4–8 semanas (exemplo) |

---

### Instruções de trabalho para o agente técnico
1. **Antes de propor regras finais, liste todas as perguntas de decisão** necessárias e apresente para cada uma: (a) por que a pergunta importa; (b) 3+ opções plausíveis; (c) trade‑offs; (d) recomendação técnica com justificativa.  
2. **Proponha 2–3 arquiteturas candidatas** (ex.: serverless, containerizada, big‑data batch) usando serviços AWS concretos. Para cada arquitetura, detalhe: componentes, fluxo de dados, paralelismo/particionamento, pontos de escalonamento, custos estimados, e riscos.  
3. **Forneça estimativas de performance** (throughput por componente) e explique como atingir o objetivo de ≤ 20 minutos. Inclua cálculos e suposições. Use LaTeX para fórmulas simples se necessário.  
4. **Inclua plano de testes de carga** com dados sintéticos, métricas a coletar (tempo total, tempo por etapa, CPU/memória, I/O), e critérios de aceitação.  
5. **Defina requisitos de observabilidade**: métricas CloudWatch, logs estruturados, tracing (X‑Ray), alertas (SNS/Slack), dashboards (CloudWatch/QuickSight).  
6. **Defina runbook** para falhas (ex.: timeout >20min, parsing errors, S3 upload fail), com passos de mitigação e templates de notificação.  
7. **Forneça exemplos de payloads de saída** (CSV com coluna `tipo_divergencia`, JSON para API) e esquema.  
8. **Inclua recomendações IaC** (ex.: módulos Terraform/CloudFormation) e políticas IAM mínimas.  
9. **Apresente um cronograma de implementação** com milestones e estimativa de esforço por papel (Engenharia, SRE, QA, Negócio).  

---

### Exemplos de perguntas de decisão que o agente deve sempre gerar (cada pergunta deve ser respondida antes de fixar regras)
#### 1. Chave de correspondência (matching key)
- **Por que importa:** define unicidade e sensibilidade a duplicatas.  
- **Opções plausíveis:**  
  - **A:** `fornecedor_id + ordem_id` (simples, rápido).  
  - **B:** `fornecedor_id + ordem_id + ativo_id + data` (mais restritivo).  
  - **C:** gerar **hash** de todos os campos relevantes (`fornecedor_id, ordem_id, ativo_id, data, nome`) e usar como chave.  
- **Trade‑offs:** A é mais permissiva e pode gerar falsos positivos; B reduz falsos positivos; C garante rastreabilidade por linha e facilita detecção de alterações.  
- **Recomendação:** sugerir C para auditoria, com opção de usar B para matching primário.  

#### 2. Tolerância em valores numéricos (diferença aceitável)
- **Opções:**  
  - **A:** diferença absoluta = 0 (exata).  
  - **B:** tolerância fixa (ex.: R$0,01).  
  - **C:** tolerância percentual (ex.: 0.1%).  
- **Recomendação:** apresentar impacto operacional e sugerir default B com parametrização por fornecedor.

#### 3. Estratégia de processamento (batch vs streaming vs hybrid)
- **Opções:**  
  - **A:** **Batch distribuído** (EMR/Glue/Batch + S3) — bom para 1M linhas.  
  - **B:** **Serverless paralelo** (AWS Glue jobs, Lambda + Step Functions) — menor ops, limites de execução.  
  - **C:** **Containerizado em ECS/EKS** com workers paralelos — controle fino, maior ops.  
- **Trade‑offs:** custo, tempo de cold start, paralelismo, complexidade.  
- **Recomendação:** sugerir A ou B com justificativa baseada em SLAs e custo.

#### 4. Armazenamento e consulta dos dados de referência
- **Opções:** RDS (Postgres), Redshift, DynamoDB, ou S3 + Athena.  
- **Trade‑offs:** latência de lookup, custo, consistência, facilidade de joins.  
- **Recomendação:** para 1M linhas diárias e consultas massivas, considerar **S3 + Athena/Glue catalog** ou **Redshift** se joins complexos e baixa latência forem necessários.

#### 5. Como entregar divergências para negócio
- **Opções:** CSV em S3 + notificação; API REST para consulta; dashboard (QuickSight).  
- **Recomendação:** CSV em S3 + notificação imediata + dashboard para análise exploratória.

#### 6. Estratégia de retry e DLQ
- **Opções:** retries exponenciais com DLQ em SQS/S3; abortar e notificar; partial commit com checkpoint.  
- **Recomendação:** implementar retries com DLQ e reprocessamento idempotente.

---

### Arquiteturas candidatas (exemplos que o agente deve detalhar)
#### A. **Serverless Batch (recomendado para menor ops)**
- **Ingress:** upload CSV para S3 (prefixo D-1).  
- **Orquestração:** Step Functions inicia Glue job ou Lambda orchestration.  
- **Processamento:** AWS Glue (Spark) particionado por hash(fornecedor_id, ordem_id) → compara S3 CSV vs snapshot da base (exportada para S3/Glue catalog).  
- **Saída:** divergências escritas em S3 (CSV/Parquet), notificação via SNS/SES/Slack.  
- **Observabilidade:** CloudWatch, Glue metrics, X‑Ray.  
- **Prós/Contras:** baixo ops, escalável; custo Glue; atenção a limites de Glue startup.

#### B. **EMR / Spark Batch (alto desempenho para 1M+)**
- **Ingress:** S3.  
- **Processamento:** EMR cluster com Spark, jobs paralelos, joins distribuídos.  
- **Saída:** S3 + Redshift para análises.  
- **Prós/Contras:** alto throughput, custo controlável com spot; maior ops.

#### C. **Containerized Workers (ECS/EKS)**
- **Ingress:** S3 + SQS tasks.  
- **Processamento:** workers em paralelo que processam partições do CSV.  
- **Prós/Contras:** controle fino, bom para lógica customizada; maior overhead operacional.

---

### Métricas e metas técnicas (exemplos)
- **Throughput necessário:** se queremos 1.000.000 linhas em 20 minutos → \( \frac{1\,000\,000}{20} = 50\,000 \) linhas/minuto ≈ 833 linhas/segundo.  
- **Meta por worker:** se cada worker processa 200 linhas/s, precisamos ≈ 5 workers paralelos por shard; dimensionar com margem.  
- **Tempo por etapa:** ingestão (≤ 2 min), parsing/validation (≤ 5 min), matching (≤ 10 min), output/notify (≤ 3 min).

---

### Observabilidade, segurança e compliance (resumo)
- **Logs estruturados** (JSON) em CloudWatch + S3 para retenção.  
- **Métricas customizadas**: linhas processadas, divergências por tipo, tempo por etapa.  
- **Tracing**: AWS X‑Ray para latência E2E.  
- **Alertas:** CloudWatch Alarms → SNS → Slack/email.  
- **Segurança:** KMS para criptografia em repouso, TLS em trânsito, IAM least privilege, VPC endpoints para S3, masking de PII nos outputs.

---

### Requisitos de teste e validação
- **Testes unitários** para parsing e regras de matching.  
- **Testes de integração** com S3, Glue/EMR, DB de referência.  
- **Testes de carga**: simular 1M linhas com variação de tamanho de campos; medir E2E; testar reprocessamento e idempotência.  
- **Testes de falha**: injetar erros de rede, arquivos corrompidos, timeouts.

---

### Formato de resposta esperado do agente técnico
- **Comece listando perguntas de decisão** (priorize por impacto).  
- **Para cada pergunta**: apresente opções, trade‑offs e recomendação.  
- **Apresente 2–3 arquiteturas** com diagramas ASCII/descrição e tabela comparativa.  
- **Inclua cálculos de throughput** e suposições.  
- **Forneça plano de testes, runbook e entregáveis IaC** (exemplos de recursos a criar).  
- **Finalize com riscos, mitigação e próximos passos**.

---

### Exemplo de prompt curto que o agente deve usar internamente ao gerar a resposta
> "Avalie a solução de conciliação D-1 (1M linhas, SLA 20min). Liste perguntas de decisão críticas com 3+ opções cada. Proponha 2 arquiteturas AWS (serverless e EMR), calcule throughput necessário, detalhe trade‑offs, e entregue plano de implementação, testes de carga e runbook."

---

Use este prompt para produzir o **documento técnico inicial**. Sempre **pergunte antes de fixar regras** e, para cada decisão, apresente **opções plausíveis** com trade‑offs e uma recomendação técnica justificada.
