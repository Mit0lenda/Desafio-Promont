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

### Prioridade de Classificação

Caso múltiplas condições ocorram simultaneamente:

1. **INVASAO** tem prioridade máxima
2. **VULNERABILIDADE** é considerada apenas se não houver INVASAO
3. Caso contrário, **NORMAL**

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

A reconstrução de estado e a detecção inicial de inconsistências são realizadas por **regras determinísticas**, garantindo consistência física e previsibilidade na análise dos eventos.

A IA atua como **mecanismo de julgamento contextual** do cenário consolidado, sendo responsável por:

- Classificar o tipo final do evento (`INVASAO | VULNERABILIDADE | NORMAL`)
- Determinar se a situação exige notificação de segurança
- Gerar justificativa técnica estruturada
- Informar nível de confiança da decisão

A **decisão persistida no sistema é baseada na resposta estruturada da IA**.

A IA não substitui as regras determinísticas de reconstrução de estado, mas complementa a análise com interpretação contextual.

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

## Estrutura do Banco de Dados

### events_raw

Armazena os eventos brutos recebidos via webhook. Representa a **fonte de verdade imutável** do sistema.

**Campos:**

| Campo | Tipo | Descrição |
|-------|------|----------|
| `id` | uuid | Identificador único (PK) |
| `device_id` | text | ID do dispositivo |
| `timestamp` | timestamptz | Timestamp do evento (ordem real) |
| `alarm_id` | integer | Tipo de sensor (32 ou 33) |
| `value` | integer | Estado do sensor (0 ou 1) |
| `processed` | boolean | Flag de processamento |
| `created_at` | timestamptz | Data de recebimento |

**DDL:**
```sql
create table public.events_raw (
  id uuid not null default gen_random_uuid (),
  device_id text null,
  timestamp timestamp with time zone null,
  alarm_id integer null,
  value integer null,
  processed boolean null default false,
  created_at timestamp with time zone null default now(),
  constraint events_raw_pkey primary key (id)
);
```

---

### device_state

Mantém o **estado consolidado atual** do dispositivo (upsert por `device_id`).

**Campos:**

| Campo | Tipo | Descrição |
|-------|------|----------|
| `device_id` | text | ID do dispositivo (PK) |
| `is_closed` | boolean | Se o cadeado está fechado |
| `is_locked` | boolean | Se o cadeado está travado |
| `updated_at` | timestamptz | Última atualização |

**DDL:**
```sql
create table public.device_state (
  device_id text not null,
  is_closed boolean null,
  is_locked boolean null,
  updated_at timestamp with time zone null,
  constraint device_state_pkey primary key (device_id),
  constraint device_state_device_id_key unique (device_id)
);
```

---

### diagnostics

Histórico das decisões geradas pela IA para **auditoria e rastreabilidade**.

**Campos:**

| Campo | Tipo | Descrição |
|-------|------|----------|
| `id` | uuid | Identificador único (PK) |
| `device_id` | text | ID do dispositivo analisado |
| `type` | text | Tipo de classificação (INVASAO, VULNERABILIDADE, NORMAL) |
| `is_incident` | boolean | Se é incidente |
| `explanation` | text | Justificativa técnica da IA |
| `created_at` | timestamptz | Data da análise |

**DDL:**
```sql
create table public.diagnostics (
  id uuid not null default gen_random_uuid (),
  device_id text null,
  type text null,
  is_incident boolean null,
  explanation text null,
  created_at timestamp with time zone null default now(),
  constraint diagnostics_pkey primary key (id)
);
```

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
