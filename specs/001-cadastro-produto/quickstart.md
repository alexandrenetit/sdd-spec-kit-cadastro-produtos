# Quickstart: Cadastro de Produtos

Guia para subir o ambiente de desenvolvimento e validar a feature de ponta a ponta.
Contratos: [produtos-api.yaml](./contracts/produtos-api.yaml) e [eventos-produto.md](./contracts/eventos-produto.md).

## Pré-requisitos

- Docker Desktop (ou Docker Engine no WSL2) com Compose v2.
- .NET 10 SDK, para rodar os testes fora do contêiner.
- Portas livres: `8080` (API), `5432` (PostgreSQL), `5672` e `15672` (RabbitMQ).

## 1. Configurar o ambiente

Copiar o modelo de variáveis e revisar os valores de desenvolvimento:

```bash
cp .env.example .env
```

Variáveis principais (todas lidas por `IOptions`, nenhuma fixa no código):

| Variável | Uso |
|----------|-----|
| `POSTGRES_PASSWORD` | senha do PostgreSQL de desenvolvimento |
| `ConnectionStrings__Padrao` | conexão da API (montada pelo Compose) |
| `RabbitMq__Host`, `RabbitMq__Usuario`, `RabbitMq__Senha` | broker |
| `Sessao__NomeCookie` | nome do cookie de sessão (padrão `Cadastro.Sessao`) |
| `Sessao__TempoInatividade` | tempo de inatividade até expirar (padrão `08:00:00`, FR-015) |
| `Seed__Operador__Email`, `Seed__Operador__Senha` | operador criado na primeira subida |
| `Outbox__IntervaloPolling`, `Outbox__MaxTentativas` | worker do outbox |

## 2. Subir todos os recursos

```bash
docker compose up -d --build
docker compose ps
```

Resultado esperado: `api`, `postgres` e `rabbitmq` com status `running` e `healthy`.

Na primeira subida, a API aplica as migrations (`Database__AplicarMigrationsNaInicializacao=true`)
e cria o operador de `Seed__*`.

Verificar a API:

```bash
curl -s http://localhost:8080/health
```

Esperado: `Healthy`.

## 3. Validar o fluxo do operador

A sessão usa cookie. O `curl` guarda o cookie em `cookies.txt` com `-c` e envia com `-b`.

Login (esperado `200`, cookie gravado em `cookies.txt`):

```bash
curl -s -c cookies.txt -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"<Seed__Operador__Email>","senha":"<Seed__Operador__Senha>"}'
```

Sem sessão, a API responde `401` (FR-015):

```bash
curl -i http://localhost:8080/api/produtos
```

Cadastrar um produto (esperado `201`):

```bash
curl -s -b cookies.txt -X POST http://localhost:8080/api/produtos \
  -H "Content-Type: application/json" \
  -d '{"nome":"Camiseta Básica","sku":"cam-bas-001","preco":49.90,"estoque":120}'
```

Conferir na resposta:
- `sku` volta como `CAM-BAS-001` (normalizado).
- `precoExibicao` é `R$ 49,90` (FR-024).
- `criadoPor` e `alteradoPor` têm o mesmo id do operador logado (FR-022).

Repetir o mesmo cadastro (esperado `409`, SKU em uso, FR-004).

Preço negativo (esperado `422`, campo `preco`, FR-005):

```bash
curl -i -b cookies.txt -X POST http://localhost:8080/api/produtos \
  -H "Content-Type: application/json" \
  -d '{"nome":"Teste","sku":"NEG-1","preco":-1,"estoque":1}'
```

Listar (esperado `200`, só produtos ativos, em ordem alfabética, 50 por página, FR-012 e FR-020):

```bash
curl -s -b cookies.txt "http://localhost:8080/api/produtos?pagina=1&tamanho=50"
```

Inativar, listar de novo (o produto some), depois reativar (volta). Os dados continuam os mesmos (FR-010, FR-011, FR-017):

```bash
curl -s -b cookies.txt -X POST http://localhost:8080/api/produtos/<ID>/inativacao
curl -s -b cookies.txt -X POST http://localhost:8080/api/produtos/<ID>/reativacao
```

Consultar um produto inativo pelo id (FR-021):

```bash
curl -s -b cookies.txt http://localhost:8080/api/produtos/<ID>
```

Encerrar a sessão (esperado `204`, o cookie é removido):

```bash
curl -i -b cookies.txt -c cookies.txt -X POST http://localhost:8080/api/auth/logout
```

## 4. Validar banco indisponível (FR-023)

```bash
docker compose stop postgres
curl -i -b cookies.txt -X POST http://localhost:8080/api/produtos \
  -H "Content-Type: application/json" \
  -d '{"nome":"Teste Queda","sku":"QUEDA-1","preco":10,"estoque":1}'
docker compose start postgres
```

Esperado: `503` com Problem Details. Depois do `start`, o mesmo cadastro funciona (nada foi
gravado na queda). Confirmar que `QUEDA-1` não existe na listagem. Se o operador ainda não
tiver o cookie válido após a parada, faça login de novo.

## 5. Validar o outbox e o RabbitMQ

1. Abrir `http://localhost:15672` (usuário e senha de `RabbitMq__*`).
2. Em **Exchanges**, confirmar `cadastro.produtos` do tipo `topic`.
3. Criar uma fila de teste com binding em `produto.criado` e `produto.atualizado`.
4. Repetir a criação e a edição pela API. A fila recebe uma mensagem por operação com o envelope de [eventos-produto.md](./contracts/eventos-produto.md).
5. Verificar no banco que as linhas de `outbox_mensagens` estão processadas (`processado_em` preenchido):

```bash
docker compose exec postgres psql -U cadastro -d cadastro -c \
  "SELECT tipo, tentativas, processado_em IS NOT NULL AS publicado FROM outbox_mensagens ORDER BY ocorrido_em;"
```

Teste de falha do broker: `docker compose stop rabbitmq`, criar um produto, conferir que a linha
fica com `publicado = f` e `tentativas` sobe. Depois `docker compose start rabbitmq` e conferir
que a linha é publicada (FR-019, SC-007).

## 6. Rodar os testes

Os testes de integração e de API usam o PostgreSQL e o RabbitMQ do Compose.

```bash
docker compose up -d postgres rabbitmq
dotnet build Cadastro.sln
dotnet test Cadastro.sln
```

Variáveis de teste (em `.env` ou no ambiente do shell):

| Variável | Uso |
|----------|-----|
| `TESTES__ConnectionString` | banco dos testes de integração (use um banco separado, ex.: `cadastro_testes`) |
| `TESTES__RabbitMq__Host` | broker dos testes de integração |

Critério de conclusão: `dotnet build` sem erros e `dotnet test` com todos os testes passando
(princípio VI da constituição).

## 7. Derrubar o ambiente

```bash
docker compose down
```

Cuidado: `docker compose down -v` apaga o volume do PostgreSQL e todos os dados de desenvolvimento.
Não usar sem confirmação explícita.
