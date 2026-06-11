# Workflow: Laudo envio email+cpf

**ID n8n:** `(ver JSON)`
**Trigger:** `POST /webhook/reporte-email-cpf`
**Status:** ✅ Ativo
**Chamado por:** `calc-precatorio-tjsp/main.py` (via `pipeline_completo.sh`) e pela **Etapa 9b** do pipeline para CPFs 100% rejeitados.

---

## Nós (19 nós)

### Caminho Principal (todos processados)

| Nó | Tipo | Função |
|---|---|---|
| `Webhook Email + CPF` | webhook | Recebe `{ cpf, email }` |
| `Check Processamento Completo` | postgres | Verifica se todos os processos foram processados ou rejeitados |
| `Todos Processados?` | if | Bifurca: `todos_processados = true/false` |
| `Fetch Data for CPF` | postgres | `SELECT * FROM vw_precatorios_full WHERE cpf = ...` |
| `Build HTML Content` | function | Gera HTML completo do laudo |
| `Send Report Email` | emailSend | Envia laudo ao cliente (cc: revisaprecatorio) |
| `Log Success` | postgres | INSERT `logs` |
| `Webhook Response` | respondToWebhook | Responde 200 com resumo |
| `Update Report Status` | postgres | UPDATE `FINAL_REPORT_SENT` |
| `PT Report Enviado` | postgres | INSERT `process_tracking` (LAUDO_ENVIADO) |

### Caminho Parcial (nem todos processados)

| Nó | Tipo | Função |
|---|---|---|
| `Fetch Data for CPF - parcial` | postgres | `SELECT * FROM vw_precatorios_full WHERE cpf = ...` |
| `Build HTML Parcial` | function | Gera HTML parcial do laudo |
| `Send Report Revisa` | emailSend | Envia cópia interna para equipe |
| `phone e nome` | set | Extrai whatsapp_phone_number e nome |
| `Whatsapp Parcial` | whatsApp | Notifica cliente: "laudo em 7 dias úteis" |
| `Log Parcial e Manual` | postgres | INSERT `logs` |
| `Webhook Response Parcial` | respondToWebhook | Responde 200 parcial |
| `Update Partial Report` | postgres | UPDATE `PARTIAL_REPORT_SENT` |
| `PT Report Parcial` | postgres | INSERT `process_tracking` (LAUDO_PARCIAL) |

---

## Flowchart

```mermaid
flowchart TD
    WH([POST /reporte-email-cpf\n{ cpf, email }]) --> CPC[(Check Processamento Completo\nvw_precatorios_full\n+ esaj_calc_precatorio_resumo\n+ rejeitado=true conta como Processado)]
    CPC --> TP{Todos\nProcessados?}

    TP -->|true| FD[(Fetch Data for CPF\nvw_precatorios_full)]
    FD --> BH[Build HTML Content\nlaudo completo\nvalores, processos, indicadores]
    BH --> SE[Send Report Email\nao cliente + cc:revisa]
    SE --> LS[(Log Success\nlogs INSERT)]
    LS --> WR[Webhook Response 200]
    WR --> UR[(Update Report Status\nFINAL_REPORT_SENT)]
    UR --> PT1[(PT Report Enviado\nprocess_tracking\nLAUDO_ENVIADO)]

    TP -->|false| FDP[(Fetch Data for CPF parcial\nvw_precatorios_full)]
    FDP --> BHP[Build HTML Parcial]
    BHP --> SR[Send Report Revisa\nemail interno equipe]
    SR --> PN[phone e nome\nextrai whatsapp]
    PN --> WP[Whatsapp Parcial\n7 dias úteis]
    WP --> LP[(Log Parcial e Manual\nlogs INSERT)]
    LP --> WRP[Webhook Response Parcial 200]
    WRP --> UPR[(Update Partial Report\nPARTIAL_REPORT_SENT)]
    UPR --> PT2[(PT Report Parcial\nprocess_tracking\nLAUDO_PARCIAL)]

    style TP fill:#FF9800,color:#fff
    style PT1 fill:#1565C0,color:#fff
    style PT2 fill:#E65100,color:#fff
    style WP fill:#25D366,color:#fff
```

---

## Diagrama de Sequência

```mermaid
sequenceDiagram
    participant PIPE as pipeline_completo.sh
    participant WF as Laudo envio email+cpf
    participant DB as PostgreSQL
    participant EM as Email (SMTP)
    actor CLI as Cliente
    participant WA as WhatsApp

    PIPE->>WF: POST /reporte-email-cpf\n{ cpf, email }

    WF->>DB: Check Processamento Completo\nSELECT todos_processados\nFROM consultas_esaj + esaj_calc_precatorio_resumo\n+ vw_precatorios_full (rejeitado=true conta)

    alt todos_processados = true
        DB-->>WF: todos_processados: true
        WF->>DB: SELECT * FROM vw_precatorios_full WHERE cpf=...
        DB-->>WF: dados completos do laudo
        WF->>WF: Build HTML Content\n(valores, processos, indicadores idoso/doença)
        WF->>EM: Send Report Email → cliente\ncc: revisaprecatorio@dr.com
        EM-->>CLI: Email "Laudo Diagnóstico de Precatório"
        WF->>DB: INSERT logs
        WF->>WF: Webhook Response 200
        WF->>DB: UPDATE consultas_esaj FINAL_REPORT_SENT
        WF->>DB: INSERT process_tracking LAUDO_ENVIADO
    else todos_processados = false
        DB-->>WF: todos_processados: false
        WF->>DB: SELECT * FROM vw_precatorios_full WHERE cpf=...
        WF->>WF: Build HTML Parcial
        WF->>EM: Send Report Revisa → equipe interna
        WF->>WA: WhatsApp cliente\n"laudo em 7 dias úteis"
        WA-->>CLI: "Prezado(a)... 7 dias úteis"
        WF->>DB: INSERT logs
        WF->>WF: Webhook Response Parcial 200
        WF->>DB: UPDATE consultas_esaj PARTIAL_REPORT_SENT
        WF->>DB: INSERT process_tracking LAUDO_PARCIAL
    end
```

---

## Lógica de `Check Processamento Completo`

Esta é a query mais importante do workflow. Determina se o laudo é completo ou parcial:

```sql
WITH consulta_alvo AS (
    SELECT * FROM consultas_esaj
    WHERE cpf = '{cpf}' AND email = '{email}'
      AND current_state NOT IN ('REPORT_SENT', 'FINAL_REPORT_SENT')
    ORDER BY created_at DESC LIMIT 1
),
processos_esperados AS (
    SELECT ..., p.value ->> 'numero' AS numero_processo
    FROM consulta_alvo CROSS JOIN LATERAL jsonb_array_elements(c.processos -> 'lista') p
),
status_processos AS (
    SELECT ...,
        CASE
            WHEN r.numero_processo_cnj IS NOT NULL THEN 'Processado'  -- tem cálculo
            WHEN COALESCE(vp.rejeitado, false) = true THEN 'Processado'  -- rejeitado conta!
            ELSE 'Não Processado'
        END AS status_calculo
    FROM processos_esperados pe
    LEFT JOIN esaj_calc_precatorio_resumo r ON r.numero_processo_cnj = pe.numero_processo
    LEFT JOIN vw_precatorios_full vp ON vp.numero_processo_cnj = pe.numero_processo
)
SELECT ..., BOOL_AND(status_calculo = 'Processado') AS todos_processados
```

> **Ponto-chave:** `rejeitado = true` é considerado "Processado" — garante que CPFs 100% rejeitados recebam laudo completo (não parcial) quando acionados pela Etapa 9b.

---

## Etapa 9b — Trigger para CPFs 100% Rejeitados

Quando `calc-precatorio-tjsp/main.py` não gera nenhum registro em `esaj_calc_precatorio_resumo` (todos os processos rejeitados), o `pipeline_completo.sh` aciona este webhook diretamente:

```bash
# Etapa 9b em pipeline_completo.sh
curl -s --connect-timeout 10 --max-time 30 \
  -X POST "${N8N_WEBHOOK_BASE}/webhook/reporte-email-cpf" \
  -H "Content-Type: application/json" \
  -d "{\"cpf\": \"${CPF}\", \"email\": \"${EMAIL}\"}"
```

O workflow detecta `todos_processados = true` (via `rejeitado = true` conta como processado) e envia o laudo normalmente.

---

## Tabelas Afetadas

| Tabela | Operação |
|---|---|
| `consultas_esaj` | R (check) + UPDATE (FINAL/PARTIAL_REPORT_SENT) |
| `esaj_calc_precatorio_resumo` | R (check) |
| `vw_precatorios_full` | R (dados do laudo) |
| `process_tracking` | INSERT (LAUDO_ENVIADO / LAUDO_PARCIAL) |
| `logs` | INSERT |
