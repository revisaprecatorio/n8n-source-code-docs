# Workflows de Alerta (Monitoramento)

Três workflows agendados que rodam a cada 10 minutos para monitorar estados de erro e notificar equipe e clientes.

---

## Índice

| Workflow | ID n8n | Detecta | Nós |
|---|---|---|---|
| [Alerta_ERROS_GRAVES](#alerta_erros_graves) | `GnL3nOy64DmpjHTD` | MANUAL_PROCESS, PIPELINE_ERROR, AUTH_ERROR, DOWNLOAD_FAILED | 8 |
| [Alerta_Laudo_Parcial](#alerta_laudo_parcial) | `nWttny9O5BjKabz2` | LAUDO_PARCIAL sem PARCIAL_INFORMADO | 7 |
| [Alerta_Reporte_Manual](#alerta_reporte_manual) | `XIx9gn1ifI7jsyoP` | MANUAL_PROCESS | 8 |

---

## Alerta_ERROS_GRAVES

**Detecta:** CPFs com `current_state IN (MANUAL_PROCESS, PIPELINE_ERROR, AUTH_ERROR, DOWNLOAD_FAILED)`
**Ação:** Notifica cliente via WhatsApp + equipe interna por email (2 destinatários)
**Estado final:** `ALERTA_MANUAL_SENT`

### Flowchart

```mermaid
flowchart LR
    SC([Schedule\n10min]) --> Q[(Query consultas_esaj\nJOIN process_tracking\ncurrent_state IN\nMANUAL_PROCESS\nPIPELINE_ERROR\nAUTH_ERROR\nDOWNLOAD_FAILED)]
    Q --> PM[Prepara Mensagens\npor item\nnome + processos + erro OCR]
    PM --> WA[WhatsApp Cliente\n7 dias uteis]
    WA --> E1["Email\ncontato@revisaprecatorio"]
    E1 --> E2[Email\npersival + rodrigo]
    E2 --> UPD[(UPDATE consultas_esaj\nALERTA_MANUAL_SENT)]
    UPD --> LOG[(INSERT logs\nAlerta grave)]

    classDef entrada fill:#238636,color:#fff,stroke:#2ea043
    classDef banco   fill:#0d419d,color:#fff,stroke:#1f6feb
    classDef alerta  fill:#da3633,color:#fff,stroke:#f85149
    classDef cinza   fill:#57606a,color:#fff,stroke:#6e7781

    class SC,WA entrada
    class Q,UPD,LOG banco
    class PM,E1,E2 cinza
```

### Query de Detecção

```sql
SELECT
    ce.id AS consulta_id, ce.cpf, ce.current_state,
    ce.whatsapp_from, ce.nome_requerente, ce.email, ce.processos,
    STRING_AGG(DISTINCT pt.mensagem_erro, ' | ') AS erros_ocr
FROM consultas_esaj ce
LEFT JOIN process_tracking pt
    ON pt.consulta_id = ce.id
    AND pt.evento = 'OCR_ERRO'
    AND pt.mensagem_erro IS NOT NULL
WHERE ce.current_state IN ('MANUAL_PROCESS','PIPELINE_ERROR','AUTH_ERROR','DOWNLOAD_FAILED')
GROUP BY ce.id, ce.cpf, ce.current_state, ce.whatsapp_from, ce.nome_requerente, ce.email, ce.processos
```

---

## Alerta_Laudo_Parcial

**Detecta:** Registros com evento `LAUDO_PARCIAL` em `process_tracking` que ainda **não** têm `PARCIAL_INFORMADO` (sem alerta duplicado)
**Ação:** Email interno para equipe — reprocessamento manual necessário
**Estado final:** INSERT `PARCIAL_INFORMADO` em `process_tracking`

### Flowchart

```mermaid
flowchart LR
    SC([Schedule\n10min]) --> Q[(Query Laudo Parcial\nprocess_tracking\nevento=LAUDO_PARCIAL\nsem PARCIAL_INFORMADO)]
    Q --> PM[Prepara Mensagens\nnome + processos + email]
    PM --> E1["Email\ncontato@revisaprecatorio"]
    E1 --> E2[Email\npersival + rodrigo]
    E2 --> INS[(INSERT process_tracking\nPARCIAL_INFORMADO\nevita reenvio)]
    INS --> LOG[(INSERT logs\nAlerta parcial)]

    classDef entrada fill:#238636,color:#fff,stroke:#2ea043
    classDef banco   fill:#0d419d,color:#fff,stroke:#1f6feb
    classDef parcial fill:#d29922,color:#000,stroke:#e3b341
    classDef cinza   fill:#57606a,color:#fff,stroke:#6e7781

    class SC entrada
    class Q,LOG banco
    class INS parcial
    class PM,E1,E2 cinza
```

### Query de Detecção (Anti-Duplicata)

```sql
SELECT pt.id as pt_id, pt.consulta_id, pt.cpf,
       ce.whatsapp_from, ce.nome_requerente, ce.email, ce.processos
FROM process_tracking pt
JOIN consultas_esaj ce ON ce.cpf = pt.cpf
WHERE pt.evento = 'LAUDO_PARCIAL'
AND NOT EXISTS (
    SELECT 1 FROM process_tracking pt2
    WHERE pt2.cpf = pt.cpf
      AND pt2.consulta_id IS NOT DISTINCT FROM pt.consulta_id
      AND pt2.evento = 'PARCIAL_INFORMADO'
)
ORDER BY ce.created_at ASC
```

---

## Alerta_Reporte_Manual

**Detecta:** CPFs com `current_state = MANUAL_PROCESS`
**Ação:** Notifica cliente via WhatsApp + equipe interna por email
**Estado final:** `ALERTA_MANUAL_SENT`

> **Diferença de `Alerta_ERROS_GRAVES`:** Detecta apenas `MANUAL_PROCESS` (sem os outros estados de erro) e atualiza via `WHERE current_state = 'MANUAL_PROCESS'` (batch) em vez de por `consulta_id`.

### Flowchart

```mermaid
flowchart LR
    SC([Schedule\n10min]) --> Q[(Query consultas_esaj\ncurrent_state\n= MANUAL_PROCESS)]
    Q --> PM[Prepara Mensagens\npor item]
    PM --> WA[WhatsApp Cliente\n7 dias uteis]
    WA --> E1["Email\ncontato@revisaprecatorio"]
    E1 --> E2[Email\npersival + rodrigo]
    E2 --> UPD[(UPDATE consultas_esaj\nALERTA_MANUAL_SENT\nWHERE MANUAL_PROCESS)]
    UPD --> LOG[(INSERT logs\nAlerta reporte manual)]

    classDef entrada fill:#238636,color:#fff,stroke:#2ea043
    classDef banco   fill:#0d419d,color:#fff,stroke:#1f6feb
    classDef cinza   fill:#57606a,color:#fff,stroke:#6e7781

    class SC,WA entrada
    class Q,UPD,LOG banco
    class PM,E1,E2 cinza
```

---

## Diagrama de Sequência Comparativo

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'actorBkg': '#1f6feb', 'actorTextColor': '#ffffff', 'actorBorder': '#388bfd', 'actorLineColor': '#8b949e', 'signalColor': '#8b949e', 'signalTextColor': '#c9d1d9', 'labelBoxBkgColor': '#21262d', 'labelBoxBorderColor': '#30363d', 'labelTextColor': '#c9d1d9', 'loopTextColor': '#c9d1d9', 'noteBkgColor': '#21262d', 'noteTextColor': '#c9d1d9', 'activationBkgColor': '#388bfd', 'activationBorderColor': '#58a6ff'}}}%%
sequenceDiagram
    participant SCH as Schedule (10min)
    participant WF as Workflow Alerta
    participant DB as PostgreSQL
    participant WA as WhatsApp
    participant EM as Email (SMTP)

    SCH->>WF: Dispara a cada 10 min

    alt Alerta_ERROS_GRAVES
        WF->>DB: SELECT consultas_esaj JOIN process_tracking<br/>WHERE state IN (MANUAL_PROCESS, PIPELINE_ERROR...)
    else Alerta_Laudo_Parcial
        WF->>DB: SELECT process_tracking WHERE LAUDO_PARCIAL<br/>AND NOT EXISTS PARCIAL_INFORMADO
    else Alerta_Reporte_Manual
        WF->>DB: SELECT consultas_esaj WHERE MANUAL_PROCESS
    end

    DB-->>WF: Linhas com CPFs afetados

    loop Para cada CPF encontrado
        WF->>WF: Prepara msg_cliente + msg_interna_html
        WF->>WA: Envia msg_cliente ao cliente
        WF->>EM: Email contato@revisaprecatorio
        WF->>EM: Email persival + rodrigo
    end

    alt Alerta_ERROS_GRAVES ou Alerta_Reporte_Manual
        WF->>DB: UPDATE consultas_esaj SET current_state=ALERTA_MANUAL_SENT
    else Alerta_Laudo_Parcial
        WF->>DB: INSERT process_tracking (PARCIAL_INFORMADO)
    end

    WF->>DB: INSERT logs
```

---

## Tabelas Afetadas

| Tabela | Alerta_ERROS_GRAVES | Alerta_Laudo_Parcial | Alerta_Reporte_Manual |
|---|---|---|---|
| `consultas_esaj` | R + UPDATE | R | R + UPDATE |
| `process_tracking` | R (LEFT JOIN) | R + INSERT | — |
| `logs` | INSERT | INSERT | INSERT |

---

## Mensagem ao Cliente (WhatsApp)

```
Prezado(a) {nome_requerente} recebemos a sua solicitação e informamos que,
devido a uma instabilidade do sistema do Tribunal de Justiça do Estado de
São Paulo, o seu laudo diagnóstico de precatório será enviado em até 7
(sete) dias úteis.

Contamos com a sua compreensão e agradecemos o contato.

Atenciosamente,
Revisa Precatório
```
