# Tasks: Cadastro de Produtos

**Input**: Design documents de `/specs/001-cadastro-produto/`

**Prerequisites**: plan.md (obrigatório), spec.md (obrigatório), research.md, data-model.md, contracts/, quickstart.md

**Testes**: obrigatórios. O princípio V da constituição exige teste automatizado em toda mudança de código de produção. Por isso cada fase traz as tarefas de teste, mesmo sem pedido explícito no spec.

**Organização**: tarefas agrupadas por história de usuário, para implementação e teste independentes.

**Revisão pós-análise**: C2 resolvido (testes HTTP black-box, sem `Mvc.Testing`). H1 a H4 aplicados em T023, T024, T037, T039, T042, T048, T054, T060, T063, T070, T071 e T072. Os IDs T070 a T072 foram acrescentados no fim de suas fases, sem renumerar o restante.

## Formato: `[ID] [P?] [História] Descrição com caminho do arquivo`

- **[P]**: pode rodar em paralelo (arquivos diferentes, sem dependência de tarefa incompleta)
- **[USn]**: história de usuário à qual a tarefa pertence
- Fases de Setup, Foundational e Polish não têm rótulo de história

## Convenções de caminho

- Solução `Cadastro.sln` na raiz
- Produção: `src/Cadastro.Domain`, `src/Cadastro.Application`, `src/Cadastro.Infrastructure`, `src/Cadastro.Api`
- Testes: `tests/Cadastro.Domain.Tests`, `tests/Cadastro.Application.Tests`, `tests/Cadastro.Infrastructure.Tests`, `tests/Cadastro.Api.Tests`

---

## Fase 1: Setup (infraestrutura compartilhada)

**Propósito**: estrutura de solução, build e ambiente de contêineres

- [ ] T001 Criar a solução `Cadastro.sln` na raiz e os projetos `src/Cadastro.Domain`, `src/Cadastro.Application`, `src/Cadastro.Infrastructure` e `src/Cadastro.Api` (net10.0). Referências apenas nesta direção: Application→Domain; Infrastructure→Application e Domain; Api→Application e Infrastructure (constituição I)
- [ ] T002 Criar os projetos de teste `tests/Cadastro.Domain.Tests`, `tests/Cadastro.Application.Tests`, `tests/Cadastro.Infrastructure.Tests` e `tests/Cadastro.Api.Tests`, com xUnit, Moq, FluentAssertions e referência ao projeto testado. `Cadastro.Api.Tests` usa `HttpClient` (BCL) contra a API do Compose, sem `Microsoft.AspNetCore.Mvc.Testing` (plan.md, Complexity Tracking, item 2, resolvido). Variável `TESTES__ApiBaseUrl`
- [ ] T003 [P] Criar `Directory.Build.props` na raiz com `TargetFramework` net10.0, `Nullable` enable, `ImplicitUsings` enable e `TreatWarningsAsErrors` true (constituição VI: warnings zero)
- [ ] T004 [P] Criar `.gitignore` na raiz ignorando `bin/`, `obj/`, `.vs/` e `.env`
- [ ] T005 [P] Criar `.env.example` na raiz com as variáveis listadas em quickstart.md, seção 1, com valores de desenvolvimento e sem segredos reais
- [ ] T006 [P] Criar `Dockerfile` multi-stage na raiz: build com `mcr.microsoft.com/dotnet/sdk:10.0`, runtime com `mcr.microsoft.com/dotnet/aspnet:10.0`, porta 8080
- [ ] T007 [P] Criar `docker-compose.yml` na raiz com `postgres` (imagem `postgres:17`, volume nomeado, healthcheck), `rabbitmq` (imagem `rabbitmq:4-management`, portas 5672 e 15672, healthcheck) e `api` (build do `Dockerfile`, porta 8080, `depends_on` com `condition: service_healthy` para os dois serviços de infraestrutura)

---

## Fase 2: Foundational (pré-requisitos bloqueantes)

**Propósito**: domínio central, persistência, outbox, mensageria, autenticação e host da API. Nada das histórias começa antes desta fase.

**⚠️ CRÍTICO**: nenhuma história pode começar antes de esta fase terminar

### Domínio

- [ ] T008 [P] Criar as exceções em `src/Cadastro.Domain/Excecoes/`: `SkuDuplicadoException.cs`, `DadosInvalidosException.cs` (com a propriedade `Campo`) e `ProdutoNaoEncontradoException.cs`
- [ ] T009 [P] Criar `src/Cadastro.Domain/Produtos/ProdutoStatus.cs` com os valores `Ativo` e `Inativo` (data-model.md: situação "`Ativo` ou `Inativo`")
- [ ] T010 Criar o value object `src/Cadastro.Domain/Produtos/Sku.cs`: normaliza com trim e maiúsculas, valida com `^[A-Z0-9-]{1,30}$` (data-model.md: "SKU ... formato `^[A-Z0-9-]{1,30}$`"). Valor inválido lança `DadosInvalidosException` com `Campo = "sku"`
- [ ] T011 [P] Criar os eventos de domínio `src/Cadastro.Domain/Produtos/Eventos/ProdutoCriado.cs` e `ProdutoAtualizado.cs`. Carregam id, nome, sku, preço, estoque, situação, operadorId e ocorridoEm, que formam o campo `dados` do envelope (contracts/eventos-produto.md)
- [ ] T012 Criar o agregado `src/Cadastro.Domain/Produtos/Produto.cs` com fábrica `Criar` e métodos `Atualizar`, `Inativar` e `Reativar`. Regras: nome obrigatório após trim, máximo 150 caracteres (FR-016: "O nome MUST ter no máximo 150 caracteres"); preço "não negativo com até duas casas decimais" (FR-005); estoque "inteiro não negativo" (FR-006); `Atualizar` só com `Situacao == Ativo` (data-model.md, regra 3); `CriadoPor` e `AlteradoPor` recebem `operadorId` (regra 6); inativar produto já inativo não altera nada e não gera evento (FR-018). Acumula eventos em `EventosPendentes`
- [ ] T013 Criar a porta `src/Cadastro.Domain/Produtos/IProdutoRepository.cs` com `ObterPorIdAsync`, `ExisteSkuAsync(sku, idIgnorado)`, `AdicionarAsync`, `ListarAtivosAsync(pagina, tamanho)` ordenada por nome, `ContarAtivosAsync` e `CommitAsync` (justificativa em plan.md, Complexity Tracking, item 1)

### Application (base)

- [ ] T014 [P] Criar `src/Cadastro.Application/Produtos/Dtos.cs` com os records `ProdutoEntrada`, `ProdutoSaida` e `PaginaSaida`

### Configuração (IOptions)

- [ ] T015 [P] Criar as opções em `src/Cadastro.Infrastructure/Options/`: `DatabaseOptions.cs` (`AplicarMigrationsNaInicializacao`), `RabbitMqOptions.cs` (`Host`, `Porta`, `Usuario`, `Senha`, `Exchange` com padrão `cadastro.produtos`), `OutboxOptions.cs` (`IntervaloPolling`, `TamanhoLote`, `MaxTentativas`) e `SessaoOptions.cs` (`NomeCookie` com padrão `Cadastro.Sessao`, `TempoInatividade` com padrão 8 horas). Todas com atributos de validação
- [ ] T016 Criar `src/Cadastro.Infrastructure/DependencyInjection.cs` que registra as opções com `BindConfiguration` e `ValidateDataAnnotations().ValidateOnStart()` (depende de T015)

### Persistência (EF Core + PostgreSQL)

- [ ] T017 [P] Criar `src/Cadastro.Infrastructure/Persistence/Configurations/ProdutoConfiguration.cs` com as constraints de data-model.md: `nome varchar(150)` NOT NULL com CHECK `length(trim(nome)) > 0`; `sku varchar(30)` NOT NULL com índice único `ux_produtos_sku`; `preco numeric(12,2)` com CHECK `preco >= 0`; `estoque integer` com CHECK `estoque >= 0`; `situacao varchar(10)` com CHECK `situacao IN ('Ativo','Inativo')`; `criado_por` e `alterado_por` `varchar(450)` NOT NULL com FK para `AspNetUsers.Id` ON DELETE RESTRICT; `criado_em` e `atualizado_em` `timestamptz` NOT NULL; índice parcial `ix_produtos_listagem_ativos` em `(nome) WHERE situacao = 'Ativo'`
- [ ] T018 [P] Criar `src/Cadastro.Infrastructure/Persistence/Outbox/OutboxMensagem.cs` e `Configurations/OutboxMensagemConfiguration.cs`: colunas `id uuid`, `tipo varchar(100)`, `agregado_id uuid`, `payload jsonb`, `ocorrido_em timestamptz`, `processado_em timestamptz NULL`, `tentativas integer`, `ultimo_erro text NULL`. Índice parcial `ix_outbox_pendentes` em `(ocorrido_em) WHERE processado_em IS NULL`
- [ ] T019 Criar `src/Cadastro.Infrastructure/Persistence/CadastroDbContext.cs`, herdando `IdentityDbContext<IdentityUser>`, com `DbSet<Produto>` e `DbSet<OutboxMensagem>` e aplicando as configurações de T017 e T018 (depende de T017, T018)
- [ ] T020 Criar `src/Cadastro.Infrastructure/Persistence/ProdutoRepository.cs` implementando `IProdutoRepository` com EF Core e LINQ parametrizado, sem SQL concatenado (constituição IV). Depende de T013 e T019
- [ ] T021 Criar `src/Cadastro.Infrastructure/Persistence/Outbox/OutboxInterceptor.cs`: `SaveChangesInterceptor` que, antes do commit, lê `EventosPendentes` dos `Produto` alterados, grava uma `OutboxMensagem` por evento (tipo `produto.criado` ou `produto.atualizado`, payload no envelope de contracts/eventos-produto.md) e limpa a lista. Grava na mesma transação do produto (research.md, item 3). Depende de T011 e T018
- [ ] T022 Registrar o `CadastroDbContext` com Npgsql (connection string em `ConnectionStrings:Padrao`) e o `OutboxInterceptor` em `src/Cadastro.Infrastructure/DependencyInjection.cs` (depende de T016, T019, T021)
- [ ] T023 Gerar a migration inicial `Inicial` em `src/Cadastro.Infrastructure/Persistence/Migrations/`, com `dotnet ef migrations add Inicial --project src/Cadastro.Infrastructure --startup-project src/Cadastro.Api`. As tabelas do Identity precisam ser criadas antes de `produtos` (data-model.md, Migrations). Depende de T019 e de T029, porque o `--startup-project` precisa compilar (H2)

### Mensageria (outbox + RabbitMQ)

- [ ] T024 [P] Criar `src/Cadastro.Infrastructure/Messaging/RabbitMqPublicador.cs` com `RabbitMQ.Client`: declara a exchange `cadastro.produtos` (topic, durável), publica com mensagem persistente, `content-type: application/json`, headers `messageId`, `tipo` e `schemaVersion = 1`, e usa publisher confirms. Classe concreta, sem interface. O método `PublicarAsync` é `virtual`, para o Moq substituí-lo em T037 (research.md, item 4). Depende de T015
- [ ] T025 Criar `src/Cadastro.Infrastructure/Messaging/OutboxWorker.cs` (`BackgroundService`): a cada `IntervaloPolling`, busca lote de `OutboxMensagem` pendentes com `FOR UPDATE SKIP LOCKED`, publica pelo `RabbitMqPublicador` e marca `processado_em`. Em falha, incrementa `tentativas` e grava `ultimo_erro`. Acima de `MaxTentativas`, a mensagem fica parada. Depende de T018 e T024
- [ ] T026 Registrar `RabbitMqPublicador` e `OutboxWorker` em `src/Cadastro.Infrastructure/DependencyInjection.cs` (depende de T025)

### Identidade e sessão (FR-015)

- [ ] T027 Criar `src/Cadastro.Infrastructure/Identity/OperadorSeed.cs`: cria o operador de `Seed:Operador:Email` e `Seed:Operador:Senha` na primeira subida, se não existir (depende de T019)
- [ ] T028 Criar `src/Cadastro.Infrastructure/Identity/SessaoConfiguracao.cs` configurando o cookie: `ExpireTimeSpan` = `SessaoOptions.TempoInatividade`, `SlidingExpiration = true`, `HttpOnly`, `SameSite = Strict`, `SecurePolicy = Always` fora de desenvolvimento (research.md, item 5). Depende de T015

### Host e erros da API

- [ ] T029 Criar `src/Cadastro.Api/Program.cs`: registra serviços e opções, autenticação por cookie, autorização com todas as rotas de produto protegidas (FR-015), `/health`, e aplica migrations quando `Database:AplicarMigrationsNaInicializacao` for true. Depende de T016, T022, T026, T027, T028
- [ ] T030 [P] Criar `src/Cadastro.Api/Erros/ExcecoesParaProblemDetails.cs`: `SkuDuplicadoException` → 409, `DadosInvalidosException` → 422 com `campo`, `ProdutoNaoEncontradoException` → 404, `BancoIndisponivelException` → 503. Todos em Problem Details (RFC 9457)
- [ ] T031 Criar `BancoIndisponivelException` em `src/Cadastro.Infrastructure/Persistence/Erros/` e traduzir, em `ProdutoRepository.CommitAsync`, as falhas de conexão do Npgsql (SQLSTATE classe `08`) e timeouts, para essa exceção (FR-023: "recusar a operação com mensagem de indisponibilidade"). Depende de T020
- [ ] T032 [P] Criar `src/Cadastro.Api/Apresentacao/ProdutoResposta.cs`: converte `ProdutoSaida` em resposta com `id`, `nome`, `sku`, `preco`, `precoExibicao`, `estoque`, `situacao`, `criadoPor`, `alteradoPor`, `criadoEm` e `atualizadoEm`. `precoExibicao` é gerado com `CultureInfo("pt-BR")` no formato `R$ 1.234,56` (FR-024; research.md, item 13)
- [ ] T033 [P] Criar `src/Cadastro.Api/Endpoints/AuthEndpoints.cs`: `POST /api/auth/login` (define o cookie, 200 com `OperadorSaida`; credenciais inválidas → 401; dados inválidos → 422) e `POST /api/auth/logout` (204, remove o cookie). Depende de T028

### Testes da fase Foundational

- [ ] T034 [P] Criar `tests/Cadastro.Domain.Tests/Produtos/SkuTests.cs`: `" abc-001 "` normaliza para `ABC-001`; vazio, caractere inválido e mais de 30 caracteres são rejeitados. Depende de T010
- [ ] T035 [P] Criar `tests/Cadastro.Domain.Tests/Produtos/ProdutoTests.cs`: criar com dados válidos gera `ProdutoCriado` e define `CriadoPor` = `AlteradoPor` = operador; preço negativo, nome com 151 caracteres e nome só com espaços são recusados; inativar produto já inativo não gera evento; atualizar produto inativo é recusado. Depende de T012
- [ ] T036 [P] Criar `tests/Cadastro.Infrastructure.Tests/Persistence/OutboxInterceptorTests.cs`: uma linha de outbox por evento; se o `SaveChanges` falhar, nem produto nem outbox são gravados (mesma transação). Usa o PostgreSQL do Compose via `TESTES__ConnectionString`. Depende de T021, T022
- [ ] T037 [P] Criar `tests/Cadastro.Infrastructure.Tests/Messaging/OutboxWorkerTests.cs` com Moq no método virtual `PublicarAsync` do publicador (T024): publicação com sucesso marca `processado_em`; falha incrementa `tentativas` e grava `ultimo_erro`; acima de `MaxTentativas`, não tenta de novo. Depende de T025
- [ ] T038 Criar `tests/Cadastro.Infrastructure.Tests/Messaging/RabbitMqPublicadorTests.cs`: publica no broker do Compose (`TESTES__RabbitMq__Host`) e consome da fila de teste com a routing key certa. Depende de T024
- [ ] T039 [P] Criar `tests/Cadastro.Api.Tests/AuthTests.cs` (HttpClient contra a API do Compose, sem `Mvc.Testing`): sem cookie, `GET /api/produtos` retorna 401 (FR-015); login válido define o cookie `Cadastro.Sessao`; logout remove o cookie. Depende de T029 e T033

- [ ] T070 [P] Criar `tests/Cadastro.Infrastructure.Tests/Identity/SessaoConfiguracaoTests.cs` (H1, FR-015): verifica que o cookie configurado em `SessaoConfiguracao` tem `ExpireTimeSpan` igual a `SessaoOptions.TempoInatividade` (padrão 8 horas) e `SlidingExpiration = true`. Depende de T028

**Checkpoint**: fundação pronta. As histórias podem começar.

---

## Fase 3: User Story 1 - Cadastrar produto (Prioridade: P1) 🎯 MVP

**Objetivo**: o operador cadastra um produto com nome, SKU, preço e estoque. O produto nasce ativo, com autoria, e o SKU repetido é recusado.

**Teste independente**: `POST /api/produtos` com dados válidos cria o produto; SKU repetido e preço negativo são recusados e nada é gravado.

### Testes da US1

- [ ] T040 [P] [US1] Criar `tests/Cadastro.Application.Tests/Produtos/ProdutoServiceCadastrarTests.cs` (Moq em `IProdutoRepository`): SKU repetido lança `SkuDuplicadoException` e não chama `AdicionarAsync`; cadastro válido grava `operadorId` em `CriadoPor` e `AlteradoPor` e chama `CommitAsync`
- [ ] T041 [P] [US1] Criar `tests/Cadastro.Infrastructure.Tests/Persistence/ProdutoRepositoryCadastroTests.cs`: dois produtos com o mesmo SKU; o segundo `CommitAsync` lança `SkuDuplicadoException` (SQLSTATE 23505, índice `ux_produtos_sku`); o CHECK do banco rejeita preço negativo
- [ ] T042 [P] [US1] Criar `tests/Cadastro.Api.Tests/ProdutosCadastrarTests.cs` (HttpClient com cookie, contra a API do Compose): `POST /api/produtos` válido → 201 com `precoExibicao` `R$ 49,90` e `criadoPor` preenchido; SKU repetido → 409; preço negativo → 422 com `campo = preco`; nome com 151 caracteres → 422 com `campo = nome`

### Implementação da US1

- [ ] T043 [US1] Criar `src/Cadastro.Application/Produtos/ProdutoService.cs` com `CadastrarAsync(ProdutoEntrada, operadorId)`: chama `ExisteSkuAsync` e lança `SkuDuplicadoException` se o SKU estiver em uso; cria `Produto`; chama `AdicionarAsync` e `CommitAsync`. Depende de T012, T013, T014 e T040
- [ ] T044 [US1] Traduzir a violação do índice `ux_produtos_sku` (SQLSTATE 23505) em `ProdutoRepository.CommitAsync` para `SkuDuplicadoException` (depende de T020 e T041)
- [ ] T045 [US1] Adicionar `POST /api/produtos` em `src/Cadastro.Api/Endpoints/ProdutosEndpoints.cs`: exige autenticação, lê o `operadorId` do `ClaimsPrincipal`, retorna 201 com `Location` e `ProdutoResposta` (depende de T043, T032 e T029)

**Checkpoint**: US1 funciona sozinha. Esta é a entrega MVP.

---

## Fase 4: User Story 2 - Listar produtos ativos (Prioridade: P1)

**Objetivo**: a listagem padrão mostra só os produtos ativos, paginada, em ordem alfabética.

**Teste independente**: com ativos e inativos cadastrados, `GET /api/produtos` mostra somente os ativos, na ordem certa e na página certa.

### Testes da US2

- [ ] T046 [P] [US2] Criar `tests/Cadastro.Application.Tests/Produtos/ProdutoServiceListarTests.cs`: padrão de 50 por página; tamanho acima de 200 é limitado a 200; o filtro de ativos é repassado ao repositório; lista vazia retorna total 0 (FR-014)
- [ ] T047 [P] [US2] Criar `tests/Cadastro.Infrastructure.Tests/Persistence/ProdutoRepositoryListagemTests.cs`: ativos ordenados por nome; inativos ausentes; com 120 produtos ativos, a página 3 traz 20 itens (FR-020)
- [ ] T048 [P] [US2] Criar `tests/Cadastro.Api.Tests/ProdutosListarTests.cs` (HttpClient com cookie, contra a API do Compose): com um ativo e um inativo, a listagem traz só o ativo (FR-012); sem ativos, traz lista vazia (FR-014); página além do total traz itens vazios

### Implementação da US2

- [ ] T049 [US2] Implementar `ListarAtivosAsync` e `ContarAtivosAsync` em `ProdutoRepository`: filtro `Situacao == Ativo`, ordenação por `Nome`, `Skip` e `Take` (depende de T020 e T047)
- [ ] T050 [US2] Implementar `ProdutoService.ListarAtivosAsync(pagina, tamanho)` em `src/Cadastro.Application/Produtos/ProdutoService.cs`, com padrão 50 e limite 1..200 (depende de T049 e T046)
- [ ] T051 [US2] Adicionar `GET /api/produtos` com `pagina` e `tamanho` em `ProdutosEndpoints.cs`, retornando `PaginaSaida` mapeada por `ProdutoResposta` (depende de T050 e T048)

**Checkpoint**: US1 e US2 funcionam de forma independente.

---

## Fase 5: User Story 3 - Editar produto (Prioridade: P2)

**Objetivo**: o operador corrige nome, SKU, preço ou estoque de um produto ativo. A edição é recusada se o SKU já for de outro produto ou se o produto foi inativado no meio tempo.

**Teste independente**: `PUT /api/produtos/{id}` aceita edição válida e recusa SKU em uso e produto inativo, sem gravar nada na recusa.

### Testes da US3

- [ ] T052 [P] [US3] Criar `tests/Cadastro.Application.Tests/Produtos/ProdutoServiceAtualizarTests.cs`: SKU de outro produto → `SkuDuplicadoException` e valores anteriores preservados (FR-009); SKU igual ao próprio é aceito; produto inativo → edição recusada; `AlteradoPor` atualizado
- [ ] T053 [P] [US3] Criar `tests/Cadastro.Infrastructure.Tests/Persistence/ProdutoRepositoryAtualizacaoTests.cs`: produto inativado entre a leitura e o salvamento; o `CommitAsync` da edição falha com conflito de concorrência e nada é gravado (caso de borda da clarificação)
- [ ] T054 [P] [US3] Criar `tests/Cadastro.Api.Tests/ProdutosAtualizarTests.cs` (HttpClient com cookie, contra a API do Compose): `PUT` válido → 200 com `alteradoPor`; SKU em uso → 409; preço negativo → 422; produto inativo → 409

### Implementação da US3

- [ ] T055 [US3] Adicionar token de concorrência `xmin` do PostgreSQL ao mapeamento de `Produto` em `ProdutoConfiguration.cs` (`UseXminAsConcurrencyToken`), para detectar a inativação concorrente (depende de T017 e T053)
- [ ] T056 [US3] Implementar `ProdutoService.AtualizarAsync(id, entrada, operadorId)`: checa SKU excluindo o próprio id, como em T043; recusa produto inativo; converte `DbUpdateConcurrencyException` em erro de edição recusada (depende de T052 e T055)
- [ ] T057 [US3] Adicionar `PUT /api/produtos/{id}` em `ProdutosEndpoints.cs` (depende de T056 e T054)

**Checkpoint**: US1, US2 e US3 funcionam de forma independente.

---

## Fase 6: User Story 4 - Inativar, reativar e consultar produto (Prioridade: P2)

**Objetivo**: o operador inativa um produto (ele some da listagem, sem perder dados), reativa e consulta por id, ativo ou inativo.

**Teste independente**: inativar tira o produto da listagem; reativar devolve; `GET /api/produtos/{id}` mostra o produto inativo com os mesmos dados.

### Testes da US4

- [ ] T058 [P] [US4] Criar `tests/Cadastro.Application.Tests/Produtos/ProdutoServiceSituacaoTests.cs`: inativar grava `Inativo` e `AlteradoPor`; inativar produto já inativo não gera evento nem altera dados (FR-018); reativar volta a `Ativo` (FR-017)
- [ ] T059 [P] [US4] Criar `tests/Cadastro.Infrastructure.Tests/Persistence/ProdutoRepositoryObterTests.cs`: `ObterPorIdAsync` retorna produto inativo com nome, SKU, preço e estoque iguais aos anteriores (FR-011, SC-005, FR-021)
- [ ] T060 [P] [US4] Criar `tests/Cadastro.Api.Tests/ProdutosSituacaoTests.cs` (HttpClient com cookie, contra a API do Compose): `POST .../inativacao` → 200 e o produto some da listagem; `POST .../reativacao` → 200 e volta; `GET /api/produtos/{id}` de produto inativo → 200 (FR-021); id inexistente → 404

### Implementação da US4

- [ ] T061 [US4] Implementar `ProdutoService.InativarAsync`, `ReativarAsync` e `ObterPorIdAsync` em `ProdutoService.cs` (depende de T058 e T059)
- [ ] T062 [US4] Adicionar `POST /api/produtos/{id}/inativacao`, `POST /api/produtos/{id}/reativacao` e `GET /api/produtos/{id}` em `ProdutosEndpoints.cs` (depende de T061 e T060)

**Checkpoint**: todas as histórias funcionam de forma independente.

---

## Fase 7: Polish e preocupações transversais

**Propósito**: validações de ponta a ponta, ambiente e revisão final

- [ ] T063 [P] Criar `tests/Cadastro.Api.Tests/Erros/ExcecoesParaProblemDetailsTests.cs`: teste de função pura, sem servidor. `BancoIndisponivelException` mapeia para 503 em Problem Details; `SkuDuplicadoException` para 409; `DadosInvalidosException` para 422 com `campo`. A queda real do banco é validada em T067 (quickstart, seção 4) (FR-023)
- [ ] T064 [P] Criar `tests/Cadastro.Infrastructure.Tests/Messaging/OutboxBrokerIndisponivelTests.cs`: com o broker parado, os eventos ficam pendentes e são publicados quando ele volta, sem perda (FR-019, SC-007)
- [ ] T065 [P] Rodar `dotnet build Cadastro.sln` e confirmar zero erros e zero warnings (constituição VI)
- [ ] T066 Subir o ambiente com `docker compose up -d --build` e confirmar `docker compose ps` com `api`, `postgres` e `rabbitmq` em `running` e `healthy`, depois `curl http://localhost:8080/health` → `Healthy` (quickstart.md, seção 2)
- [ ] T067 Executar quickstart.md, seções 3 a 5 (fluxo do operador, indisponibilidade do banco, outbox e RabbitMQ), e registrar a saída real de cada passo
- [ ] T068 Executar `dotnet test Cadastro.sln` com todos os testes passando, e reportar a saída real (constituição VI)
- [ ] T069 [P] Revisar o diff contra a constituição: direção de dependência (I), invariantes no agregado (II), nenhum valor fixo de ambiente (III), nenhum SQL concatenado (IV), teste em cada mudança (V) e abstração sem segundo uso (VII)
- [ ] T071 [P] Validar expiração e renovação da sessão ponta a ponta (H1, FR-015): subir o Compose com `Sessao__TempoInatividade=00:00:30`; fazer login; esperar 35 s sem uso e confirmar `401` em `GET /api/produtos`. Fazer login de novo, usar a cada 20 s durante 60 s e confirmar que a sessão continua válida (renovação por uso). Registrar a saída real. Depende de T066
- [ ] T072 [P] Medir a listagem com 10.000 produtos (H3, SC-006): popular 10.000 produtos (ativos e inativos), medir o tempo da primeira página de `GET /api/produtos` e confirmar em `EXPLAIN ANALYZE` que o índice `ix_produtos_listagem_ativos` é usado. Critério: primeira página em menos de 2 s. Depende de T023 e T049

---

## Dependências e ordem de execução

### Dependências entre fases

- **Setup (Fase 1)**: sem dependências
- **Foundational (Fase 2)**: depende do Setup e bloqueia todas as histórias
- **Histórias (Fases 3 a 6)**: dependem da Foundational. Podem seguir em paralelo, ou em ordem de prioridade (US1 → US2 → US3 → US4)
- **Polish (Fase 7)**: depende das histórias concluídas

### Dependências entre histórias

- **US1 (P1)**: independente depois da Foundational. É o MVP
- **US2 (P1)**: independente depois da Foundational. Usa dados que a US1 cria, mas os testes criam os seus próprios
- **US3 (P2)**: independente depois da Foundational. Reusa a checagem de SKU da US1 (T043) como referência
- **US4 (P2)**: independente depois da Foundational. Depende da listagem (US2) só para a verificação visual de inativação

### Dentro de cada história

- Testes antes da implementação (cada teste deve falhar antes do código)
- Domínio e aplicação antes da API
- Endpoints por último

### Oportunidades de paralelismo

- Setup: T003 a T007 em paralelo
- Foundational: T008, T009, T011, T014, T015, T017, T018, T024, T030, T032, T033 em paralelo, quando as dependências anteriores estiverem prontas
- Testes de Foundational: T034, T035, T036, T037, T039 em paralelo
- Cada história: testes marcados [P] em paralelo
- Histórias diferentes podem ser feitas por pessoas diferentes depois da Foundational

---

## Exemplo de paralelismo: User Story 1

```text
# Testes da US1, em paralelo:
T040 ProdutoServiceCadastrarTests.cs
T041 ProdutoRepositoryCadastroTests.cs
T042 ProdutosCadastrarTests.cs
```

---

## Estratégia de implementação

### MVP primeiro (somente US1)

1. Fase 1: Setup (T001 a T007)
2. Fase 2: Foundational (T008 a T039)
3. Fase 3: US1 (T040 a T045)
4. **PARAR e validar**: cadastro e recusas funcionam

### Entrega incremental

1. Setup + Foundational → fundação pronta
2. US1 → MVP
3. US2 → listagem
4. US3 → edição
5. US4 → inativação, reativação e consulta
6. Polish → validações de ponta a ponta

---

## Resumo

| Fase | Tarefas | Intervalo |
|------|---------|-----------|
| Setup | 7 | T001 a T007 |
| Foundational | 33 | T008 a T039, T070 |
| US1 - Cadastrar (P1) | 6 | T040 a T045 |
| US2 - Listar (P1) | 6 | T046 a T051 |
| US3 - Editar (P2) | 6 | T052 a T057 |
| US4 - Inativar, reativar, consultar (P2) | 5 | T058 a T062 |
| Polish | 9 | T063 a T069, T071, T072 |
| **Total** | **72** | |

**Formato**: todas as tarefas seguem `- [ ] Txxx [P?] [USn?] Descrição com caminho`.
