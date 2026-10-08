# AiOrchestrator

Laboratório de arquitetura distribuída em C#/.NET 10 no Windows, **sem Docker**, com mensageria real em RabbitMQ. O objetivo é orquestrar tarefas entre modelos de IA: o Claude analisa, o ChatGPT revisa e o sistema consolida.

- Especificação original: [`docs/AI-Orchestrator-Claude-Plan.md`](docs/AI-Orchestrator-Claude-Plan.md)
- Decisões de design das Fases 0–2: [`docs/superpowers/specs/2026-10-08-fases-0-2-arquitetura-design.md`](docs/superpowers/specs/2026-10-08-fases-0-2-arquitetura-design.md)

## Sumário

1. [Status](#1-status)
2. [Regras do projeto](#2-regras-do-projeto)
3. [Estrutura](#3-estrutura)
4. [Arquitetura](#4-arquitetura)
5. [Contratos de mensagem](#5-contratos-de-mensagem)
6. [Executar](#6-executar)

---

## 1. Status

| Fase | Descrição | Situação |
|---|---|---|
| 0 | Diagnóstico do ambiente | ✅ concluída |
| 1 | Instalação de Erlang + RabbitMQ | ⏳ pendente (requer Administrador) |
| 2 | API + Worker mínimos (modo simulado) | 📝 arquitetura definida |
| 3 | Robustez: DLQ, retentativas com backoff, deduplicação | 🔜 |
| 4–5 | Integração com Anthropic e OpenAI (`IAiProvider`) | 🔜 |
| 6 | Orchestrator Worker, PostgreSQL, Outbox | 🔜 |
| 7 | Angular, SignalR, autenticação, health checks | 🔜 |

## 2. Regras do projeto

- Sem Docker nem containers.
- Nenhuma chamada paga a provedores de IA sem confirmação explícita; o modo simulado é o padrão.
- Segredos só em User Secrets ou variáveis de ambiente, nunca no Git, em logs ou em mensagens.
- A saída dos modelos é tratada como conteúdo não confiável: nunca é executada automaticamente.
- O broker escuta só localmente; as portas não são expostas.

## 3. Estrutura

```text
AiOrchestrator/
├── backend/                                 # .NET 10 — ferramenta: dotnet
│   ├── AiOrchestrator.sln
│   ├── src/
│   │   ├── AiOrchestrator.Contracts/        # eventos, enums, routing keys — sem dependências
│   │   ├── AiOrchestrator.Messaging/        # infraestrutura RabbitMQ — depende de Contracts
│   │   ├── AiOrchestrator.Api/              # API HTTP + consumidor de resultados — depende de Messaging
│   │   └── AiOrchestrator.Worker/           # processamento simulado — depende de Messaging
│   └── tests/
│       ├── AiOrchestrator.UnitTests/        # sem dependências externas
│       └── AiOrchestrator.IntegrationTests/ # RabbitMQ local, vhost aiorchestrator-tests
├── frontend/                                # Angular (Fase 7) — ferramenta: npm / ng
├── docs/                                    # especificação original e specs de design
├── .gitignore
└── README.md
```

Monorepo: backend e frontend ficam no mesmo repositório, cada um com suas próprias ferramentas. Eles se comunicam **apenas por HTTP e SignalR**: o Angular não referencia nada do .NET. Em desenvolvimento, o `ng serve` usa um proxy para `/api` e `/hubs` e dispensa CORS. O cliente TypeScript poderá ser gerado a partir do OpenAPI da API.

| Projeto | Responsabilidade |
|---|---|
| **Contracts** | Records imutáveis `AnalysisRequestedV1` e `AnalysisCompletedV1`, enums `ExecutionMode` e `AnalysisStatus`, classe `RoutingKeys`. Não referencia o RabbitMQ. |
| **Messaging** | `RabbitMQ.Client` 7.x: `RabbitMqOptions`, conexão singleton com recuperação automática, `IMessagePublisher` com publisher confirms, `RabbitMqConsumerBase<T>` com ACK manual, topologia idempotente, JSON. |
| **Api** | `POST /api/tasks`, `GET /api/tasks/{id}`, `ITaskStore` em memória e `AnalysisCompletedConsumer`. |
| **Worker** | `AnalysisRequestedConsumer`: gera um resultado simulado e publica `AnalysisCompleted.v1`. Nunca chama provedores de IA nesta fase. |
| **UnitTests** | Serialização dos contratos, validação da API, transições do store, lógica do Worker. |
| **IntegrationTests** | Publicação e consumo reais. Filas com nome único por teste. Os testes são **pulados** se o broker não estiver acessível. |

API e Worker **não** se referenciam: o único acoplamento entre eles são os contratos de mensagem.

## 4. Arquitetura

### Fluxo (Fase 2)

```text
Cliente ── POST /api/tasks ──► Api ──(analysis.requested.v1)──► RabbitMQ ──► Worker
Cliente ◄─ GET /api/tasks/{id} ─ Api ◄─(analysis.completed.v1)── RabbitMQ ◄── Worker
```

1. `POST /api/tasks` valida a entrada: apenas `Mode = Simulation` (`Live` → `400`) e `Input` dentro do limite de tamanho.
2. A API gera `TaskId` e `CorrelationId` e grava a tarefa como `Pending` no store em memória.
3. A API publica `AnalysisRequested.v1` e aguarda o publisher confirm.
   - Se a publicação falhar, a tarefa vira `Failed` e a resposta é `503`.
4. A API responde `202 Accepted` com `Location: /api/tasks/{id}`.
5. O Worker consome a mensagem, gera o resultado simulado, publica `AnalysisCompleted.v1` e faz ACK.
6. O consumidor da API muda a tarefa para `Completed` ou `Failed`.
   - Uma tarefa já finalizada não é alterada (idempotente por `TaskId`).
7. `GET /api/tasks/{id}` devolve o estado atual.

Estados: `Pending → Completed | Failed`.

### Topologia RabbitMQ

| Elemento | Nome | Detalhes |
|---|---|---|
| Virtual host | `aiorchestrator` | aplicação |
| Virtual host | `aiorchestrator-tests` | testes de integração |
| Exchange | `aiorchestrator.events` | `topic`, durável |
| Fila | `worker.analysis-requested` | durável, binding `analysis.requested.v1` |
| Fila | `api.analysis-completed` | durável, binding `analysis.completed.v1` |

Cada processo declara, de forma idempotente ao iniciar, o exchange e as filas que consome. A aplicação usa um usuário próprio, sem privilégios administrativos; a conta `guest` não é usada.

### Rastreabilidade

Toda mensagem leva `MessageId`, `TaskId` e `CorrelationId`, no corpo e nas propriedades AMQP. API e Worker abrem um escopo de log com esses campos, o que permite seguir uma tarefa de ponta a ponta pelo `CorrelationId`.

### Garantias e limitações (Fase 2)

| Situação | Comportamento |
|---|---|
| Entrega | Pelo menos uma vez. Uma entrega repetida é inofensiva. |
| Mensagem inválida | `BasicNack(requeue: false)` + log. **Descartada**, pois ainda não há DLQ. |
| Exceção no Worker | O Worker publica `AnalysisCompleted.v1` com `Status = Failed`. Se essa publicação também falhar, faz NACK e a tarefa fica `Pending`. |
| Reinício da API | O store em memória é perdido. |
| Retentativas / backoff | Ainda não existem (Fase 3). |

## 5. Contratos de mensagem

Todos os eventos são publicados em `aiorchestrator.events`. A routing key é o nome do evento em minúsculas, seguido da versão.

**Regras**
- Uma mudança incompatível gera um novo evento `.v2` com uma nova routing key.
- Um novo campo opcional não muda a versão; os consumidores ignoram campos desconhecidos.
- JSON (`System.Text.Json`), camelCase, enums como string, datas em UTC (ISO 8601).
- Proibido incluir chaves, credenciais, prompts de sistema ou dados sensíveis.

**Propriedades AMQP:** `message_id` = MessageId, `correlation_id` = CorrelationId, `type` = routing key, `content_type` = `application/json`, `delivery_mode` = 2 (persistente), `timestamp` = CreatedAtUtc.

**Envelope comum:** `messageId` (Guid), `taskId` (Guid), `correlationId` (Guid), `createdAtUtc` (DateTimeOffset), `schemaVersion` (int), `mode` (`Simulation` | `Live`).

### `AnalysisRequested.v1`

- **Routing key:** `analysis.requested.v1`
- **Fluxo:** Api → Worker

| Campo | Tipo | Descrição |
|---|---|---|
| *envelope* | | |
| `input` | `string` | Instrução do usuário, com tamanho limitado |

```json
{
  "messageId": "6f1c2a9e-1d2b-4a8e-9a77-1f3c5b2e8d10",
  "taskId": "0b7e8c44-5c1a-4f3e-8d2b-9e6a1c7f4b21",
  "correlationId": "c2d4e6f8-1a3b-4c5d-8e7f-9a0b1c2d3e4f",
  "createdAtUtc": "2026-10-08T14:00:00Z",
  "schemaVersion": 1,
  "mode": "Simulation",
  "input": "Explique o padrão Outbox."
}
```

### `AnalysisCompleted.v1`

- **Routing key:** `analysis.completed.v1`
- **Fluxo:** Worker → Api

| Campo | Tipo | Descrição |
|---|---|---|
| *envelope* | | Mesmos `taskId` e `correlationId` da solicitação; `messageId` novo |
| `status` | `Completed` \| `Failed` | |
| `finalResult` | `string?` | Presente quando `Completed` |
| `errorCode` | `string?` | Presente quando `Failed`; sem detalhes sensíveis |

```json
{
  "messageId": "a9b8c7d6-e5f4-4321-8765-0fedcba98765",
  "taskId": "0b7e8c44-5c1a-4f3e-8d2b-9e6a1c7f4b21",
  "correlationId": "c2d4e6f8-1a3b-4c5d-8e7f-9a0b1c2d3e4f",
  "createdAtUtc": "2026-10-08T14:00:02Z",
  "schemaVersion": 1,
  "mode": "Simulation",
  "status": "Completed",
  "finalResult": "[SIMULAÇÃO] Análise gerada sem chamar provedores de IA.",
  "errorCode": null
}
```

**Mensagens inválidas** (JSON malformado, campo obrigatório ausente, `schemaVersion` não suportado) recebem NACK sem requeue e são registradas em log. Na Fase 3 passarão a ir para a DLQ.

**Eventos planejados:** `claude.analysis.requested.v1`, `claude.analysis.completed.v1`, `openai.review.requested.v1`, `openai.review.completed.v1`. Todos usarão o mesmo envelope.

## 6. Executar

> Será completado quando os projetos forem criados (Fase 2).

```powershell
dotnet build backend
dotnet run --project backend/src/AiOrchestrator.Api      # terminal 1
dotnet run --project backend/src/AiOrchestrator.Worker   # terminal 2
dotnet test backend
```
