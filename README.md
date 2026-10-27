# Cadastro de Produtos

> **Aviso:** este repositório é um projeto de **estudo** de Spec-Driven Development (SDD) com o [Spec-Kit](https://github.com/github/spec-kit) e o Claude Code. Os artefatos de especificação, plano e tarefas foram gerados e revisados com o apoio de IA. Não é um sistema em produção, e a implementação ainda não foi feita.

API REST para um operador de loja cadastrar, editar, listar, inativar e reativar produtos (nome, SKU único, preço e estoque).

> **Status:** o código ainda **não foi implementado**. Não há código de produção nem testes neste repositório. A especificação, o plano técnico, os contratos e as tarefas estão prontos em `specs/001-cadastro-produto/`. A implementação será iniciada em breve.

---

## Sumário

- [O que o sistema faz](#o-que-o-sistema-faz)
- [Stack técnica planejada](#stack-técnica-planejada)
- [Decisões técnicas](#decisões-técnicas)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Endpoints planejados](#endpoints-planejados)
- [Como o projeto foi construído (Spec-Kit e SDD)](#como-o-projeto-foi-construído-spec-kit-e-sdd)
- [Artefatos de especificação](#artefatos-de-especificação)
- [Pendências](#pendências)

---

## O que o sistema faz

O operador da loja:

1. **Cadastra** um produto com nome, SKU, preço e estoque. Todo produto novo nasce **ativo**.
2. **Lista** os produtos ativos em uma listagem paginada (50 por página, máximo 200), em ordem alfabética pelo nome.
3. **Edita** nome, SKU, preço ou estoque de um produto ativo.
4. **Inativa** um produto. Ele some da listagem padrão, mas nenhum dado é apagado.
5. **Reativa** um produto inativo, que volta à listagem com os mesmos dados.
6. **Consulta** qualquer produto pelo identificador, ativo ou inativo.

Regras principais:

- **SKU único** entre todos os produtos, ativos ou inativos. Comparação sem diferenciar maiúsculas/minúsculas e sem espaços nas bordas. Aceita só letras, números e hífen, com até 30 caracteres.
- **Preço** não negativo, com até duas casas decimais. Zero é aceito.
- **Estoque** inteiro não negativo. Zero é aceito.
- **Nome** obrigatório, com até 150 caracteres. Nome só com espaços é recusado.
- **Login obrigatório** para qualquer operação de produto. A sessão expira após 8 horas de inatividade, renovadas a cada uso. Cada operador tem conta própria; não há perfis diferentes.
- **Autoria**: cada produto guarda quem o criou e quem fez a última alteração, com data.
- **Eventos**: cada criação gera `produto.criado`. Cada edição, inativação ou reativação aceita gera `produto.atualizado`. Os eventos são publicados no RabbitMQ pelo padrão Transactional Outbox, com entrega pelo menos uma vez.
- **Preço exibido** no formato `R$ 1.234,56`. O valor é gravado sem formatação.
- **Banco indisponível**: a operação é recusada com mensagem de indisponibilidade, e nada é gravado.
- **Concorrência**: se dois operadores salvarem o mesmo SKU ao mesmo tempo, o primeiro vence e o segundo recebe a mensagem de SKU em uso.

Volume previsto: até **10.000 produtos** (ativos e inativos somados), com a primeira página da listagem abaixo de 2 segundos.

---

## Stack técnica planejada

- .NET 10 (`net10.0`), C# 14
- ASP.NET Core 10 (Minimal APIs), com autenticação por cookie e expiração deslizante (ASP.NET Core Identity)
- EF Core 10 com Npgsql, sobre PostgreSQL 17
- RabbitMQ 4, com `RabbitMQ.Client` (cliente oficial); exchange `cadastro.produtos` do tipo `topic`
- Testes com xUnit, Moq e FluentAssertions
- Docker Compose com três serviços: `api`, `postgres` e `rabbitmq`

---

## Decisões técnicas

Decisões registradas em `specs/001-cadastro-produto/research.md`, cada uma com justificativa e alternativas rejeitadas:

| Decisão | Motivo |
|---|---|
| Autenticação por cookie com expiração deslizante, em vez de JWT | JWT sem estado não mede inatividade sem estado no servidor |
| Inativar e reativar geram `produto.atualizado` | A spec pede eventos de criação e atualização; um tipo extra não teria consumidor |
| Outbox gravado na mesma transação do produto | Evita dupla escrita: sem isso, o produto pode ser gravado e o evento perdido |
| SKU normalizado (maiúsculas, sem espaços) com índice único sobre o valor normalizado | Garante a regra de unicidade sem diferenciar maiúsculas dentro do próprio banco |
| Formatação `R$` na API, com cultura `pt-BR` fixa | Qualquer cliente recebe o mesmo texto; moeda é regra de negócio, não configuração de ambiente |
| Testes contra PostgreSQL e RabbitMQ reais do Compose, sem Testcontainers nem provedor em memória | Provedores em memória se comportam diferente do PostgreSQL (índice parcial, `SKIP LOCKED`) |
| Sem interface `IEventPublisher` | Só o worker usa o publicador; interface sem segundo uso não se justifica |
| Sem `Microsoft.AspNetCore.Mvc.Testing` | Fora da lista de bibliotecas de teste definida pelo projeto; testes HTTP usam `HttpClient` |

---

## Estrutura do repositório

Estado atual (somente documentação):

```text
.
├── specs/
│   └── 001-cadastro-produto/
│       ├── spec.md
│       ├── plan.md
│       ├── research.md
│       ├── data-model.md
│       ├── quickstart.md
│       ├── tasks.md
│       ├── contracts/
│       └── checklists/
└── README.md
```

Estrutura planejada, ainda não criada (seção *Source Code* do `plan.md`):

```text
Cadastro.sln
Dockerfile
docker-compose.yml
.env.example
src/
├── Cadastro.Domain/           # agregado Produto, Sku, eventos, exceções, porta IProdutoRepository
├── Cadastro.Application/      # ProdutoService (um método por caso de uso), DTOs
├── Cadastro.Infrastructure/   # EF Core, Npgsql, outbox, RabbitMQ, Identity, opções
└── Cadastro.Api/              # Minimal APIs, autenticação, erros em Problem Details
tests/
├── Cadastro.Domain.Tests/
├── Cadastro.Application.Tests/
├── Cadastro.Infrastructure.Tests/
└── Cadastro.Api.Tests/
```

---

## Endpoints planejados

| Método | Rota | Função |
|---|---|---|
| `POST` | `/api/auth/login` | Login do operador (cookie de sessão) |
| `POST` | `/api/auth/logout` | Encerra a sessão |
| `POST` | `/api/produtos` | Cadastrar |
| `GET` | `/api/produtos?pagina=&tamanho=` | Listar produtos ativos |
| `GET` | `/api/produtos/{id}` | Consultar, ativo ou inativo |
| `PUT` | `/api/produtos/{id}` | Editar |
| `POST` | `/api/produtos/{id}/inativacao` | Inativar |
| `POST` | `/api/produtos/{id}/reativacao` | Reativar |
| `GET` | `/health` | Saúde da aplicação |

---

## Como o projeto foi construído (Spec-Kit e SDD)

O projeto segue **Spec-Driven Development (SDD)** com o [Spec-Kit](https://github.com/github/spec-kit) e o Claude Code. A regra é: primeiro o contrato escrito (o quê e por quê), depois o plano (como), depois as tarefas, e só então o código. Cada etapa gera um artefato Markdown versionado no Git.

**Instalação** (feita uma vez, na raiz do repositório):

```bash
uvx --from git+https://github.com/github/spec-kit.git specify init --here --integration claude --script sh --extension git
```

Isso instala os comandos `/speckit-...` como skills do Claude Code (`.claude/skills/`), os modelos e scripts em `.specify/`, e a extensão git, que cria a branch `001-...` a cada funcionalidade. O commit automático ficou desligado (`auto_commit.default: false`); os commits são feitos manualmente.

**Etapas executadas, na ordem:**

| # | Comando | Artefato gerado | O que foi feito |
|---|---|---|---|
| 1 | `/speckit-constitution` | `.specify/memory/constitution.md` | Princípios do projeto: Clean Architecture com dependência em um sentido, DDD, configuração externa, EF Core com PostgreSQL, teste em toda mudança, e "pronto só com build e testes verdes". Versão 1.1.0, após uma emenda que resolveu um conflito apontado pela análise. |
| 2 | `/speckit-specify` | `specs/001-cadastro-produto/spec.md` + `checklists/requirements.md` | Especificação em histórias de usuário priorizadas (P1, P2...), requisitos funcionais, critérios de sucesso e casos de borda. Sem tecnologia na spec. |
| 3 | `/speckit-clarify` | seção `## Clarifications` do `spec.md` | Perguntas de múltipla escolha respondidas e aplicadas aos requisitos afetados (ex.: regras de SKU, inativação, estoque). |
| 4 | `/speckit-plan` | `plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md` | Stack (.NET 10, EF Core, PostgreSQL 17, RabbitMQ 4), arquitetura em 4 projetos, decisões com alternativas rejeitadas, contrato OpenAPI e contrato dos eventos. |
| 5 | `/speckit-tasks` | `tasks.md` | 72 tarefas por fase e história de usuário, com IDs (T001...), caminhos de arquivo e marcação `[P]` para paralelismo. |
| 6 | `/speckit-analyze` | relatório (sem alterar arquivos) | Conferência de coerência entre spec, plano, tarefas e constituição. Os apontamentos foram tratados antes de seguir. |
| 7 | `/speckit-checklist` | `checklists/` | Checklists de qualidade dos requisitos. Os itens de prontidão ainda estão em aberto. |

**Próximas etapas (ainda não executadas):**

- `/speckit-implement`, em blocos com checkpoint: primeiro a fase de base, depois cada história, sempre com build e testes verdes antes de liberar o bloco seguinte.
- `/speckit-converge`, ao fim da implementação, para acrescentar ao `tasks.md` qualquer item que tenha ficado sem código.
- Revisão de commit e PR referenciando `specs/001-cadastro-produto/`.

**Regras de trabalho com o agente** estão no `CLAUDE.md`: mudanças pequenas, nada de migration sem pedido, nada de pacote NuGet sem justificativa, e um formato fixo de resposta ao terminar (o que mudou, arquivos, como validar, riscos). A seção "Lições" cresce quando um erro se repete.

**Diferença:** a constituição diz *o que nunca muda*. O `CLAUDE.md` diz *como trabalhar neste repositório*. A spec diz *o quê e por quê*. O plano diz *como*.

---

## Artefatos de especificação

| Arquivo | Conteúdo |
|---|---|
| `specs/001-cadastro-produto/spec.md` | Requisitos funcionais, histórias de usuário, critérios de sucesso e clarificações |
| `specs/001-cadastro-produto/plan.md` | Plano técnico, verificação de conformidade e justificativas de complexidade |
| `specs/001-cadastro-produto/research.md` | Decisões técnicas com alternativas consideradas |
| `specs/001-cadastro-produto/data-model.md` | Modelo de dados, índices, regras de domínio e migrations |
| `specs/001-cadastro-produto/contracts/produtos-api.yaml` | Contrato HTTP (OpenAPI 3.1) |
| `specs/001-cadastro-produto/contracts/eventos-produto.md` | Contrato dos eventos no RabbitMQ |
| `specs/001-cadastro-produto/quickstart.md` | Roteiro para subir e validar o ambiente |
| `specs/001-cadastro-produto/tasks.md` | Lista de 72 tarefas por fase e história, todas pendentes |
| `specs/001-cadastro-produto/checklists/` | Checklists de qualidade da especificação e de prontidão |

---

## Pendências

- **Implementação pendente.** Nenhum código de produção ou teste existe ainda. Nada descrito como funcionando foi executado.
- **Checklist de prontidão aberto.** `checklists/readiness.md` tem itens sem marcação, pendentes de revisão.
- **Comentário de rascunho na constituição.** O bloco de relatório de sincronização no topo de `.specify/memory/constitution.md` deve ser removido antes do primeiro commit.
- **Sem commits de implementação.** Os artefatos de especificação já estão versionados na branch `001-cadastro-produto`; o código ainda não.

Quando a implementação começar, o ambiente deve subir com `docker compose up -d --build` (ver `quickstart.md`) e os testes com `dotnet test Cadastro.sln`.
