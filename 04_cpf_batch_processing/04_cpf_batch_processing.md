# Workflow: CPF_batch_processing

**ID n8n:** `jMzstMZfztUMz7O6`
**Trigger:** `POST /webhook/cpf-batch-processing`
**Status:** ✅ Ativo
**Uso:** Cadastro administrativo de CPFs sem passar pelo fluxo WhatsApp/pagamento. Insere diretamente com `current_state = PAYMENT_APPROVED` para forçar o processamento pelo crawler.

---

## Nós (6 nós)

| Nó | Tipo | Função |
|---|---|---|
| `Webhook Trigger` | webhook | Recebe `{ cpf }` via POST |
| `Extract CPF` | set | Sanitiza CPF (apenas dígitos), define `whatsapp_from` fixo |
| `Consulta e-SAJ` | httpRequest | GET e-SAJ por CPF — retorna HTML |
| `Parse e-SAJ Response` | code | Extrai nome e lista de processos do HTML |
| `Upsert Consulta` | postgres | INSERT/UPDATE `consultas_esaj` com `PAYMENT_APPROVED` |
| `Respond to Webhook` | respondToWebhook | Retorna JSON com resultado |

---

## Flowchart

```mermaid
flowchart LR
    WH([POST\n/cpf-batch-processing\ncpf]) --> EX[Extract CPF\nsanitiza digitos\nwhatsapp_from fixo]
    EX --> SAJ[Consulta e-SAJ\nGET HTML por CPF]
    SAJ --> PRS[Parse e-SAJ Response\nextrai nome + processos\ndo HTML]
    PRS --> UPS[(Upsert Consulta\nconsultas_esaj\nPAYMENT_APPROVED\nBATCH_epoch)]
    UPS --> RES([Respond 200\nsuccess cpf nome\ntotal processos])

    classDef entrada fill:#238636,color:#fff,stroke:#2ea043
    classDef wf      fill:#1f6feb,color:#fff,stroke:#388bfd
    classDef banco   fill:#0d419d,color:#fff,stroke:#1f6feb
    classDef sucesso fill:#1a7f37,color:#fff,stroke:#2da44e
    classDef cinza   fill:#57606a,color:#fff,stroke:#6e7781

    class WH entrada
    class EX,PRS cinza
    class SAJ wf
    class UPS banco
    class RES sucesso
```

---

## Diagrama de Sequência

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'actorBkg': '#1f6feb', 'actorTextColor': '#ffffff', 'actorBorder': '#388bfd', 'actorLineColor': '#8b949e', 'signalColor': '#8b949e', 'signalTextColor': '#c9d1d9', 'labelBoxBkgColor': '#21262d', 'labelBoxBorderColor': '#30363d', 'labelTextColor': '#c9d1d9', 'loopTextColor': '#c9d1d9', 'noteBkgColor': '#21262d', 'noteTextColor': '#c9d1d9', 'activationBkgColor': '#388bfd', 'activationBorderColor': '#58a6ff'}}}%%
sequenceDiagram
    actor Admin as Admin/Sistema
    participant WF as CPF_batch_processing
    participant SAJ as e-SAJ TJSP
    participant DB as consultas_esaj

    Admin->>WF: POST /cpf-batch-processing { cpf }
    WF->>WF: Sanitiza CPF (remove não-dígitos)
    WF->>SAJ: GET esaj.tjsp.jus.br/cpopg/search.do?cpf=...
    SAJ-->>WF: HTML com lista de processos
    WF->>WF: Parse HTML → nome + processos[]
    WF->>DB: INSERT ... ON CONFLICT DO UPDATE\ncurrent_state=PAYMENT_APPROVED\nmp_payment_status=approved\nmp_payment_id=BATCH_{epoch}
    DB-->>WF: id, cpf, nome_requerente, total_processos
    WF-->>Admin: 200 { success: true, cpf, nome, processos }

    Note over DB: Watchdog detecta PAYMENT_APPROVED<br/>→ Crawler inicia download dos PDFs
```

---

## Detalhes Técnicos

**URL e-SAJ consultada:**
```
https://esaj.tjsp.jus.br/cpopg/search.do?cbPesquisa=DOCPARTE&dadosConsulta.valorConsulta={cpf}&cdForo=-1
```

**SQL Upsert:**
```sql
INSERT INTO consultas_esaj (
  whatsapp_from, whatsapp_phone_number, cpf, nome_requerente,
  processos, total_processos, resposta_formatada,
  current_state, mp_payment_status, mp_payment_id, mp_payment_amount,
  payment_confirmed_at, ...timestamps...
)
VALUES (...)
ON CONFLICT (whatsapp_from, cpf) DO UPDATE SET
  nome_requerente = EXCLUDED.nome_requerente,
  processos = EXCLUDED.processos,
  current_state = EXCLUDED.current_state,
  mp_payment_status = EXCLUDED.mp_payment_status,
  ...
RETURNING id, cpf, nome_requerente, total_processos
```

> **Nota:** `mp_payment_id` é gerado como `BATCH_{epoch}` e `mp_payment_amount = 1.00` para simular pagamento aprovado sem passar pelo Mercado Pago.

---

## Tabelas Afetadas

| Tabela | Operação |
|---|---|
| `consultas_esaj` | INSERT / UPDATE (upsert) |

---

## Exemplo de Request/Response

**Request:**
```json
POST https://n8n.srv987902.hstgr.cloud/webhook/cpf-batch-processing
{ "cpf": "12345678901" }
```

**Response:**
```json
{
  "success": true,
  "cpf": "12345678901",
  "nome": "JOAO DA SILVA",
  "total_processos": 2,
  "processos": [
    { "numero": "0001234-56.2020.8.26.0053", "classe": "Precatório" },
    { "numero": "0007890-12.2019.8.26.0053", "classe": "Precatório" }
  ],
  "status_pagamento": "PAYMENT_APPROVED"
}
```
