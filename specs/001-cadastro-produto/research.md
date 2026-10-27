# Research: Cadastro de Produtos

Cada decisão segue o formato: **Decisão**, **Justificativa**, **Alternativas consideradas**.

## 1. Persistência: EF Core com PostgreSQL

- **Decisão**: EF Core 10 com `Npgsql.EntityFrameworkCore.PostgreSQL`, PostgreSQL 17 no Compose.
- **Justificativa**: exigido pela constituição (princípio IV). Consultas por LINQ são parametrizadas por padrão.
- **Alternativas consideradas**: Dapper (rejeitado: duplica o mapeamento e a constituição pede EF Core); PostgreSQL 16 (rejeitado: 17 é a versão atual com suporte longo).

## 2. Validação de SKU único: domínio e banco

- **Decisão**: duas camadas.
  1. **Domínio**: `Sku` normaliza (trim, maiúsculas) e valida formato (`^[A-Z0-9-]{1,30}$`). O `ProdutoService` consulta `IProdutoRepository.ExisteSkuAsync(sku, idIgnorado)` antes de gravar e lança `SkuDuplicadoException`.
  2. **Banco**: índice único em `produtos.sku`. Violação de chave única (SQLSTATE `23505`) vira `SkuDuplicadoException` no repositório.
- **Justificativa**: a consulta prévia dá a mensagem certa. O índice cobre a corrida entre dois operadores, conforme a clarificação Q2 (o primeiro vence).
- **Alternativas consideradas**: só índice único (rejeitado: viola o requisito de validação no domínio); só validação no código (rejeitado: não resolve a corrida).

## 3. Padrão Transactional Outbox

- **Decisão**:
  - Tabela `outbox_mensagens` no mesmo banco.
  - `OutboxInterceptor` (EF Core `SaveChangesInterceptor`) lê os eventos de domínio dos agregados alterados e grava as linhas de outbox **na mesma transação** do produto.
  - `OutboxWorker` (`BackgroundService`) consulta lotes com `SELECT ... FOR UPDATE SKIP LOCKED`, publica no RabbitMQ com *publisher confirms*, e marca `processado_em`.
  - Entrega **pelo menos uma vez**. Cada mensagem tem `messageId`, que o consumidor usa para idempotência.
  - Em falha, incrementa `tentativas` e grava `ultimo_erro`. Acima de `MaxTentativas` (configurável), a mensagem fica parada para inspeção.
- **Justificativa**: evita o problema de dupla escrita (gravar no banco e publicar, com uma das duas falhando). `SKIP LOCKED` permite mais de uma instância da API sem publicar duplicado em paralelo.
- **Alternativas consideradas**:
  - Publicar direto após o commit (rejeitado: dupla escrita, perda de evento se o processo cair entre os dois passos).
  - Debezium/CDC (rejeitado: mais um serviço no Compose para um volume pequeno).
  - Fila em memória (rejeitada: perde eventos em reinício).

## 4. Publicação no RabbitMQ

- **Decisão**: `RabbitMQ.Client` oficial (API assíncrona). Exchange `cadastro.produtos` do tipo `topic`, durável. Routing keys `produto.criado` e `produto.atualizado`. Mensagens persistentes, `content-type: application/json`, header `messageId`.
- **Justificativa**: exchange tipo topic permite que novos consumidores se liguem por padrão de chave sem mudar o publicador.
- **Alternativas consideradas**: MassTransit (rejeitado: framework de mensageria inteiro para dois tipos de evento, e a constituição proíbe dependência sem segundo uso); uma fila direta por evento (rejeitada: acopla o publicador aos consumidores).
- **Sem interface `IEventPublisher`**: a classe concreta `RabbitMqPublicador` é usada só pelo worker. Testada por integração contra o broker do Compose. O método `PublicarAsync` é `virtual`, para o Moq substituí-lo no teste do worker (T037). Criar interface seria abstração sem segundo uso (princípio VII).

## 5. Autenticação do operador

- **Decisão**: ASP.NET Core Identity (usuários no PostgreSQL) com autenticação por cookie e expiração deslizante de 8 h (`ExpireTimeSpan` = 8 h, `SlidingExpiration` = true). Cada requisição autenticada renova o prazo, o que implementa "8 horas de inatividade, renovadas a cada uso". Cookie `HttpOnly`, `SameSite=Strict`, `Secure` fora de desenvolvimento. Nome, prazo e demais opções em `SessaoOptions`. `POST /api/auth/logout` remove o cookie. Em desenvolvimento, um operador inicial é criado por configuração (`Seed__Operador__Email` e `Seed__Operador__Senha` via variável de ambiente).
- **Justificativa**: clarificação Q1 (login de operador, sem perfis) e clarificação de sessão (8 h de inatividade). A expiração por inatividade é nativa do cookie de autenticação, sem tabela extra de sessão. Identity já trata hash de senha.
- **Trade-off**: o cliente precisa guardar e reenviar o cookie. Ferramentas de linha de comando usam um arquivo de cookies (ver quickstart).
- **Proteção contra CSRF**: `SameSite=Strict` mais endpoints que só aceitam `application/json` em operações que alteram estado.
- **Testes**: T070 valida a configuração do cookie (`ExpireTimeSpan`, `SlidingExpiration`). T071 valida expiração e renovação ponta a ponta, com `Sessao__TempoInatividade` curto no Compose.
- **Alternativas consideradas**:
  - JWT de 8 h fixo (rejeitado: não renova a cada uso, viola a clarificação).
  - JWT com renovação por cabeçalho (rejeitado: token sem estado não sabe quando houve o último uso; depende de o cliente trocar o token a cada resposta, e um cliente que ignora a troca vira expiração absoluta).
  - Tabela de sessão no servidor (rejeitado: tabela e escrita a cada requisição sem ganho frente ao cookie nativo).
  - Chave de API (rejeitada: sem identificação por operador, o que a loja precisa).

## 6. Estados e eventos de inativação e reativação

- **Decisão**: inativar e reativar mudam a situação do produto e geram `produto.atualizado`. Não há evento próprio de inativação.
- **Justificativa**: a constituição e o spec pedem eventos de criação e atualização. Inativar é uma atualização do estado do produto. Um tipo a mais seria abstração sem consumidor.
- **Alternativas consideradas**: `produto.inativado` e `produto.reativado` (rejeitados: nenhum consumidor os pede, YAGNI).

## 7. Listagem com até 10.000 produtos

- **Decisão**: paginação por `pagina` e `tamanho` (padrão 50, máximo 200). Índice parcial `WHERE situacao = 'Ativo'` sobre `nome` para a ordenação padrão. Contagem total por `COUNT` com filtro.
- **Justificativa**: atende SC-006 (primeira página em menos de 2 s com 10.000 produtos) sem cache.
- **Alternativas consideradas**: carregar tudo (rejeitado: viola SC-006 à medida que o catálogo cresce); cache (rejeitado: complexidade sem necessidade medida, YAGNI).
- **Verificação**: T072 mede a primeira página com 10.000 produtos e confirma pelo plano de execução que o índice parcial é usado (SC-006).

## 8. Configuração e segredos

- **Decisão**: `IOptions<T>` com `ValidateDataAnnotations().ValidateOnStart()`. Variáveis de ambiente com `__` como separador (ex.: `ConnectionStrings__Padrao`, `RabbitMq__Host`). `.env` com segredos fica fora do Git; `.env.example` é commitado com valores de desenvolvimento não sensíveis.
- **Justificativa**: princípio III da constituição.
- **Alternativas consideradas**: Azure Key Vault ou similares (rejeitado: fora do escopo de desenvolvimento local).

## 9. Docker e ambiente de desenvolvimento

- **Decisão**: `docker-compose.yml` com três serviços:
  - `api`: build multi-stage a partir do `Dockerfile` da raiz (imagens `mcr.microsoft.com/dotnet/sdk:10.0` e `mcr.microsoft.com/dotnet/aspnet:10.0`), dependência de `postgres` e `rabbitmq` com `healthcheck`.
  - `postgres`: `postgres:17`, volume nomeado para dados.
  - `rabbitmq`: `rabbitmq:4-management`, painel na porta 15672.
- Migrations aplicadas na subida da API quando `Database__AplicarMigrationsNaInicializacao=true` (padrão em desenvolvimento).
- **Justificativa**: um comando sobe tudo, conforme o pedido.
- **Alternativas consideradas**: Aspire (rejeitado: nova ferramenta de orquestração fora do pedido); subir só banco e broker e rodar a API na máquina (mantido como opção no quickstart).

## 10. Testes

- **Decisão**:
  - Unitários (Domain, Application): xUnit, Moq para `IProdutoRepository`, FluentAssertions.
  - Integração (Infrastructure): xUnit contra o PostgreSQL e o RabbitMQ do Compose. Endereços vêm de variáveis de ambiente (`TESTES__ConnectionString`, `TESTES__RabbitMq__Host`). Sem Testcontainers.
  - HTTP (Api): `HttpClient` (BCL) contra a API do Compose, com cookie de sessão. Sem `WebApplicationFactory` e sem `Mvc.Testing`, por causa da stack da constituição (Complexity Tracking, item 2).
- **Justificativa**: princípio V da constituição. Testes de integração rodam contra os mesmos serviços do ambiente de desenvolvimento, sem provedor em memória.
- **Alternativas consideradas**: SQLite em memória (rejeitado: diferente do PostgreSQL, principalmente no índice parcial e no `SKIP LOCKED`); Testcontainers (rejeitado: biblioteca fora da constituição).

## 11. Autoria: criado por e alterado por (FR-022)

- **Decisão**: colunas `criado_por` e `alterado_por`, `varchar(450)`, `NOT NULL`, com FK para a tabela de usuários do Identity. Guardam o id do operador autenticado. `criado_em` e `atualizado_em` já existem e cobrem as datas. `criado_por` é gravado só na criação. `alterado_por` e `atualizado_em` mudam a cada alteração, incluindo inativação e reativação.
- **Como o id chega ao domínio**: a Api lê o id do usuário autenticado e o passa como parâmetro aos métodos do `ProdutoService`. Não há abstração nova para "usuário atual".
- **Justificativa**: rastreio mínimo pedido em Q2 (quem criou e quem fez a última alteração).
- **Alternativas consideradas**: guardar o e-mail (rejeitado: dado pessoal duplicado, e o registro fica errado se o e-mail mudar); tabela de auditoria com histórico completo (rejeitado: fora do pedido, YAGNI).

## 12. Banco indisponível (FR-023)

- **Decisão**: falhas de conexão do Npgsql (SQLSTATE classe `08`) e timeouts viram `BancoIndisponivelException` na Infrastructure. `ExcecoesParaProblemDetails` responde 503 com Problem Details. A transação não conclui e o EF Core faz rollback: nada é gravado.
- **Justificativa**: Q3 pediu recusa com mensagem, sem fila local nem sucesso falso.
- **Alternativas consideradas**: fila local para gravar depois (rejeitado: Q3, opção B); nova tentativa automática dentro da requisição (rejeitado: atrasa a resposta e mantém a incerteza para o operador).

## 13. Preço em real brasileiro (FR-024)

- **Decisão**: a Api devolve `preco` (número com duas casas) e `precoExibicao` (texto, ex.: `R$ 1.234,56`). A formatação usa a cultura `pt-BR` fixa, na camada Api. O valor é gravado sem formatação.
- **Justificativa**: o plano não tem interface gráfica. Formatar no servidor garante que qualquer cliente mostre o mesmo texto. A cultura é regra de negócio (moeda BRL), não configuração de ambiente, então não viola o princípio III.
- **Alternativas consideradas**: formatar só no cliente (rejeitado: nenhum cliente definido, risco de texto divergente); usar a cultura do servidor (rejeitado: muda com o ambiente e torna a saída não determinística).

## Resolução de NEEDS CLARIFICATION

Nenhum item pendente na Technical Context. Pontos que dependem de decisão do usuário, registrados no plan.md:
- `Mvc.Testing`: removido. Testes HTTP são black-box (Complexity Tracking, item 2).
- Porta de repositório: coberta pela exceção da constituição 1.1.0 (Complexity Tracking, item 1).
