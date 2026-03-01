IoT Incident Orchestrator

Sistema de orquestração de incidentes IoT baseado em eventos assíncronos, reconstrução determinística de estado e análise contextual assistida por IA.

📐 Arquitetura Geral

O sistema foi projetado com separação clara entre ingestão e processamento, seguindo princípios de desacoplamento e rastreabilidade.

Camadas:

Event Ingestion Layer

Incident Orchestrator (1-minute window processor)

🔹 1. Event Ingestion Layer

Responsável exclusivamente por receber eventos brutos e persistir no banco de dados.

Fluxo:

Webhook → Normalização → Persistência (events_raw)

Objetivo:

Garantir imutabilidade do evento original

Evitar aplicação prematura de regra de negócio

Permitir auditoria futura

🔹 2. Incident Orchestrator

Executado automaticamente a cada 1 minuto.

Fluxo:

Cron → Query eventos não processados →
Agrupamento por dispositivo →
Ordenação cronológica →
Reconstrução de estado →
Análise com IA →
Persistência de diagnóstico →
Atualização de estado consolidado →
Marcação de eventos processados

🧠 Estratégia de Classificação

O sistema combina:

1️⃣ Reconstrução determinística

Regras aplicadas:

INVASÃO → abertura sem destravamento prévio

VULNERABILIDADE → fechado mas destravado

2️⃣ Análise contextual com IA

A IA recebe:

Timeline ordenada

Estado final consolidado

Flags determinísticas

Resumo estruturado

Ela pode confirmar ou discordar das flags.

Isso garante:

Robustez lógica

Contextualização inteligente

Evita decisões exclusivamente heurísticas

⏱ Processamento em Janela

O processamento ocorre a cada 1 minuto.

Motivação:

Tolerância a eventos fora de ordem

Redução de decisões precipitadas

Consistência temporal

Balanceamento entre latência e precisão

🗃 Estrutura do Banco
events_raw

Armazena eventos brutos recebidos.

Campos principais:

device_id

timestamp

alarm_id

value

processed

created_at

Função:
Fonte de verdade imutável.

device_state

Armazena estado consolidado por dispositivo.

Campos:

device_id

is_closed

is_locked

updated_at

Função:
Consulta rápida do estado atual.

diagnostics

Armazena decisões finais da IA.

Campos:

device_id

type

is_incident

explanation

created_at

Função:
Histórico auditável de decisões.

🚀 Como Executar

Subir ambiente:

docker-compose up -d

Acessar:

http://localhost:5679

Porta alternativa utilizada para evitar conflito com instâncias locais de n8n.

🧪 Como Testar

Enviar evento para o webhook:

POST /webhook/events

Exemplo:

{
"device_id": "RS-CADEADO-01",
"timestamp": "2026-02-19T10:00:00Z",
"alarm_id": 32,
"value": 1
}

Aguardar até 1 minuto e verificar:

diagnostics

device_state

events_raw.processed = true

📌 Decisões Arquiteturais

Separação entre ingestão e processamento

Reconstrução de estado antes da IA

IA como camada de validação contextual

Persistência obrigatória

Processamento idempotente