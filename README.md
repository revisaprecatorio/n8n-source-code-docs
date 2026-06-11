# Workflows n8n — Revisa Precatório

Documentação detalhada de todos os workflows ativos no n8n. Os JSONs exportados estão em `workflows_n8n/`.

**Instância n8n:** `https://n8n.srv987902.hstgr.cloud`
**Última atualização:** 2026-06-11

---

## Índice e Ordem de Execução na Plataforma

| # | Arquivo | Workflow | Trigger | Status |
|---|---------|----------|---------|--------|
| 1 | [01_chatbot_revisa.md](01_chatbot_revisa.md) | Chatbot Revisa | WhatsApp webhook | ✅ Ativo |
| 2 | [02_mercado_pago.md](02_mercado_pago.md) | Mercado Pago Unified | Webhook (2 endpoints) | ✅ Ativo |
| 3 | [03_laudo_envio.md](03_laudo_envio.md) | Laudo envio email+cpf | Webhook POST | ✅ Ativo |
| 4 | [04_cpf_batch_processing.md](04_cpf_batch_processing.md) | CPF_batch_processing | Webhook POST | ✅ Ativo |
| 5 | [05_alertas.md](05_alertas.md) | Alerta_ERROS_GRAVES | Schedule 10min | ✅ Ativo |
| 5 | [05_alertas.md](05_alertas.md) | Alerta_Laudo_Parcial | Schedule 10min | ✅ Ativo |
| 5 | [05_alertas.md](05_alertas.md) | Alerta_Reporte_Manual | Schedule 10min | ✅ Ativo |

---

## Fluxo Completo da Plataforma

```mermaid
flowchart TD
    WA([Cliente\nWhatsApp]) -->|CPF| CW[1. Chatbot Revisa\nwebhook/chatbot]
    CW -->|email + confirmação| CW
    CW -->|trigger_payment=true| MP[2. Mercado Pago Unified\nwebhook/generate-payment-link]
    MP -->|link MP| WA
    WA -->|paga| MP
    MP -->|webhook approved\nmercadopago-notification| MP
    MP -->|PAYMENT_APPROVED\n+ DELETE OCR antigo| DB[(consultas_esaj\nPAYMENT_APPROVED)]

    DB -->|watchdog Windows Server| CR[Crawler TJSP\nSelenium + Chrome]
    CR -->|PDFs baixados| OCR[pipeline_completo.sh\nOCR + Ingestão]
    OCR -->|POST /reporte-email-cpf| LE[3. Laudo envio email+cpf]
    OCR -->|100% rejeitados\nEtapa 9b| LE

    LE -->|todos_processados=true| EMAIL[Email ao cliente\nFINAL_REPORT_SENT]
    LE -->|todos_processados=false| PARCIAL[Email Revisa + WA\nPARTIAL_REPORT_SENT]

    BATCH([Admin/Sistema]) -->|POST /cpf-batch-processing| BP[4. CPF_batch_processing]
    BP -->|PAYMENT_APPROVED direto| DB

    SCHED([Schedule 10min]) --> AG[5. Alertas\nERROS_GRAVES\nLaudo_Parcial\nReporte_Manual]
    AG -->|WA + Email| NOTIF[Notificações\ncliente + equipe]

    style WA fill:#25D366,color:#fff
    style EMAIL fill:#2196F3,color:#fff
    style PARCIAL fill:#FF9800,color:#fff
    style SCHED fill:#9E9E9E,color:#fff
    style OCR fill:#7B1FA2,color:#fff
```

---

## Diagrama de Sequência — Jornada Completa do Cliente

```mermaid
sequenceDiagram
    actor Cliente as Cliente (WhatsApp)
    participant CW as Chatbot Revisa
    participant DB as PostgreSQL
    participant MP as Mercado Pago
    participant WS as Windows Server
    participant OCR as pipeline_completo.sh
    participant LE as Laudo envio email+cpf

    Cliente->>CW: Envia CPF via WhatsApp
    CW->>DB: INSERT consultas_esaj (AWAITING_EMAIL)
    CW-->>Cliente: "Informe seu email"

    Cliente->>CW: Informa email
    CW->>DB: UPDATE (AWAITING_CODE)
    CW-->>Cliente: Envia código por email

    Cliente->>CW: Digita código
    CW->>DB: Consulta e-SAJ, UPDATE (AWAITING_CONFIRMATION)
    CW-->>Cliente: Mostra processos encontrados

    Cliente->>CW: Confirma
    CW->>MP: POST /generate-payment-link
    MP->>DB: UPDATE (AWAITING_PAYMENT + mp_preference_id)
    MP-->>Cliente: Envia link de pagamento WhatsApp

    Cliente->>MP: Realiza pagamento
    MP->>MP: POST /mercadopago-notification
    MP-->>Cliente: Responde 200 OK imediato
    MP->>DB: UPDATE (PAYMENT_APPROVED)
    MP->>DB: DELETE esaj_detalhe_processos + esaj_calc_precatorio_resumo
    MP-->>Cliente: "🎉 Pagamento confirmado!"

    WS->>WS: watchdog detecta PAYMENT_APPROVED
    WS->>WS: Crawler baixa PDFs do e-SAJ
    WS->>OCR: Executa pipeline_completo.sh

    OCR->>DB: INSERT esaj_detalhe_processos (OCR)
    OCR->>DB: INSERT esaj_calc_precatorio_resumo (cálculo)
    OCR->>LE: POST /reporte-email-cpf

    LE->>DB: SELECT vw_precatorios_full
    alt todos_processados = true
        LE-->>Cliente: Email laudo completo
        LE->>DB: UPDATE FINAL_REPORT_SENT
    else todos_processados = false
        LE-->>Cliente: WhatsApp "7 dias úteis"
        LE->>DB: UPDATE PARTIAL_REPORT_SENT
    end
```

---

## Tabelas Afetadas por Workflow

| Tabela | Chatbot | MP Unified | Laudo | Batch | Alertas |
|--------|---------|------------|-------|-------|---------|
| `consultas_esaj` | R/W | R/W | R/W | W | R/W |
| `process_tracking` | W | W | W | — | W |
| `esaj_detalhe_processos` | — | DELETE | R | — | — |
| `esaj_calc_precatorio_resumo` | — | DELETE | R | — | — |
| `logs` | — | W | — | — | W |

---

## Credenciais Utilizadas

| Credencial | Tipo | Usada em |
|---|---|---|
| `Postgres account` (b0F0gRzrpEq6BR3M) | PostgreSQL | Todos |
| `WhatsApp account` (ejhZtEKHF0Kh9HeQ) | WhatsApp API | Chatbot, MP, Alertas |
| `Mercado Pago API` (KytrAZe3o5ngsDTa) | HTTP Header Auth | MP Unified |
| `SMTP revisa` (QhA560IeJvTVywaS) | SMTP | Laudo, Alertas |
