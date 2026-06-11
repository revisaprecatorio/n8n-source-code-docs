# Workflow: Chatbot Revisa

**ID n8n:** `bXqi8RykpGxXMBGE`
**Trigger:** `POST /webhook/chatbot` (WhatsApp Cloud API)
**Status:** ✅ Ativo
**Descrição:** Ponto de entrada da plataforma. Recebe mensagens WhatsApp, gerencia estado da conversa e conduz o cliente desde a identificação por CPF até o pagamento do laudo.

---

## Nós (35 nós)

| Nó | Tipo | Função |
|---|---|---|
| `Webhook Trigger` | webhook | Recebe eventos WhatsApp + verificação Facebook |
| `Check Facebook Verification` | if | Separa challenge de verificação FB vs mensagem real |
| `Facebook Verification Response` | respondToWebhook | Responde `hub.challenge` ao Facebook |
| `Process Input` | code | Extrai `from`, `text`, `type` da mensagem |
| `Get User State` | postgres | SELECT `current_state` de `consultas_esaj` |
| `Merge State` | set | Une dados do webhook + estado do banco |
| `Route Message Type` | switch | Roteador central — 10 saídas por estado+mensagem |
| `Response NOT_TEXT` | set | Mensagem para mídia/áudio não suportados |
| `Consulta e-SAJ` | httpRequest | GET HTML por CPF no e-SAJ TJSP |
| `PT - Consulta Solicitada` | postgres | INSERT `process_tracking` (CONSULTA_SOLICITADA) |
| `If HTTP 200?` | if | Verifica se e-SAJ respondeu OK |
| `Parse e-SAJ Response` | code | Extrai nome e processos do HTML |
| `Save Consulta` | postgres | INSERT/UPDATE `consultas_esaj` |
| `Consulta Realizada` | postgres | INSERT `process_tracking` (CONSULTA_REALIZADA) |
| `Response CPF` | set | Monta resposta com lista de processos |
| `Generate Verification Code` | code | Gera código numérico 6 dígitos |
| `Send Verification Email` | emailSend | Envia código ao email informado |
| `Response EMAIL` | set | Monta resposta de confirmação de email |
| `Validate Code` | code | Valida código digitado pelo cliente |
| `Response CODE` | set | Resposta para código correto/errado |
| `Response CONFIRM_YES` | set | Resposta para confirmação positiva |
| `Response CONFIRM_NO` | set | Resposta para confirmação negativa |
| `Response MENU` | set | Resposta para "menu" |
| `Response INFO_PRECATORIOS` | set | Informações sobre precatórios |
| `Response CONSULTAR` | set | Início de nova consulta |
| `Response AGENT` | set | Resposta de agente/suporte |
| `Update State` | postgres | UPDATE `consultas_esaj.current_state` |
| `Trigger Payment Workflow` | httpRequest | POST `/generate-payment-link` ao Mercado Pago |
| `prepare response` | code | Prepara texto final da resposta WA |
| `Send WhatsApp Response` | whatsApp | Envia mensagem ao cliente |
| `Respond OK` | respondToWebhook | Responde 200 ao webhook do FB |
| `PT - TJSP Inoperante` | postgres | Registra indisponibilidade do e-SAJ |
| `Email Alerta TJSP` | emailSend | Notifica equipe: e-SAJ fora |
| `WA Alerta Operacao TJSP` | whatsApp | WA interno: e-SAJ fora |
| `Send TJSP Out` | whatsApp | WA ao cliente: sistema indisponível |

---

## Flowchart

```mermaid
flowchart TD
    WH([POST /webhook/chatbot\nWhatsApp Event]) --> CFV{Check Facebook\nVerification?}
    CFV -->|hub.challenge| FVR([Facebook Verification\nResponse])
    CFV -->|mensagem real| PI[Process Input\nextrai from + text + type]

    PI --> GUS[(Get User State\nconsultas_esaj\ncurrent_state)]
    GUS --> MS[Merge State\nwebhook + banco]
    MS --> RMT{Route Message Type\nSwitch\n10 saídas}

    RMT -->|0 NOT_TEXT| RNT[Response NOT_TEXT\nmídia não suportada]
    RMT -->|1 CPF recebido\nAWAITING_CPF| SAJ[Consulta e-SAJ\nGET HTML por CPF]
    RMT -->|2 email recebido\nAWAITING_EMAIL| GVC[Generate Verification Code\n6 dígitos]
    RMT -->|3 código recebido\nAWAITING_CODE| VC[Validate Code\ncompara hash]
    RMT -->|4 CONFIRM_YES| RCY[Response CONFIRM_YES]
    RMT -->|5 CONFIRM_NO| RCN[Response CONFIRM_NO]
    RMT -->|6 menu| RMN[Response MENU]
    RMT -->|7 info precatórios| RIP[Response INFO_PRECATORIOS]
    RMT -->|8 consultar| RCO[Response CONSULTAR]
    RMT -->|9 agente| RAG[Response AGENT]

    SAJ --> PCS[(PT - Consulta Solicitada\nprocess_tracking)]
    PCS --> HTTP{If HTTP 200?}
    HTTP -->|Sim| PAR[Parse e-SAJ Response\nnome + processos]
    HTTP -->|Não - e-SAJ fora| PTI[(PT - TJSP Inoperante)]
    PTI --> EAT[Email Alerta TJSP\nequipe]
    EAT --> WAT[WA Alerta Operacao\ninterno]
    WAT --> STO[Send TJSP Out\ncliente: indisponível]

    PAR --> SC[(Save Consulta\nconsultas_esaj)]
    SC --> CR[(Consulta Realizada\nprocess_tracking)]
    CR --> RCPF[Response CPF\nlista de processos]

    GVC --> SVE[Send Verification Email\ncódigo ao cliente]
    SVE --> REM[Response EMAIL\nconfirmação]

    RCPF --> US
    RNT --> US
    REM --> US
    VC --> US
    RCY --> US
    RCN --> US
    RMN --> US
    RIP --> US
    RCO --> US
    RAG --> US

    US[(Update State\nconsultas_esaj\ncurrent_state)] --> TPW[Trigger Payment Workflow\nPOST /generate-payment-link\nse AWAITING_CONFIRMATION→AWAITING_PAYMENT]
    US --> PR[prepare response\ntexto final WA]
    PR --> SWA[Send WhatsApp Response\nao cliente]
    SWA --> ROK([Respond OK 200])

    classDef entrada fill:#238636,color:#fff,stroke:#2ea043
    classDef wf      fill:#1f6feb,color:#fff,stroke:#388bfd
    classDef banco   fill:#0d419d,color:#fff,stroke:#1f6feb
    classDef sucesso fill:#1a7f37,color:#fff,stroke:#2da44e
    classDef alerta  fill:#da3633,color:#fff,stroke:#f85149
    classDef cinza   fill:#57606a,color:#fff,stroke:#6e7781
    classDef decisao fill:#bf8700,color:#fff,stroke:#d29922

    class WH,WAT,STO,SWA entrada
    class CFV,RMT,HTTP decisao
    class FVR,ROK sucesso
    class GUS,SC,CR,US,PCS banco
    class SAJ,GVC,SVE,PAR,VC,TPW wf
    class PTI,EAT alerta
    class PI,MS,PR,RNT,RCY,RCN,RMN,RIP,RCO,RAG,REM,RCPF cinza
```

---

## Diagrama de Sequência — Conversa Completa

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'actorBkg': '#1f6feb', 'actorTextColor': '#ffffff', 'actorBorder': '#388bfd', 'actorLineColor': '#8b949e', 'signalColor': '#8b949e', 'signalTextColor': '#c9d1d9', 'labelBoxBkgColor': '#21262d', 'labelBoxBorderColor': '#30363d', 'labelTextColor': '#c9d1d9', 'loopTextColor': '#c9d1d9', 'noteBkgColor': '#21262d', 'noteTextColor': '#c9d1d9', 'activationBkgColor': '#388bfd', 'activationBorderColor': '#58a6ff'}}}%%
sequenceDiagram
    actor CLI as Cliente (WhatsApp)
    participant WF as Chatbot Revisa
    participant DB as consultas_esaj
    participant SAJ as e-SAJ TJSP
    participant EM as Email (SMTP)
    participant MP as Mercado Pago Unified

    CLI->>WF: Qualquer mensagem inicial
    WF->>DB: SELECT current_state WHERE whatsapp_from=...
    DB-->>WF: NULL (novo cliente) → state=IDLE
    WF-->>CLI: "Olá! Para começar, informe seu CPF"
    WF->>DB: UPDATE current_state=AWAITING_CPF

    CLI->>WF: "123.456.789-01"
    WF->>DB: SELECT current_state → AWAITING_CPF
    WF->>SAJ: GET /cpopg/search.do?cpf=12345678901
    WF->>DB: INSERT process_tracking (CONSULTA_SOLICITADA)

    alt e-SAJ disponível
        SAJ-->>WF: HTML com processos
        WF->>WF: Parse: nome + lista de processos CNJ
        WF->>DB: INSERT/UPDATE consultas_esaj\n(nome, processos, total_processos)
        WF->>DB: INSERT process_tracking (CONSULTA_REALIZADA)
        WF-->>CLI: "Encontrei X precatório(s):\n[lista]\nInforme seu email para continuar"
        WF->>DB: UPDATE AWAITING_EMAIL
    else e-SAJ indisponível
        WF->>DB: INSERT process_tracking (TJSP_INOPERANTE)
        WF->>EM: Alerta equipe: e-SAJ fora
        WF-->>CLI: "Sistema TJSP temporariamente indisponível"
    end

    CLI->>WF: "email@exemplo.com.br"
    WF->>WF: Gera código verificação 6 dígitos
    WF->>EM: Envia código para email@exemplo.com.br
    WF-->>CLI: "Código enviado para seu email. Digite o código:"
    WF->>DB: UPDATE AWAITING_CODE (salva código hash)

    CLI->>WF: "123456"
    WF->>WF: Valida código
    WF-->>CLI: "✅ Email verificado!\nDeseja confirmar a consulta? (Sim/Não)"
    WF->>DB: UPDATE AWAITING_CONFIRMATION

    CLI->>WF: "Sim"
    WF->>MP: POST /generate-payment-link\n{ whatsapp_from, cpf, email, trigger_payment: true }
    MP-->>CLI: "💳 Link de pagamento: https://..."
    WF->>DB: UPDATE AWAITING_PAYMENT
```

---

## Máquina de Estados

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1f6feb', 'primaryTextColor': '#ffffff', 'primaryBorderColor': '#388bfd', 'lineColor': '#8b949e', 'background': '#0d1117', 'secondaryColor': '#21262d', 'tertiaryColor': '#161b22', 'labelColor': '#c9d1d9'}}}%%
stateDiagram-v2
    [*] --> IDLE : sem registro

    IDLE --> AWAITING_CPF : mensagem recebida
    AWAITING_CPF --> AWAITING_EMAIL : CPF válido\ne-SAJ consultado\nprocessos encontrados
    AWAITING_CPF --> IDLE : e-SAJ indisponível

    AWAITING_EMAIL --> AWAITING_CODE : email recebido\ncódigo enviado

    AWAITING_CODE --> IDLE : timeout
    AWAITING_CODE --> AWAITING_CONFIRMATION : código correto\nconsulta confirmada

    AWAITING_CONFIRMATION --> AWAITING_PAYMENT : cliente confirmou\n/generate-payment-link chamado

    AWAITING_PAYMENT --> PAYMENT_APPROVED : MP webhook approved
    AWAITING_PAYMENT --> PAYMENT_REJECTED : MP webhook rejected
    AWAITING_PAYMENT --> IDLE : timeout 60min

    PAYMENT_APPROVED --> PROCESSING : watchdog Windows Server\nFOR UPDATE SKIP LOCKED
    PROCESSING --> FINAL_REPORT_SENT : todos processados
    PROCESSING --> PARTIAL_REPORT_SENT : processamento parcial
    PROCESSING --> MANUAL_PROCESS : OCR falhou
    PROCESSING --> PIPELINE_ERROR : crash no pipeline
    PROCESSING --> AUTH_ERROR : login e-SAJ falhou
    PROCESSING --> DOWNLOAD_FAILED : PDFs não baixados

    MANUAL_PROCESS --> ALERTA_MANUAL_SENT : Alerta_ERROS_GRAVES ou\nAlerta_Reporte_Manual
    PIPELINE_ERROR --> ALERTA_MANUAL_SENT : Alerta_ERROS_GRAVES
```

---

## Roteador Central (Route Message Type — Switch)

| Saída | Condição | Estado Atual → Próximo |
|---|---|---|
| 0 | `type != text` | qualquer → mantém |
| 1 | CPF válido recebido | `AWAITING_CPF` → `AWAITING_EMAIL` |
| 2 | Email recebido | `AWAITING_EMAIL` → `AWAITING_CODE` |
| 3 | Código numérico recebido | `AWAITING_CODE` → `AWAITING_CONFIRMATION` |
| 4 | "sim" / confirmação | `AWAITING_CONFIRMATION` → `AWAITING_PAYMENT` |
| 5 | "não" / negação | `AWAITING_CONFIRMATION` → `IDLE` |
| 6 | "menu" | qualquer → `IDLE` |
| 7 | "precatório" / info | qualquer → mantém |
| 8 | "consultar" | qualquer → `AWAITING_CPF` |
| 9 | qualquer outro | qualquer → mantém |

---

## Tabelas Afetadas

| Tabela | Operação |
|---|---|
| `consultas_esaj` | INSERT (novo cliente) + UPDATE (state a cada mensagem) |
| `process_tracking` | INSERT (CONSULTA_SOLICITADA, CONSULTA_REALIZADA, TJSP_INOPERANTE) |

---

## Notas Importantes

- **Verificação Facebook:** O webhook do WhatsApp Cloud API também recebe um `GET` com `hub.challenge` para verificação. O nó `Check Facebook Verification` separa esse caso e responde diretamente com o challenge sem processar como mensagem.
- **Trigger Payment:** O `Trigger Payment Workflow` só é chamado quando `current_state` transita para `AWAITING_PAYMENT` — não em todas as atualizações de estado.
- **e-SAJ indisponível:** Se o e-SAJ retornar status != 200, o workflow notifica a equipe internamente e responde ao cliente com mensagem de indisponibilidade, sem fazer retry automático.
