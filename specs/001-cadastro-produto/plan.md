# Implementation Plan: Cadastro de Produtos

**Branch**: `001-cadastro-produto` | **Date**: 2026-10-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-cadastro-produto/spec.md`

## Summary

API REST para o operador cadastrar, editar, listar, inativar e reativar produtos
(nome, SKU único, preço, estoque), com login de operador (sessão expira após 8 h de inatividade). Cada criação e
atualização de produto gera um evento publicado no RabbitMQ pelo padrão
**Transactional Outbox**: o evento é gravado na mesma transação do produto e
publicado depois por um worker em segundo plano. Toda a solução sobe com
Docker Compose (API, PostgreSQL e RabbitMQ).

Abordagem técnica (detalhes em [research.md](./research.md)):
- Clean Architecture em 4 projetos: Domain, Application, Infrastructure, Api.
- Sem MediatR: `ProdutoService` na Application, uma classe com um método por caso de uso.
- SKU validado no domínio (formato e normalização) e garantido no banco por índice único.
- Outbox gravada pelo `SaveChanges` via interceptor, no mesmo commit do produto.

## Technical Context

**Language/Version**: C# 14 sobre .NET 10 (`net10.0`)

**Primary Dependencies**: ASP.NET Core 10 (Minimal APIs), EF Core 10 com Npgsql,
ASP.NET Core Identity com autenticação por cookie (expiração deslizante), RabbitMQ.Client (cliente oficial .NET)

**Storage**: PostgreSQL 17 (tabelas de produto, outbox e Identity)

**Testing**: xUnit, Moq, FluentAssertions

**Target Platform**: contêineres Linux via Docker Compose; desenvolvimento em Windows com WSL2

**Project Type**: web-service (API REST) com worker em segundo plano no mesmo processo

**Performance Goals**: listagem padrão com primeira página em menos de 2 s com 10.000 produtos (SC-006)

**Constraints**: nenhum valor de ambiente no código (IOptions e variáveis de ambiente);
SQL sempre parametrizado; entrega do evento "pelo menos uma vez" com consumidores idempotentes

**Scale/Scope**: até 10.000 produtos (ativos e inativos); um operador por conta; sem perfis

## Constitution Check

*GATE: avaliado antes da Fase 0 e reavaliado após a Fase 1.*

| Princípio | Verificação | Resultado |
|-----------|-------------|-----------|
| I. Clean Architecture | Domain sem referências; Application referencia só Domain; Infrastructure referencia Application e Domain; Api referencia Application e Infrastructure (composição de DI) | PASS |
| II. DDD | `Produto` é o agregado; invariantes (nome, SKU, preço, estoque) dentro da entidade; eventos de domínio emitidos pelo agregado | PASS |
| III. Configuração externa | Opções tipadas (`DatabaseOptions`, `RabbitMqOptions`, `OutboxOptions`, `SessaoOptions`) com `ValidateOnStart`; segredos via variáveis de ambiente. Cultura pt-BR do preço é regra de negócio (FR-024), não configuração | PASS |
| IV. EF Core + PostgreSQL | Npgsql; consultas via LINQ; migrations versionadas; sem SQL concatenado | PASS |
| V. Testes | Quatro projetos de teste; xUnit, Moq, FluentAssertions; testes de integração usam PostgreSQL e RabbitMQ reais do Compose, sem Testcontainers | PASS (testes HTTP black-box via HttpClient, sem Mvc.Testing) |
| VI. Build e testes verdes | Portões definidos no fluxo de tasks e implement | PASS (aplicado na implementação) |
| VII. YAGNI | Porta de repositório com uma implementação: coberta pela exceção da constituição 1.1.0 (ver Complexity Tracking, item 1) | PASS (com exceção) |
| Stack | .NET 10, EF Core, PostgreSQL, xUnit/Moq/FluentAssertions | PASS |

**Dependências NuGet novas** (justificadas pelo requisito): `Npgsql.EntityFrameworkCore.PostgreSQL`,
`Microsoft.AspNetCore.Identity.EntityFrameworkCore`, `RabbitMQ.Client`. Autenticação por cookie já vem no framework ASP.NET Core.
`Microsoft.AspNetCore.Mvc.Testing` foi removido (ver Complexity Tracking, item 2).

## Project Structure

### Documentation (this feature)

```text
specs/001-cadastro-produto/
├── plan.md                    # Este arquivo
├── research.md                # Fase 0: decisões técnicas
├── data-model.md              # Fase 1: entidades, tabela outbox, regras de banco
├── quickstart.md              # Fase 1: como subir e validar o ambiente
├── contracts/
│   ├── produtos-api.yaml      # Fase 1: contrato HTTP (OpenAPI 3.1)
│   └── eventos-produto.md     # Fase 1: contrato dos eventos no RabbitMQ
├── checklists/requirements.md
└── tasks.md                   # NÃO criado aqui (/speckit-tasks)
```

### Source Code (repository root)

```text
Cadastro.sln
Dockerfile                     # build multi-stage da Api
docker-compose.yml             # api + postgres + rabbitmq
.env.example                   # variáveis de ambiente de desenvolvimento (sem segredos reais)

src/
├── Cadastro.Domain/
│   ├── Produtos/
│   │   ├── Produto.cs                 # agregado: invariantes, inativar, reativar, atualizar
│   │   ├── Sku.cs                     # value object: normalização e formato
│   │   ├── ProdutoStatus.cs           # Ativo | Inativo
│   │   ├── IProdutoRepository.cs      # porta de persistência e verificação de SKU
│   │   └── Eventos/ProdutoCriado.cs, ProdutoAtualizado.cs
│   └── Excecoes/SkuDuplicadoException.cs, DadosInvalidosException.cs, ProdutoNaoEncontradoException.cs
├── Cadastro.Application/
│   └── Produtos/
│       ├── ProdutoService.cs          # casos de uso: cadastrar, editar, listar, obter, inativar, reativar
│       └── Dtos.cs                    # entradas e saídas (records)
├── Cadastro.Infrastructure/
│   ├── Persistence/
│   │   ├── CadastroDbContext.cs
│   │   ├── Configurations/ProdutoConfiguration.cs, OutboxMensagemConfiguration.cs
│   │   ├── ProdutoRepository.cs       # implementa IProdutoRepository com EF Core
│   │   ├── Outbox/OutboxInterceptor.cs  # grava eventos na mesma transação do SaveChanges
│   │   └── Migrations/
│   ├── Messaging/
│   │   ├── RabbitMqPublicador.cs      # conexão, exchange, publisher confirms
│   │   └── OutboxWorker.cs            # BackgroundService: lê outbox, publica, marca processado
│   ├── Identity/                      # usuários de operador, seed de desenvolvimento
│   ├── Options/                       # DatabaseOptions, RabbitMqOptions, OutboxOptions, SessaoOptions
│   └── DependencyInjection.cs
└── Cadastro.Api/
    ├── Program.cs
    ├── Endpoints/ProdutosEndpoints.cs, AuthEndpoints.cs
    ├── Erros/ExcecoesParaProblemDetails.cs   # mapeia exceções para 409, 422, 404, 503
    ├── appsettings.json               # apenas padrões não sensíveis
    └── Properties/launchSettings.json

tests/
├── Cadastro.Domain.Tests/             # unitários: Produto, Sku
├── Cadastro.Application.Tests/        # unitários: ProdutoService com Moq
├── Cadastro.Infrastructure.Tests/     # integração: repositório, índice único, outbox, RabbitMQ
└── Cadastro.Api.Tests/                # HTTP black-box (HttpClient) contra a API do Compose; problem details por função pura
```

**Structure Decision**: 4 projetos de produção, conforme pedido, com dependências em
uma direção só. Testes em projetos separados por camada. Dockerfile e Compose ficam na
raiz porque o build precisa da solução inteira.

## Complexity Tracking

> Violações do Constitution Check que precisam de justificativa.

| Violação | Por que é necessária | Alternativa mais simples rejeitada porque |
|----------|----------------------|-------------------------------------------|
| 1. `IProdutoRepository` com uma única implementação de produção (princípio VII), coberto pela exceção da constituição 1.1.0 | O princípio I proíbe a Application de referenciar EF Core. Sem a porta, `ProdutoService` teria que depender de `DbContext`, e a regra de SKU "no domínio" não teria como consultar o banco | Usar `DbContext` direto na Application quebra o princípio I. Lógica de SKU só no banco quebra o requisito de validação no domínio |
| 2. `Microsoft.AspNetCore.Mvc.Testing` nos testes de Api (stack da constituição) | Hospedaria a API em memória. Não está na lista permitida (xUnit, Moq, FluentAssertions) | **Resolvido**: pacote removido. Testes HTTP usam `HttpClient` (BCL) contra a API do Compose. A validação de falha de banco vira teste de função pura (T063) e teste manual (T067) |

## Decisões de Projeto que Afetam o Spec

- **Reativação e eventos**: inativar e reativar produto geram `produto.atualizado`, pois
  mudam a situação do produto. A lista de eventos do spec (criação e atualização) é respeitada.
- **SKU normalizado**: o SKU é gravado em maiúsculas, sem espaços nas bordas, com índice
  único sobre o valor normalizado. Isso garante a regra "sem diferenciar maiúsculas" no banco.
- **Autoria (FR-022)**: cada produto guarda `criado_por` e `alterado_por` (id do operador
  autenticado, FK para a tabela de usuários). O id chega ao `ProdutoService` como parâmetro
  vindo da Api, sem nova abstração. Ver research.md, item 11.
- **Sessão de 8 h de inatividade (FR-015)**: autenticação por cookie com expiração deslizante.
  Substitui o JWT previsto antes, que não consegue medir inatividade sem estado no servidor.
  Ver research.md, item 5.
- **Banco indisponível (FR-023)**: falha de conexão vira `BancoIndisponivelException` na
  Infrastructure e responde 503 com Problem Details. Nada é gravado. Ver research.md, item 12.
- **Preço em R$ (FR-024)**: a Api devolve `preco` numérico e `precoExibicao` formatado em
  pt-BR ("R$ 1.234,56"). Ver research.md, item 13.
- **Testes da API sem pacote novo**: testes HTTP usam `HttpClient` contra a API do Compose.
  Ver Complexity Tracking, item 2.
- **Sessão (FR-015)**: a expiração é coberta por T070 (configuração do cookie) e por T071
  (ponta a ponta com `Sessao__TempoInatividade` curto).
- **Performance (SC-006)**: T072 mede a listagem com 10.000 produtos.
