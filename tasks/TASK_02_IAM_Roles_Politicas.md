# TASK 02: Configuração de IAM Roles e Políticas

**Sprint:** 1-2 (Infraestrutura Base)  
**Prioridade:** CRÍTICA  
**Estimativa:** 1-2 dias  
**Dependências:** TASK 01 (Buckets S3 criados)

---

## Objetivo

Criar e configurar todas as IAM Roles e Políticas necessárias para o sistema de conciliação, seguindo o princípio do menor privilégio (Least Privilege).

---

## Requisitos

- AWS Account configurada
- Permissões IAM para criar roles e políticas
- Buckets S3 criados (TASK 01)
- RDS existente (para configurar acesso)

---

## Passo a Passo Detalhado

### 1. Criar KMS Key para Criptografia

**Objetivo:** Chave KMS para criptografia de dados em repouso

**Passos:**

1.1. Criar chave KMS:
```bash
aws kms create-key \
  --description "Chave para criptografia de dados de conciliação" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT
```

1.2. Criar alias:
```bash
aws kms create-alias \
  --alias-name alias/reconciliation-key \
  --target-key-id <KEY_ID>
```

1.3. Criar política da chave (permitir uso por serviços AWS):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow Glue to use key",
      "Effect": "Allow",
      "Principal": {
        "Service": "glue.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow Lambda to use key",
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

1.4. Aplicar política:
```bash
aws kms put-key-policy \
  --key-id <KEY_ID> \
  --policy-name default \
  --policy file://kms-policy.json
```

---

### 2. Criar IAM Role para AWS Glue

**Objetivo:** Role com permissões para Glue Jobs acessarem S3 e RDS

**Passos:**

2.1. Criar política para Glue:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::reconciliation-ingress/*",
        "arn:aws:s3:::reconciliation-output/*",
        "arn:aws:s3:::reconciliation-intermediate/*",
        "arn:aws:s3:::reconciliation-checkpoints/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::reconciliation-ingress",
        "arn:aws:s3:::reconciliation-output",
        "arn:aws:s3:::reconciliation-intermediate",
        "arn:aws:s3:::reconciliation-checkpoints"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:ACCOUNT_ID:key/<KEY_ID>"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:ACCOUNT_ID:log-group:/aws-glue/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:CreateDatabase",
        "glue:CreateTable",
        "glue:UpdateTable"
      ],
      "Resource": "*"
    }
  ]
}
```

2.2. Criar política:
```bash
aws iam create-policy \
  --policy-name ReconciliationGluePolicy \
  --policy-document file://glue-policy.json
```

2.3. Criar role para Glue:
```bash
aws iam create-role \
  --role-name ReconciliationGlueRole \
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
```

2.4. Anexar política à role:
```bash
aws iam attach-role-policy \
  --role-name ReconciliationGlueRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/ReconciliationGluePolicy
```

2.5. Anexar managed policy do Glue:
```bash
aws iam attach-role-policy \
  --role-name ReconciliationGlueRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
```

---

### 3. Criar IAM Role para Lambda

**Objetivo:** Role com permissões para Lambda trigger

**Passos:**

3.1. Criar política para Lambda:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::reconciliation-ingress/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "states:StartExecution"
      ],
      "Resource": "arn:aws:states:us-east-1:ACCOUNT_ID:stateMachine:reconciliation-pipeline"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:ACCOUNT_ID:log-group:/aws/lambda/reconciliation-trigger:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:ACCOUNT_ID:key/<KEY_ID>"
    }
  ]
}
```

3.2. Criar política:
```bash
aws iam create-policy \
  --policy-name ReconciliationLambdaPolicy \
  --policy-document file://lambda-policy.json
```

3.3. Criar role para Lambda:
```bash
aws iam create-role \
  --role-name ReconciliationLambdaRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "lambda.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

3.4. Anexar política à role:
```bash
aws iam attach-role-policy \
  --role-name ReconciliationLambdaRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/ReconciliationLambdaPolicy
```

---

### 4. Criar IAM Role para Step Functions

**Objetivo:** Role com permissões para Step Functions orquestrar o pipeline

**Passos:**

4.1. Criar política para Step Functions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "glue:StartJobRun",
        "glue:GetJobRun",
        "glue:GetJobRuns",
        "glue:BatchStopJobRun"
      ],
      "Resource": [
        "arn:aws:glue:us-east-1:ACCOUNT_ID:job/reconciliation-validation",
        "arn:aws:glue:us-east-1:ACCOUNT_ID:job/reconciliation-reconciliation",
        "arn:aws:glue:us-east-1:ACCOUNT_ID:job/reconciliation-quicksight-prep"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": [
        "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:reconciliation-notification"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:us-east-1:ACCOUNT_ID:reconciliation-alerts"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogDelivery",
        "logs:GetLogDelivery",
        "logs:UpdateLogDelivery",
        "logs:DeleteLogDelivery",
        "logs:ListLogDeliveries",
        "logs:PutResourcePolicy",
        "logs:DescribeResourcePolicies",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

4.2. Criar política:
```bash
aws iam create-policy \
  --policy-name ReconciliationStepFunctionsPolicy \
  --policy-document file://stepfunctions-policy.json
```

4.3. Criar role para Step Functions:
```bash
aws iam create-role \
  --role-name ReconciliationStepFunctionsRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "states.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

4.4. Anexar política à role:
```bash
aws iam attach-role-policy \
  --role-name ReconciliationStepFunctionsRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/ReconciliationStepFunctionsPolicy
```

---

### 5. Configurar Acesso ao RDS

**Objetivo:** Permitir que Glue acesse o RDS (base interna)

**Passos:**

5.1. Obter Security Group do RDS:
```bash
aws rds describe-db-instances \
  --db-instance-identifier <RDS_INSTANCE_ID> \
  --query 'DBInstances[0].VpcSecurityGroups'
```

5.2. Obter Security Group do Glue (será criado automaticamente):
   - Glue cria VPC endpoints automaticamente
   - Ou configurar Security Group do Glue para acessar RDS

5.3. Adicionar regra de entrada no Security Group do RDS:
```bash
aws ec2 authorize-security-group-ingress \
  --group-id <RDS_SECURITY_GROUP_ID> \
  --protocol tcp \
  --port 5432 \
  --source-group <GLUE_SECURITY_GROUP_ID>
```

5.4. Criar usuário no RDS para Glue (se necessário):
```sql
-- Conectar ao RDS PostgreSQL
CREATE USER reconciliation_glue WITH PASSWORD 'SECURE_PASSWORD';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reconciliation_glue;
GRANT USAGE ON SCHEMA public TO reconciliation_glue;
```

---

### 6. Criar IAM Role para QuickSight (Opcional - se usar QuickSight)

**Objetivo:** Permitir QuickSight ler dados do S3

**Passos:**

6.1. Criar política para QuickSight:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::reconciliation-output",
        "arn:aws:s3:::reconciliation-output/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions"
      ],
      "Resource": "*"
    }
  ]
}
```

6.2. Criar role para QuickSight:
```bash
aws iam create-role \
  --role-name ReconciliationQuickSightRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "quicksight.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

6.3. Anexar política:
```bash
aws iam attach-role-policy \
  --role-name ReconciliationQuickSightRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/ReconciliationQuickSightPolicy
```

---

## Validação

### Checklist de Validação:

- [ ] KMS Key criada e alias configurado
- [ ] Role `ReconciliationGlueRole` criada com políticas corretas
- [ ] Role `ReconciliationLambdaRole` criada com políticas corretas
- [ ] Role `ReconciliationStepFunctionsRole` criada com políticas corretas
- [ ] Role `ReconciliationQuickSightRole` criada (se necessário)
- [ ] Security Group do RDS configurado para permitir acesso do Glue
- [ ] Usuário RDS criado para Glue (se necessário)
- [ ] Teste de acesso: Glue pode ler S3
- [ ] Teste de acesso: Glue pode conectar ao RDS
- [ ] Teste de acesso: Lambda pode invocar Step Functions

### Comandos de Validação:

```bash
# Listar roles criadas
aws iam list-roles --query 'Roles[?contains(RoleName, `Reconciliation`)].RoleName'

# Verificar políticas anexadas
aws iam list-attached-role-policies --role-name ReconciliationGlueRole

# Verificar política inline
aws iam get-role-policy --role-name ReconciliationGlueRole --policy-name <POLICY_NAME>

# Testar assume role (simulação)
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/ReconciliationGlueRole \
  --role-session-name test-session
```

---

## Notas Técnicas

- **Princípio do Menor Privilégio:** Cada role tem apenas as permissões necessárias
- **RDS Access:** Garantir que Security Groups permitem conexão do Glue
- **KMS Key:** Usar mesma chave para todos os buckets (consistência)
- **Rotação de Chaves:** Configurar rotação automática do KMS key (90 dias)

---

## Próximas Tasks

- **TASK 03:** Implementar Lambda Trigger (requer Lambda role criada)
- **TASK 04:** Configurar Step Functions (requer Step Functions role criada)
- **TASK 06:** Implementar Glue Jobs (requer Glue role criada)

---

## Referências

- Documento Técnico - Seção 10.4.2 (Controle de Acesso IAM)
- AWS IAM Best Practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
