# 🔐 IoT Incident Orchestrator

![n8n](https://img.shields.io/badge/n8n-latest-EA4B71?style=flat-square) ![Docker](https://img.shields.io/badge/Docker-required-2496ED?style=flat-square) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supabase-3ECF8E?style=flat-square) ![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4-412991?style=flat-square)

Orquestrador de incidentes IoT desenvolvido para processar eventos de cadeados inteligentes em ambientes com conectividade intermitente.

**Implementação alinhada ao desafio técnico Promont.**

---

## Objetivo do Sistema

Processar eventos assíncronos provenientes de dispositivos IoT e determinar, com base na sequência real dos eventos, se houve:

- **INVASÃO**
- **VULNERABILIDADE**
- **OPERAÇÃO NORMAL**

A solução considera explicitamente:

- Eventos atrasados
- Eventos fragmentados
- Timestamps fora de ordem
- Múltiplos dispositivos simultaneamente

---

## Capacidades Implementadas

| Requisito | Status |
|-----------|--------|
| Receber eventos fora de ordem | ✅ |
| Reconstruir linha do tempo por dispositivo | ✅ |
| Identificar inconsistências de segurança | ✅ |
| Executar análise contextual a cada 1 minuto | ✅ |
| Persistir diagnóstico e estado consolidado | ✅ |
| Notificar incidentes automaticamente | ✅ |

---

## Modelo do Dispositivo

O cadeado possui dois sensores:

| Alarm ID | Função | Valor 0 | Valor 1 |
|----------|--------|---------|---------|
| **32** | Fechamento | Fechado | Aberto |
| **33** | Travamento | Travado | Destravado |

### Regras de Segurança

#### Invasão
Cadeado **aberto** (`32 = 1`) sem que tenha ocorrido **destravamento** (`33 = 1`) previamente.

#### Vulnerabilidade
Cadeado **fechado** (`32 = 0`) enquanto permanece **destravado** (`33 = 1`).

---

## Arquitetura

A solução foi implementada com **separação clara entre ingestão e processamento**.

```
┌─────────────────────────────┐
│   1. INGESTÃO (Webhook)     │
│   POST /webhook/iot-events  │
├─────────────────────────────┤
│ • Armazena eventos          │
│ • Não assume ordem          │
│ • Não executa lógica        │
└──────────┬──────────────────┘
           │
           ↓ [events_raw] processed=false
           
┌──────────┴──────────────────┐
│   2. PROCESSADOR (CRON)     │
│   Trigger: */1 * * * *      │
├─────────────────────────────┤
│ 1. Buscar não processados   │
│ 2. Agrupar por device_id    │
│ 3. Ordenar por timestamp    │
│ 4. Reconstruir estado       │
│ 5. Detectar inconsistências │
│ 6. Validar com IA           │
│ 7. Persistir diagnóstico    │
│ 8. Atualizar estado         │
│ 9. Marcar como processado   │
└──────────┬──────────────────┘
           │
           ↓
      [diagnostics]
      [device_state]
      [Telegram]
```

---

## 1. Ingestão (Webhook)

**Endpoint:** `POST /webhook/iot-events`

- Armazena todos os eventos recebidos
- Não assume ordem correta
- Não executa lógica de decisão

Os eventos são persistidos na tabela `events_raw` com `processed = false`.

---

## 2. Processador (Execução a cada 1 minuto)

**Trigger CRON:**
```
*/1 * * * *
```

### Fluxo

1. Busca eventos não processados
2. Agrupa por `device_id`
3. Ordena por `timestamp`
4. Reconstrói estado sequencialmente
5. Detecta inconsistências determinísticas
6. Executa validação contextual com IA
7. Persiste diagnóstico
8. Atualiza estado consolidado
9. Marca eventos como processados

---

## Reconstrução de Estado

Para cada dispositivo:

**Estado inicial assumido como seguro:**
```javascript
{
  aberto: false,
  destravado: false
}
```

A timeline é percorrida **exclusivamente em ordem cronológica real**.

Isso garante consistência mesmo quando:
- Eventos chegam fora de ordem
- Eventos são enviados em múltiplos pacotes
- Há atraso de transmissão

---

## Uso da IA

A IA atua como **camada adicional de validação**.

### Entrada fornecida:
- Timeline ordenada
- Estado final consolidado
- Flags determinísticas
- Resumo estruturado

### Saída esperada:
```json
{
  "incident": true,
  "type": "INVASAO | VULNERABILIDADE | NORMAL",
  "confidence": 0.98,
  "justification": "string",
  "requires_notification": true
}
```

**Importante:** A IA não substitui as regras determinísticas. Ela valida coerência contextual e reforça a decisão.

---

## Persistência

Conforme exigido no desafio, são persistidos:

### `diagnostics`
- `device_id`
- `type`
- `is_incident`
- `justification`
- `created_at`

### `device_state`
Estado consolidado atual do dispositivo (upsert por `device_id`).

Todos os eventos processados são marcados como `processed = true`, garantindo **idempotência**.

---

## Stack Técnica

| Componente | Tecnologia |
|------------|------------|
| Orquestração | n8n |
| Containerização | Docker |
| Banco de Dados | Supabase (PostgreSQL) |
| IA | OpenAI (GPT-4) |
| Notificações | Telegram Bot |

---

## Execução

### Pré-requisitos

- Docker
- Credenciais configuradas:
  - Supabase
  - OpenAI
  - Telegram

### Inicialização

```bash
docker-compose up -d
```

Acessar: `http://localhost:5679`

### Importar Workflows

1. `Ingestão de eventos IOT.json`
2. `Processador de Eventos IoT.json`

Configurar credenciais e ativar workflows.

---

## Teste

### Exemplo de envio:

```bash
curl -X POST http://localhost:5679/webhook/iot-events \
  -H "Content-Type: application/json" \
  -d '[{
    "device_id": "RS-CADEADO-01",
    "timeline": [
      { "timestamp": "2026-02-19T10:06:00Z", "alarm_id": 32, "value": 1 },
      { "timestamp": "2026-02-19T10:02:00Z", "alarm_id": 33, "value": 0 },
      { "timestamp": "2026-02-19T10:05:00Z", "alarm_id": 32, "value": 0 }
    ]
  }]'
```

**Aguardar até 1 minuto para processamento.**

### Verificação

Consultar tabelas:
- `events_raw` - Eventos recebidos
- `diagnostics` - Análises realizadas
- `device_state` - Estado consolidado

---

## Características Técnicas

- ✅ Processamento idempotente
- ✅ Reconstrução determinística de estado
- ✅ Separação ingestão / análise
- ✅ Persistência auditável
- ✅ Suporte a múltiplos dispositivos simultaneamente
- ✅ Resiliência a eventos fora de ordem

---

## Critérios de Avaliação Atendidos

| Critério | Implementação |
|----------|---------------|
| **Engenharia de Dados** | Agrupamento, ordenação e consolidação de eventos fragmentados |
| **Lógica de Estado** | Identificação correta de INVASÃO e VULNERABILIDADE |
| **Uso de IA** | Validação contextual com prompt estruturado |
| **Persistência** | Auditoria completa em `diagnostics` e `device_state` |

---

## Estrutura do Repositório

```
.
├── docker-compose.yml
├── README.md
├── Ingestão de eventos IOT.json
└── Processador de Eventos IoT.json
```

---

## Conclusão

O sistema demonstra:

- Engenharia de dados para eventos assíncronos
- Modelagem clara de regras de negócio
- Uso controlado de IA como camada de decisão
- Persistência completa para auditoria
- Arquitetura desacoplada e resiliente

---

**Desenvolvido como resposta ao desafio técnico Promont.**
