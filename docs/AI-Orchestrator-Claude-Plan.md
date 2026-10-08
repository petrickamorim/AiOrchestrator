# Projeto: AI Orchestrator
## Especificação para o Claude Code — arquitetura, instalações e implementação

> **Objetivo:** construir um laboratório de arquitetura distribuída em C#/.NET no Windows, usando mensageria real com RabbitMQ, para orquestrar tarefas entre modelos de IA como Claude e OpenAI ChatGPT.
>
> **Modo de trabalho esperado:** primeiro inspecionar o ambiente e apresentar um plano; depois instalar/configurar o que estiver faltando, implementar em etapas e validar cada etapa. Não usar Docker.

---

## 1. Instruções obrigatórias para o Claude Code

1. Leia este documento inteiro antes de executar qualquer ação.
2. Inspecione o repositório e o ambiente atual antes de criar, alterar, instalar ou remover qualquer coisa.
3. **Não use Docker nem Docker Compose** em nenhuma etapa.
4. Não sobrescreva arquivos existentes sem inspecioná-los e explicar a alteração.
5. Antes de instalar ferramentas, verifique o que já está instalado e suas versões.
6. Para instalações e alterações no Windows, explique quais comandos exigem PowerShell como Administrador. Peça autorização antes de executar ações privilegiadas, modificar políticas do sistema, alterar firewall, criar usuários, alterar variáveis globais ou instalar serviços.
7. Prefira instaladores e documentação oficiais. Não baixe binários de fontes desconhecidas.
8. Nunca mostre, registre, versione ou envie chaves de API, senhas ou outros segredos. Não solicite que o usuário cole segredos no chat.
9. Não execute chamadas pagas a APIs de IA sem confirmação explícita do usuário. Implemente inicialmente um modo simulado para validar o fluxo sem custo.
10. Não invente resultados de comandos ou testes. Execute-os e informe o resultado real.
11. Mantenha as etapas pequenas. Após cada etapa, apresente o que foi feito, os arquivos alterados, os comandos de validação e eventuais problemas.
12. Se uma instalação não puder ser executada por falta de permissão, conectividade ou capacidade do ambiente, forneça o comando oficial e explique como o usuário pode executá-lo manualmente.
13. Antes de implementar, apresente um plano curto com diagnóstico, dependências, riscos e sequência de trabalho. Depois prossiga sem pedir confirmação para ações locais comuns e reversíveis. Peça confirmação para ações privilegiadas, destrutivas, pagas ou que alterem configurações globais.
14. Use português do Brasil nas explicações e documentação do projeto. Código, classes, variáveis, eventos e endpoints devem usar nomes consistentes em inglês.

---

## 2. Objetivos de aprendizagem

O projeto deve permitir estudar e praticar:

- APIs REST com ASP.NET Core.
- Workers independentes com .NET Worker Service e `BackgroundService`.
- Mensageria real com RabbitMQ.
- Produtores, consumidores, exchanges, filas, bindings e routing keys.
- ACK/NACK, retentativas, dead-letter queues e tratamento de mensagens inválidas.
- Processamento assíncrono e arquitetura orientada a eventos.
- Correlação de mensagens e rastreamento de uma execução de ponta a ponta.
- Idempotência, timeouts e falhas temporárias.
- Integração com a API da Anthropic e a API da OpenAI.
- Orquestração sequencial de modelos: Claude analisa, ChatGPT revisa e o sistema consolida.
- Persistência do estado de uma tarefa.
- Atualização de status no frontend em tempo real, em uma etapa posterior.
- Testes unitários e de integração, logs estruturados e configuração segura.

O objetivo é didático, mas o resultado deve seguir boas práticas e permitir evolução para um produto real.

---

## 3. Stack desejada

### Obrigatória no primeiro MVP

- Windows.
- .NET SDK em uma versão LTS suportada disponível no ambiente; preferir .NET 10 LTS se compatível com o ambiente e dependências. Caso contrário, usar .NET 8 LTS e documentar a escolha.
- ASP.NET Core Web API.
- .NET Worker Service.
- RabbitMQ instalado **diretamente no Windows**, sem Docker.
- Erlang/OTP em versão compatível com a versão de RabbitMQ escolhida.
- Cliente oficial ou biblioteca .NET madura e compatível para comunicação AMQP com RabbitMQ.
- Swagger/OpenAPI para testar a API.
- xUnit para testes.

### Para etapas posteriores

- PostgreSQL para persistir tarefas e resultados.
- Angular para a interface.
- SignalR para atualizações em tempo real.
- API da Anthropic (Claude).
- API da OpenAI (ChatGPT).
- Git para versionamento.

Não instale todos os componentes de uma vez. Comece com o mínimo necessário e introduza cada dependência na etapa correspondente.

---

## 4. Restrições e decisões arquiteturais

- Não utilizar Docker, containers ou Docker Compose.
- Não depender de uma fila em memória como solução final de mensageria. `Channel<T>` pode ser usado apenas em um exemplo comparativo opcional, nunca como substituto do RabbitMQ no fluxo principal.
- A API e o Worker devem ser processos/projetos independentes.
- A API não deve chamar os modelos de IA diretamente no fluxo HTTP. Deve registrar/publicar a solicitação e retornar um identificador de tarefa.
- O processamento dos modelos deve acontecer nos workers, fora da requisição HTTP.
- O RabbitMQ transporta mensagens; os workers são responsáveis por executar a lógica e chamar os provedores de IA.
- Começar com um fluxo sequencial simples. Não criar um sistema multiagente complexo antes de validar a mensageria básica.
- Não implementar microserviços excessivos, Kubernetes, Kafka, Redis ou frameworks de agentes no MVP inicial.
- Manter contratos de mensagens explícitos, versionáveis e documentados.
- Usar `CorrelationId` e `TaskId` para rastrear todas as mensagens da mesma execução.
- Projetar consumidores para lidar com entrega repetida. Não presumir entrega exatamente uma vez.
- Usar ACK manual após processamento bem-sucedido, conforme a biblioteca adotada. Definir estratégia segura para falhas, evitando loops infinitos de requeue.
- Configurar filas duráveis e mensagens persistentes para o fluxo principal.
- Planejar dead-letter queue para mensagens que falham após o limite de retentativas.
- Manter segredos fora do código e do Git. Usar .NET User Secrets em desenvolvimento e variáveis de ambiente quando apropriado.
- Não enviar código privado, dados pessoais ou informações confidenciais aos provedores de IA sem consentimento explícito do usuário.
- Limitar tamanho de prompts e respostas e definir limites de custo, tempo e número de rodadas.
- Tratar a saída dos modelos como conteúdo não confiável; nunca executar código sugerido por um modelo automaticamente.
- Configurar autenticação e HTTPS antes de qualquer exposição pública. Na fase local, vincular serviços apenas a interfaces locais quando possível.

---

## 5. Arquitetura-alvo

```text
[Angular — etapa posterior]
          |
          | HTTP
          v
[ASP.NET Core Web API]
          |
          | publica AnalysisRequested
          v
[RabbitMQ — instalado no Windows]
          |
          v
[Orchestrator Worker]
          |
          | publica ClaudeAnalysisRequested
          v
[RabbitMQ]
          |
          v
[Claude Worker] ----> [Anthropic API]
          |
          | publica ClaudeAnalysisCompleted
          v
[RabbitMQ]
          |
          v
[Orchestrator Worker]
          |
          | publica OpenAIReviewRequested
          v
[RabbitMQ]
          |
          v
[OpenAI Worker] ----> [OpenAI API]
          |
          | publica OpenAIReviewCompleted
          v
[RabbitMQ]
          |
          v
[Orchestrator Worker]
          |
          | consolida e persiste o resultado
          v
[PostgreSQL — etapa posterior]
          |
          v
[GET /api/tasks/{id} e SignalR — etapas posteriores]
```

Para reduzir complexidade, o primeiro MVP pode usar um único Worker que consome uma tarefa e publica um evento de conclusão. Depois, separar os workers de IA e introduzir o orquestrador.

### Fluxo completo esperado

1. O cliente envia uma tarefa para `POST /api/tasks`.
2. A API valida a solicitação, cria um `TaskId` e um `CorrelationId` e publica `AnalysisRequested`.
3. A API retorna `202 Accepted`, com o identificador da tarefa e uma URL para consultar o estado.
4. O worker consome a mensagem do RabbitMQ.
5. No primeiro marco, o worker simula o processamento sem chamar APIs pagas.
6. Em uma etapa posterior, o Claude analisa a tarefa.
7. O orquestrador encaminha a análise ao worker da OpenAI para revisão.
8. O resultado final é persistido e o estado da tarefa muda para concluído ou falhou.
9. O cliente consulta `GET /api/tasks/{id}`; SignalR poderá ser adicionado posteriormente.

---

## 6. Contratos de eventos iniciais

Criar contratos tipados e versionados. Os nomes podem ser ajustados se houver uma justificativa técnica documentada.

### `AnalysisRequested.v1`

Campos sugeridos:

- `MessageId`: identificador único da mensagem.
- `TaskId`: identificador da tarefa.
- `CorrelationId`: identificador para rastrear o fluxo completo.
- `CreatedAtUtc`: data/hora UTC.
- `Input`: instrução do usuário.
- `Mode`: `Simulation` ou `Live`.
- `SchemaVersion`: versão do contrato.

### `ClaudeAnalysisRequested.v1`

- `MessageId`
- `TaskId`
- `CorrelationId`
- `CreatedAtUtc`
- `Input`
- `Mode`
- `SchemaVersion`

### `ClaudeAnalysisCompleted.v1`

- `MessageId`
- `TaskId`
- `CorrelationId`
- `CreatedAtUtc`
- `Status`
- `Analysis`
- `Provider`
- `Model`
- `DurationMs`
- `ErrorCode` opcional, sem segredos nem conteúdo sensível.

### `OpenAIReviewRequested.v1`

- `MessageId`
- `TaskId`
- `CorrelationId`
- `CreatedAtUtc`
- `OriginalInput`
- `ClaudeAnalysis`
- `Mode`
- `SchemaVersion`

### `OpenAIReviewCompleted.v1`

- `MessageId`
- `TaskId`
- `CorrelationId`
- `CreatedAtUtc`
- `Status`
- `Review`
- `Provider`
- `Model`
- `DurationMs`
- `ErrorCode` opcional.

### `AnalysisCompleted.v1`

- `MessageId`
- `TaskId`
- `CorrelationId`
- `CreatedAtUtc`
- `Status`
- `FinalResult`
- `SchemaVersion`

Não incluir chaves de API, credenciais, prompts de sistema secretos ou dados sensíveis desnecessários nos eventos e logs.

---

## 7. Plano de implementação por fases

### Fase 0 — Diagnóstico do ambiente

1. Inspecionar o sistema operacional e o diretório atual.
2. Verificar Git, .NET SDK, PowerShell e ferramentas de compilação disponíveis.
3. Verificar se Erlang/OTP e RabbitMQ já estão instalados.
4. Verificar serviços e portas relevantes, principalmente `5672` e `15672`.
5. Confirmar compatibilidade entre a versão escolhida do RabbitMQ e Erlang/OTP.
6. Não instalar versões por tentativa. Consultar a documentação oficial atual.
7. Apresentar um relatório do que existe e do que falta.

### Fase 1 — Instalação e configuração local

1. Instalar somente as dependências ausentes.
2. Para instalações que exigem elevação, solicitar autorização e usar os instaladores oficiais.
3. Configurar RabbitMQ como serviço do Windows, seguindo o procedimento oficial suportado.
4. Habilitar o plugin de gerenciamento quando necessário.
5. Criar credenciais de desenvolvimento apropriadas e documentar como alterá-las.
6. Evitar usar a conta `guest` remotamente; manter acesso local restrito.
7. Verificar o serviço, a porta AMQP e o painel de gerenciamento.
8. Documentar comandos para iniciar, parar, reiniciar e diagnosticar o serviço.

### Fase 2 — API e Worker mínimos

1. Criar uma solução .NET organizada, por exemplo:

```text
AiOrchestrator/
├── AiOrchestrator.sln
├── src/
│   ├── AiOrchestrator.Api/
│   ├── AiOrchestrator.Worker/
│   └── AiOrchestrator.Contracts/
├── tests/
│   ├── AiOrchestrator.UnitTests/
│   └── AiOrchestrator.IntegrationTests/
├── docs/
│   ├── architecture.md
│   ├── setup-windows.md
│   └── message-contracts.md
├── .gitignore
└── README.md
```

2. Ajustar a estrutura padrão criada pelos templates do .NET para corresponder à solução.
3. Criar um endpoint `POST /api/tasks`.
4. Criar `GET /api/tasks/{id}`; nesta fase, o estado pode ser mantido em memória com aviso explícito de que se perde ao reiniciar.
5. Implementar publicação no RabbitMQ.
6. Implementar consumo em um Worker Service independente.
7. Usar configuração tipada, injeção de dependência e logs estruturados.
8. Criar um modo simulado que não invoque nenhum provedor pago.
9. Validar que a mensagem publicada pela API é recebida e processada pelo worker.

### Fase 3 — Robustez da mensageria

1. Usar exchanges e filas duráveis.
2. Usar mensagens persistentes quando apropriado.
3. Implementar ACK manual depois do processamento bem-sucedido.
4. Implementar tentativas limitadas e backoff para falhas temporárias.
5. Configurar dead-letter exchange/queue.
6. Tratar mensagens inválidas sem criar ciclos infinitos de requeue.
7. Implementar idempotência e deduplicação por identificador.
8. Adicionar `CorrelationId`, `TaskId` e `MessageId` aos logs.
9. Validar comportamento diante de reinicialização do worker e indisponibilidade temporária do RabbitMQ.
10. Documentar limites conhecidos e garantias reais do fluxo.

### Fase 4 — Integração com Claude

1. Implementar uma abstração `IAiProvider`.
2. Criar um adaptador para a API da Anthropic.
3. Obter a chave por User Secrets ou variável de ambiente.
4. Selecionar um modelo disponível e configurável; não fixar identificadores obsoletos sem verificar a documentação.
5. Implementar timeout, cancelamento, tratamento de rate limits e erros.
6. Registrar apenas metadados úteis; não gravar chaves, cabeçalhos de autenticação ou prompts sensíveis.
7. Manter o modo simulado como padrão até o usuário ativar explicitamente o modo real.

### Fase 5 — Integração com OpenAI

1. Criar um adaptador independente para a API da OpenAI.
2. Obter a chave por User Secrets ou variável de ambiente.
3. Usar a API recomendada pela documentação oficial atual, com modelo configurável.
4. Enviar a análise do Claude para revisão, com limites de tamanho e contexto.
5. Tratar timeouts, erros temporários, rate limits e respostas inválidas.
6. Manter o fluxo testável sem chamadas pagas usando implementações simuladas.

### Fase 6 — Orquestração e persistência

1. Definir estados da tarefa: `Pending`, `Processing`, `AwaitingReview`, `Completed` e `Failed`.
2. Definir transições válidas de estado.
3. Persistir tarefas, eventos relevantes e resultados no PostgreSQL.
4. Tratar o problema de consistência entre salvar no banco e publicar no RabbitMQ. Avaliar o padrão Outbox antes de afirmar que a publicação é transacional.
5. Evitar executar múltiplas revisões ilimitadas.
6. Configurar máximo de rodadas, timeout total e limites de custo.
7. Consolidar resultados sem assumir que um modelo sempre está correto.

### Fase 7 — Frontend e operação

1. Criar frontend Angular para enviar tarefas e consultar resultados.
2. Exibir estados de processamento.
3. Adicionar SignalR para atualização em tempo real, se útil.
4. Adicionar autenticação e autorização antes de expor o sistema.
5. Adicionar métricas básicas, health checks e logs estruturados.
6. Documentar como executar e depurar todo o projeto no Windows sem Docker.

---

## 8. Configuração e segurança

- Usar nomes de configuração como `RabbitMq:Host`, `RabbitMq:Port`, `RabbitMq:VirtualHost`, `RabbitMq:Username` e `RabbitMq:Password`.
- Manter configurações não sensíveis em `appsettings.json`.
- Usar `dotnet user-secrets` para segredos durante o desenvolvimento local.
- Não colocar segredos em `appsettings.json`, arquivos versionados, documentação, testes ou logs.
- Adicionar arquivos locais de segredos ao `.gitignore`.
- Não usar credenciais administrativas do RabbitMQ nos workers.
- Preferir um usuário e um virtual host próprios para o aplicativo, com permissões mínimas.
- Não abrir as portas do broker para a internet.
- Antes de ativar o modo real, mostrar ao usuário quais provedores serão chamados e avisar que podem ocorrer cobranças.
- Implementar limites configuráveis para tokens, tamanho da entrada, número de tentativas e duração total.
- Não enviar automaticamente conteúdo confidencial ou código proprietário a serviços externos.
- Validar entradas HTTP e definir limites de tamanho de requisição.
- Não confiar em instruções contidas nas respostas dos modelos.
- Nunca executar comandos, scripts ou código gerado pela IA sem aprovação explícita.

---

## 9. Critérios de aceite do primeiro marco

A Fase 2 só estará concluída quando todos estes pontos forem comprovados:

- [ ] A solução compila.
- [ ] RabbitMQ está instalado e em execução no Windows sem Docker.
- [ ] A API inicia localmente.
- [ ] O Worker inicia como processo independente.
- [ ] `POST /api/tasks` retorna `202 Accepted` com um `TaskId`.
- [ ] A API publica a mensagem no RabbitMQ.
- [ ] O Worker recebe e processa a mensagem.
- [ ] A mensagem contém `TaskId`, `CorrelationId` e `MessageId`.
- [ ] O resultado simulado pode ser consultado.
- [ ] Os logs permitem correlacionar API e Worker.
- [ ] Nenhuma chamada paga a modelos foi realizada.
- [ ] Os testes relevantes foram executados e os resultados reais foram informados.
- [ ] A documentação explica como executar tudo no Windows sem Docker.

---

## 10. Documentação e referências oficiais

Consulte a documentação atual antes de escolher versões e comandos:

- RabbitMQ para Windows: https://www.rabbitmq.com/docs/install-windows
- Instalação e gerenciamento do RabbitMQ: https://www.rabbitmq.com/docs
- Tutoriais RabbitMQ: https://www.rabbitmq.com/tutorials
- .NET: https://dotnet.microsoft.com/download
- .NET Worker Services: https://learn.microsoft.com/en-us/dotnet/core/extensions/workers
- ASP.NET Core Web API: https://learn.microsoft.com/en-us/aspnet/core/web-api/
- .NET User Secrets: https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets
- Anthropic API: https://platform.claude.com/docs
- OpenAI API: https://developers.openai.com/api/docs

Se algum link tiver sido reorganizado, localize a documentação oficial atual do mesmo produto. Não baseie a instalação em tutoriais antigos de terceiros sem verificar compatibilidade.

---

## 11. Formato de resposta ao concluir cada fase

Ao final de cada fase, forneça:

1. **Status:** concluída, parcial ou bloqueada.
2. **O que foi feito:** resumo objetivo.
3. **Arquivos criados/alterados:** caminhos e finalidade.
4. **Instalações/configurações:** versões verificadas e serviços envolvidos.
5. **Validação:** comandos executados e seus resultados reais.
6. **Problemas pendentes:** erros e próximos passos.
7. **Como executar localmente:** comandos exatos.
8. **Conceitos aprendidos:** breve explicação didática das decisões tomadas.

## Início

Comece pela **Fase 0 — Diagnóstico do ambiente**. Inspecione o diretório atual e as ferramentas instaladas. Apresente um plano conciso e, em seguida, avance com as ações locais seguras e reversíveis. Não use Docker. Não faça chamadas pagas de IA. Não execute ações privilegiadas sem autorização explícita.
