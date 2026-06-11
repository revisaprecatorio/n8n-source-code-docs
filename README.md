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
flowchart LR
    subgraph PRINCIPAL["Fluxo do Cliente"]
        direction TB
        WA(["Cliente\nWhatsApp"])
        CW["1. Chatbot Revisa\nwebhook/chatbot"]
        MP["2. Mercado Pago\nwebhook/generate-payment-link\nwebhook/mercadopago-notification"]
        DB[("consultas_esaj\nPAYMENT_APPROVED")]
        CR["Crawler TJSP\nSelenium + Chrome"]
        OCR["pipeline_completo.sh\nOCR + Ingestão + Cálculo"]
        LE["3. Laudo envio email+cpf\nwebhook/reporte-email-cpf"]
        EMAIL(["Email ao cliente\nFINAL_REPORT_SENT"])
        PARCIAL(["Email Revisa + WA\nPARTIAL_REPORT_SENT"])

        WA -->|CPF| CW
        CW <-->|"email + confirmação"| WA
        CW -->|"trigger_payment=true"| MP
        MP -->|"link de pagamento"| WA
        WA -->|paga| MP
        MP -->|"PAYMENT_APPROVED\n+ DELETE OCR antigo"| DB
        DB -->|"watchdog"| CR
        CR -->|"PDFs baixados"| OCR
        OCR -->|"POST /reporte-email-cpf"| LE
        OCR -->|"100% rejeitados — Etapa 9b"| LE
        LE -->|"todos_processados=true"| EMAIL
        LE -->|"todos_processados=false"| PARCIAL
    end

    subgraph LATERAL["Admin e Monitoramento"]
        direction TB
        ADMIN(["Admin / Sistema"])
        BP["4. CPF_batch_processing\nwebhook/cpf-batch-processing"]
        ADMIN -->|"POST cpf"| BP

        SCHED(["Schedule 10min"])
        AG["5. Alertas\nERROS_GRAVES\nLaudo_Parcial\nReporte_Manual"]
        NOTIF(["WA + Email\ncliente + equipe"])
        SCHED --> AG
        AG --> NOTIF
    end

    BP -->|"PAYMENT_APPROVED direto"| DB

    classDef verde fill:#1a7f37,color:#fff,stroke:#1a7f37
    classDef azul fill:#0969da,color:#fff,stroke:#0969da
    classDef laranja fill:#bc4c00,color:#fff,stroke:#bc4c00
    classDef cinza fill:#57606a,color:#fff,stroke:#57606a
    classDef roxo fill:#6639ba,color:#fff,stroke:#6639ba

    class WA,ADMIN verde
    class EMAIL,MP azul
    class PARCIAL laranja
    class SCHED,NOTIF cinza
    class OCR roxo
```

---

## Diagrama de Sequência — Jornada do Cliente

```mermaid
sequenceDiagram
    actor CLI as Cliente (WhatsApp)
    participant N8N as n8n (Chatbot + MP)
    participant DB as PostgreSQL
    participant WIN as Windows Server
    participant EM as Email (SMTP)

    CLI->>N8N: CPF via WhatsApp
    N8N->>DB: INSERT consultas_esaj (AWAITING_CPF)
    N8N->>DB: Consulta e-SAJ → UPDATE (AWAITING_EMAIL)
    N8N-->>CLI: "Informe seu email"

    CLI->>N8N: email@exemplo.com
    N8N->>EM: Envia código de verificação
    N8N->>DB: UPDATE (AWAITING_CODE)
    N8N-->>CLI: "Código enviado ao seu email"

    CLI->>N8N: código 6 dígitos
    N8N->>DB: UPDATE (AWAITING_CONFIRMATION)
    N8N-->>CLI: "Processos encontrados. Confirma?"

    CLI->>N8N: "Sim"
    N8N->>N8N: POST /generate-payment-link → Mercado Pago API
    N8N->>DB: UPDATE (AWAITING_PAYMENT)
    N8N-->>CLI: "Link de pagamento: https://..."

    CLI->>N8N: Mercado Pago webhook (approved)
    N8N-->>N8N: Responde 200 OK imediato ao MP
    N8N->>DB: UPDATE (PAYMENT_APPROVED)
    N8N->>DB: DELETE esaj_detalhe_processos + esaj_calc_precatorio_resumo
    N8N-->>CLI: "Pagamento confirmado! Laudo em até 24h"

    WIN->>WIN: watchdog detecta PAYMENT_APPROVED
    WIN->>WIN: Crawler baixa PDFs → pipeline_completo.sh
    WIN->>DB: INSERT esaj_detalhe_processos + esaj_calc_precatorio_resumo
    WIN->>N8N: POST /reporte-email-cpf

    alt todos_processados = true
        N8N->>DB: SELECT vw_precatorios_full
        N8N->>EM: Envia laudo completo ao cliente
        N8N->>DB: UPDATE FINAL_REPORT_SENT
        EM-->>CLI: Email "Laudo Diagnóstico de Precatório"
    else todos_processados = false
        N8N->>EM: Envia alerta interno (equipe)
        N8N-->>CLI: WhatsApp "laudo em 7 dias úteis"
        N8N->>DB: UPDATE PARTIAL_REPORT_SENT
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
