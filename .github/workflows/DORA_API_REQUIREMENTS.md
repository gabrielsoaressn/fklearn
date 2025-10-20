# Requisitos da API para Métricas DORA

Este documento descreve a rota que precisa ser implementada no backend para receber as métricas DORA do workflow do GitHub Actions.

## Endpoint Necessário

### POST /api/dora/deployment

**Descrição**: Registra informações de deployment para cálculo de métricas DORA.

**Content-Type**: `application/json`

**Payload Exemplo**:

```json
{
  "projectKey": "gabrielsoaressn_fklearn",
  "commitSha": "4f6a9021d1c9c8d4e83bd2563322582ddd164e1f",
  "commitTimestamp": "2025-10-20T10:56:13-03:00",
  "deploymentTimestamp": "2025-10-20T14:00:26Z",
  "status": "success",
  "branch": "master",
  "leadTimeSeconds": 253,
  "repository": "gabrielsoaressn/fklearn",
  "workflowRun": "18654338863"
}
```

**Campos do Payload**:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `projectKey` | string | Chave do projeto no SonarCloud (configurada nos secrets) |
| `commitSha` | string | Hash do commit que foi deployado |
| `commitTimestamp` | string (ISO 8601) | Timestamp de quando o commit foi feito |
| `deploymentTimestamp` | string (ISO 8601) | Timestamp de quando o deployment foi realizado |
| `status` | string | Status do deployment: `"success"` ou `"failure"` |
| `branch` | string | Nome da branch deployada |
| `leadTimeSeconds` | number | Lead Time em segundos (deploymentTimestamp - commitTimestamp) |
| `repository` | string | Nome do repositório no formato `owner/repo` |
| `workflowRun` | string | ID da execução do workflow no GitHub Actions |

**Respostas Esperadas**:

- **200 OK**: Deployment registrado com sucesso
  ```json
  {
    "message": "Deployment registrado com sucesso",
    "id": "deployment_id_123"
  }
  ```

- **400 Bad Request**: Payload inválido
  ```json
  {
    "error": "Payload inválido",
    "details": "Campo 'projectKey' é obrigatório"
  }
  ```

- **500 Internal Server Error**: Erro no servidor
  ```json
  {
    "error": "Erro ao processar deployment"
  }
  ```

## Métricas DORA que Este Endpoint Ajuda a Calcular

Este endpoint coleta dados para calcular as seguintes métricas DORA:

### 1. Deployment Frequency (Frequência de Deploy)
- **O que é**: Frequência com que código é deployado em produção
- **Como calcular**: Contar número de deployments com `status: "success"` por período de tempo
- **Exemplo**: "5 deployments por semana"

### 2. Lead Time for Changes (Tempo de Entrega de Mudanças)
- **O que é**: Tempo desde o commit até o deploy em produção
- **Como calcular**: Usar o campo `leadTimeSeconds`
- **Exemplo**: "253 segundos" ou "4 minutos e 13 segundos"

### 3. Change Failure Rate (Taxa de Falha de Mudanças)
- **O que é**: Percentual de deployments que falharam
- **Como calcular**: `(deployments com status "failure") / (total de deployments) * 100`
- **Exemplo**: "5% de falha"

### 4. Time to Restore Service (Tempo para Restaurar Serviço)
- **O que é**: Tempo para recuperar de uma falha
- **Como calcular**: Diferença entre timestamp de falha e próximo deploy bem-sucedido
- **Exemplo**: "30 minutos para restaurar"

## Implementação Sugerida (Node.js/Express)

```javascript
const express = require('express');
const router = express.Router();

// POST /api/dora/deployment
router.post('/deployment', async (req, res) => {
  try {
    const {
      projectKey,
      commitSha,
      commitTimestamp,
      deploymentTimestamp,
      status,
      branch,
      leadTimeSeconds,
      repository,
      workflowRun
    } = req.body;

    // Validação básica
    if (!projectKey || !commitSha || !status) {
      return res.status(400).json({
        error: 'Campos obrigatórios faltando',
        required: ['projectKey', 'commitSha', 'status']
      });
    }

    // Salvar no banco de dados
    const deployment = await database.deployments.create({
      projectKey,
      commitSha,
      commitTimestamp: new Date(commitTimestamp),
      deploymentTimestamp: new Date(deploymentTimestamp),
      status,
      branch,
      leadTimeSeconds,
      repository,
      workflowRun,
      createdAt: new Date()
    });

    res.status(200).json({
      message: 'Deployment registrado com sucesso',
      id: deployment.id
    });

  } catch (error) {
    console.error('Erro ao registrar deployment:', error);
    res.status(500).json({
      error: 'Erro ao processar deployment',
      message: error.message
    });
  }
});

module.exports = router;
```

## Schema do Banco de Dados Sugerido

```sql
CREATE TABLE dora_deployments (
  id SERIAL PRIMARY KEY,
  project_key VARCHAR(255) NOT NULL,
  commit_sha VARCHAR(40) NOT NULL,
  commit_timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
  deployment_timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
  status VARCHAR(20) NOT NULL CHECK (status IN ('success', 'failure')),
  branch VARCHAR(255) NOT NULL,
  lead_time_seconds INTEGER NOT NULL,
  repository VARCHAR(255) NOT NULL,
  workflow_run VARCHAR(50),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

  INDEX idx_project_key (project_key),
  INDEX idx_deployment_timestamp (deployment_timestamp),
  INDEX idx_status (status)
);
```

## Testando a Rota

Você pode testar a rota usando curl:

```bash
curl -X POST http://localhost:3000/api/dora/deployment \
  -H "Content-Type: application/json" \
  -d '{
    "projectKey": "gabrielsoaressn_fklearn",
    "commitSha": "4f6a9021d1c9c8d4e83bd2563322582ddd164e1f",
    "commitTimestamp": "2025-10-20T10:56:13-03:00",
    "deploymentTimestamp": "2025-10-20T14:00:26Z",
    "status": "success",
    "branch": "master",
    "leadTimeSeconds": 253,
    "repository": "gabrielsoaressn/fklearn",
    "workflowRun": "18654338863"
  }'
```

## Após Implementar a Rota

1. **Remover `continue-on-error: true`** do workflow em `.github/workflows/track-deployment.yml` (linha 103)
2. **Testar o workflow** fazendo um push para a branch master
3. **Verificar os logs** do GitHub Actions para confirmar sucesso

## Alteração "Executivo" para "Gerente"

No backend, procure por:
- Controllers, models ou views que usam a palavra "Executivo"
- Substituir por "Gerente" em todos os lugares relevantes
- Atualizar rotas, se houver (ex: `/api/executivo` → `/api/gerente`)
- Atualizar mensagens de UI/frontend
