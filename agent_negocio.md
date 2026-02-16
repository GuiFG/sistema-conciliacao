### Prompt inicial para o Agente de Negócio gerar o PRD detalhado

**Contexto resumido**  
A empresa recebe diariamente (D-1) um CSV do fornecedor com transações financeiras (1 milhão de linhas por dia). A empresa já possui uma base interna com os mesmos campos. É necessário um **sistema de conciliação** que identifique divergências entre as bases em até **20 minutos**, notifique falhas e entregue um pacote de divergências consumível pela área de negócio.

---

### Objetivo do PRD
Produzir um Documento de Requisitos do Produto (PRD) completo e acionável que descreva: **fluxo de ingestão**, **regras de conciliação**, **tipos de divergência**, **entregáveis para a área de negócio**, **SLAs operacionais**, **requisitos de segurança e compliance**, **métricas de sucesso**, e **dependências técnicas e organizacionais**.

---

### Escopo e conteúdo esperado no PRD
O PRD deve conter, no mínimo, as seções e detalhes abaixo. Para cada item, descreva requisitos funcionais, critérios de aceitação e exemplos concretos.

- **Dados de entrada**
  - **Formato do CSV**: campos obrigatórios: **fornecedor_id; ordem_id; nome; ativo_id; data; valor**.
  - **Validações iniciais**: esquema, tipos, formatos de data, valores nulos, duplicatas.
  - **Volume esperado**: 1.000.000 linhas por execução D-1.

- **Regras de conciliação**
  - **Chave de correspondência** (definir combinação de campos que identifica uma linha única).
  - **Tipos de divergência**:
    1. **Presente no CSV, ausente na base da empresa**.
    2. **Presente na base da empresa, ausente no CSV**.
    3. **Presente em ambos, mas com campos divergentes** — especificar quais campos são críticos (ex.: valor, data).
  - **Política de tolerância**: diferenças numéricas aceitáveis (ex.: tolerância de centavos), regras de arredondamento, tratamento de fusos horários.

- **Processamento e performance**
  - **Tempo máximo**: 20 minutos do início da ingestão até a entrega do resultado consolidado.
  - **Requisitos de escalabilidade**: picos, paralelismo, particionamento por fornecedor/ordem/data.
  - **Critérios de retry e falha**: quando reprocessar, backoff, limites.

- **Notificações e entrega das divergências**
  - **Notificação de falha**: quem é notificado, canal (e-mail, SNS, Slack), conteúdo mínimo da notificação.
  - **Entrega de resultados**: formatos exigidos (CSV com marcação de tipo de divergência; dashboard; API); payload mínimo por divergência.
  - **Prioritização**: como agrupar/ordenar divergências para ação rápida pela área de negócio.

- **Observabilidade e auditoria**
  - Logs de ingestão, métricas de latência, contadores de divergência por tipo, rastreabilidade de linha (hashs), retenção de dados para auditoria.

- **Segurança e compliance**
  - Criptografia em trânsito e repouso, controle de acesso, mascaramento de dados sensíveis, requisitos regulatórios aplicáveis.

- **Operação e escalonamento**
  - Runbook para falhas, responsáveis, SLA de resposta, contatos de emergência.

---

### Entregáveis e Critérios de Aceitação
- **Entregáveis**:
  - PRD completo em documento (PDF/Markdown) com diagramas de fluxo.
  - Exemplo de CSV de entrada e CSV de saída com divergências marcadas.
  - Lista de APIs/events necessários para integração com times consumidores.
  - Matriz de riscos e mitigação.
- **Critérios de Aceitação**:
  - Documento aprovado pelo Product Owner e pelo time de operações.
  - Cenário de teste com 1M linhas simulado e tempo de processamento ≤ 20 minutos (proposta de teste de carga).
  - Definição clara de notificações e template de mensagem.

---

### Stakeholders e responsabilidades
| **Stakeholder** | **Papel** |
|---|---|
| **Product Owner** | Aprovar PRD e priorizar requisitos |
| **Área de Conciliação / Negócio** | Consumir divergências; validar regras de negócio |
| **Equipe de Dados / Engenharia** | Implementar pipeline e infra |
| **Fornecedor** | Garantir formato e qualidade do CSV |
| **Operações / SRE** | Monitoramento e runbooks |

---

### Exemplos e artefatos que o agente deve gerar no PRD
- **Exemplo de linha CSV válida**: `12345;ORD-98765;João Silva;ATV-001;2026-02-15;1500.75`
- **Exemplo de divergência tipo 3**: mesma chave, mas `valor` no CSV = `1500.75` e na base = `1500.00` — indicar regra de tolerância e ação recomendada.
- **Template de notificação de falha** com campos mínimos: `job_id, timestamp, motivo, amostra de 5 linhas com erro, link para logs`.

---

### Perguntas abertas e dependências que o agente deve levantar
- Qual é a **chave única** definitiva para correspondência (combinação de campos)?  
- Há **regras de negócio** específicas para tratar ordens duplicadas ou reversões?  
- Quais **canais de notificação** são obrigatórios e quem são os contatos?  
- Existe um **ambiente de teste** com 1M de linhas para validação de performance?  
- Requisitos legais de **retenção** e **mascaramento** de dados sensíveis?

---

### Instruções finais para o agente
1. Produza o PRD em **Português**, estruturado conforme seções acima.  
2. Inclua **diagramas de fluxo** (alto nível) e **exemplos concretos** de CSV de entrada/saída.  
3. Liste **testes de aceitação** e um **plano de validação de performance** para garantir o processamento em ≤ 20 minutos com 1M linhas.  
4. Termine com uma **lista de decisões pendentes** e um plano de próximas ações (quem deve responder cada dependência e prazo sugerido).

---

Use este prompt como base para gerar o PRD inicial e, em seguida, peça validação à área de negócio para responder as perguntas abertas e confirmar a chave de correspondência e canais de notificação.
