# AI Orchestrator — Fases 0 a 2: Arquitetura

> Spec de design derivada de `docs/AI-Orchestrator-Claude-Plan.md`.
> Escopo: **somente a arquitetura** do primeiro marco (Fases 0–2). Fases 3–7 terão specs próprias.

## 1. Objetivo

Validar mensageria real com RabbitMQ entre dois processos .NET independentes (API e Worker), no Windows, sem Docker, em modo **simulado** (nenhuma chamada paga a provedores de IA). Os critérios de aceite são os da §9 do documento original.

## 2. Decisões

| # | Decisão | Motivo |
|---|---|---|
| D1 | O resultado volta à API **por mensageria**: o Worker publica `analysis.completed.v1` e um consumidor dentro da API atualiza o store em memória. | Mantém API e Worker desacoplados e pratica consumo nos dois sentidos. |
| D2 | Cliente **`RabbitMQ.Client` 7.x** (oficial, API assíncrona), sem MassTransit/EasyNetQ. | Objetivo didático: topologia, ACK/NACK e conexões explícitos. |
| D3 | **Um exchange `topic` durável** `aiorchestrator.events`; routing key = nome do evento versionado; uma fila durável por consumidor. | Ensina routing keys/bindings e comporta os eventos das fases seguintes sem mudar a topologia. |
| D4 | Projeto extra **`AiOrchestrator.Messaging`** (desvio da estrutura da §7). | Evita duplicar a infraestrutura RabbitMQ na API e no Worker e mantém `Contracts` sem dependências. |
| D5 | Na Fase 2 a API **aceita apenas `Mode = Simulation`**; `Live` → `400 Bad Request`. | Garante que nenhuma requisição HTTP dispare custo. |
| D6 | Testes de integração rodam contra o RabbitMQ local no vhost `aiorchestrator-tests` e são **pulados** (com mensagem explícita) se o broker não estiver acessível. | Sem Docker não há Testcontainers; testes unitários continuam rodando em qualquer máquina. |

## 3. Estrutura da solução

```text
AiOrchestrator/
├── backend/
│   ├── AiOrchestrator.sln
│   ├── src/
│   │   ├── AiOrchestrator.Contracts/     # records dos eventos + constantes de routing key; sem dependências externas
│   │   ├── AiOrchestrator.Messaging/     # conexão, publisher, consumer base, topologia, serialização JSON
│   │   ├── AiOrchestrator.Api/           # ASP.NET Core Web API + consumidor de analysis.completed.v1
│   │   └── AiOrchestrator.Worker/        # Worker Service: consome analysis.requested.v1, simula, publica analysis.completed.v1
│   └── tests/
│       ├── AiOrchestrator.UnitTests/
│       └── AiOrchestrator.IntegrationTests/
├── frontend/                          # Angular — vazio até a Fase 7
├── docs/                              # especificação original e specs de design
└── README.md                          # documentação única do projeto
```

**Monorepo (D7):** backend e frontend no mesmo repositório, em pastas com ferramentas independentes (`dotnet` / `npm`). O único acoplamento entre eles é HTTP/SignalR. Motivo: projeto mantido por uma só pessoa; uma feature que atravessa as duas camadas vai num único commit.

Dependências entre projetos:

```text
Api ──► Messaging ──► Contracts
Worker ─► Messaging ──► Contracts
```

API e Worker não se referenciam.

## 4. Fluxo

```text
Cliente ── POST /api/tasks ──► Api
                                │ 1. valida entrada (Mode = Simulation, tamanho de Input)
                                │ 2. cria TaskId, CorrelationId; grava Pending no store em memória
                                │ 3. publica AnalysisRequested.v1 e aguarda publisher confirm
                                │ 4. responde 202 Accepted + Location: /api/tasks/{id}
                                ▼
                 exchange aiorchestrator.events (topic, durável)
                   │                                   │
     analysis.requested.v1                 analysis.completed.v1
                   ▼                                   ▼
     fila worker.analysis-requested        fila api.analysis-completed
                   │                                   │
                Worker                         consumidor na Api
      simula processamento                atualiza store: Completed / Failed
      publica AnalysisCompleted.v1
      ACK após publicar

Cliente ── GET /api/tasks/{id} ──► Api lê o store em memória
```

Se a publicação do passo 3 falhar (broker indisponível ou sem confirm), a tarefa é marcada `Failed` e a API responde `503 Service Unavailable`.

## 5. Topologia RabbitMQ

| Elemento | Nome | Propriedades |
|---|---|---|
| Virtual host (app) | `aiorchestrator` | — |
| Virtual host (testes) | `aiorchestrator-tests` | — |
| Usuário da aplicação | (definido localmente) | permissões apenas nos vhosts do projeto; sem tag `administrator` |
| Exchange | `aiorchestrator.events` | `topic`, durável |
| Fila | `worker.analysis-requested` | durável; binding `analysis.requested.v1` |
| Fila | `api.analysis-completed` | durável; binding `analysis.completed.v1` |

- A conta `guest` não é usada pela aplicação.
- Cada processo declara, de forma idempotente na inicialização, o exchange e as filas/bindings que consome. O publicador declara apenas o exchange.
- O broker escuta apenas localmente; as portas não são expostas à rede.

## 6. Contratos (Fase 2)

Os dois eventos têm o mesmo envelope, corrigindo a inconsistência do documento original (eventos `*Completed` sem `SchemaVersion`/`Mode`).

**`AnalysisRequested.v1`** — routing key `analysis.requested.v1`

| Campo | Tipo | Observação |
|---|---|---|
| `MessageId` | `Guid` | único por mensagem |
| `TaskId` | `Guid` | |
| `CorrelationId` | `Guid` | igual em todas as mensagens da tarefa |
| `CreatedAtUtc` | `DateTimeOffset` | UTC |
| `SchemaVersion` | `int` | `1` |
| `Mode` | `enum` | `Simulation` (único aceito nesta fase) |
| `Input` | `string` | limite de tamanho configurável |

**`AnalysisCompleted.v1`** — routing key `analysis.completed.v1`

| Campo | Tipo | Observação |
|---|---|---|
| `MessageId`, `TaskId`, `CorrelationId`, `CreatedAtUtc`, `SchemaVersion`, `Mode` | — | como acima |
| `Status` | `enum` | `Completed` ou `Failed` |
| `FinalResult` | `string?` | presente quando `Completed` |
| `ErrorCode` | `string?` | presente quando `Failed`; sem segredos nem conteúdo sensível |

Propriedades AMQP de toda mensagem: `message_id` = `MessageId`, `correlation_id` = `CorrelationId`, `type` = routing key, `content_type` = `application/json`, `delivery_mode` = persistente. O corpo é JSON (`System.Text.Json`).

## 7. Componentes

**Contracts** — records imutáveis dos eventos, enums `ExecutionMode` e `AnalysisStatus`, classe estática `RoutingKeys`.

**Messaging**
- `RabbitMqOptions` (seção `RabbitMq`: `Host`, `Port`, `VirtualHost`, `Username`, `Password`, `ExchangeName`, `PrefetchCount`).
- Uma `IConnection` singleton por processo, com recuperação automática habilitada e `ClientProvidedName` identificando o processo.
- `IMessagePublisher.PublishAsync<T>(T message, string routingKey)` — canal com *publisher confirms*; só retorna após o confirm do broker.
- `RabbitMqConsumerBase<T>` (`BackgroundService`): declara a fila e o binding, aplica `BasicQos(PrefetchCount)`, desserializa, chama `HandleAsync`, faz `BasicAck` em caso de sucesso.
- Escopo de log com `TaskId`, `CorrelationId` e `MessageId` em toda mensagem publicada ou consumida.

**Api**
- `POST /api/tasks` e `GET /api/tasks/{id}`; Swagger/OpenAPI habilitado.
- `ITaskStore` em memória (`ConcurrentDictionary`), com aviso explícito de que os dados se perdem ao reiniciar.
- Estados nesta fase: `Pending`, `Completed`, `Failed`.
- `AnalysisCompletedConsumer` atualiza o store; é idempotente por `TaskId` (tarefa já em estado final não é alterada).

**Worker**
- `AnalysisRequestedConsumer` gera um resultado simulado determinístico (atraso configurável), publica `AnalysisCompleted.v1` e só então faz ACK.

## 8. Tratamento de falhas nesta fase

- **Mensagem inválida** (JSON malformado, campos obrigatórios ausentes, `SchemaVersion` desconhecida): `BasicNack(requeue: false)` + log de erro. Sem DLX nesta fase, a mensagem é **descartada** — limitação documentada; a DLQ entra na Fase 3.
- **Exceção durante o processamento no Worker:** o Worker publica `AnalysisCompleted.v1` com `Status = Failed` e um `ErrorCode`, e então faz ACK — assim a tarefa chega a `Failed` na API. Se essa publicação também falhar, `BasicNack(requeue: false)` + log, e a tarefa permanece `Pending` (limitação documentada). Retentativas com backoff são escopo da Fase 3.
- **Entrega repetida:** inofensiva — o Worker apenas recalcula a simulação e o consumidor da API ignora tarefas já finalizadas. A deduplicação por `MessageId` fica para a Fase 3.
- **Garantia real:** entrega pelo menos uma vez entre broker e consumidores; mensagens podem ser perdidas se descartadas por NACK ou se o processo da API reiniciar (store em memória).

## 9. Configuração e segredos

- `appsettings.json`: apenas valores não sensíveis (`Host = localhost`, `Port = 5672`, `VirtualHost`, `ExchangeName`, limites).
- `RabbitMq:Username` / `RabbitMq:Password`: `dotnet user-secrets` na API e no Worker; variáveis de ambiente como alternativa.
- Testes de integração leem as credenciais por variáveis de ambiente; sem elas ou sem broker acessível, são pulados.

## 10. Fora de escopo

DLQ, retries com backoff, deduplicação por `MessageId`, Outbox, PostgreSQL, separação Orchestrator/Claude/OpenAI workers, chamadas reais a provedores de IA, Angular, SignalR, autenticação e health checks.
