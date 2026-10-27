# Data Model: Cadastro de Produtos

## Entidade: Produto (agregado)

Origem: FR-001 a FR-017 e Key Entities do spec.

| Campo (domínio) | Coluna | Tipo PostgreSQL | Regra de domínio | Restrição de banco |
|-----------------|--------|-----------------|------------------|--------------------|
| `Id` | `id` | `uuid` | gerado na criação | chave primária |
| `Nome` | `nome` | `varchar(150)` | obrigatório após trim; máximo 150 caracteres (FR-016) | `NOT NULL`, `CHECK (length(trim(nome)) > 0)` |
| `Sku` | `sku` | `varchar(30)` | normalizado em maiúsculas, sem espaços nas bordas; formato `^[A-Z0-9-]{1,30}$` (FR-003, FR-016) | `NOT NULL`, **índice único** `ux_produtos_sku` |
| `Preco` | `preco` | `numeric(12,2)` | não negativo; até duas casas decimais (FR-005) | `NOT NULL`, `CHECK (preco >= 0)` |
| `Estoque` | `estoque` | `integer` | inteiro não negativo (FR-006) | `NOT NULL`, `CHECK (estoque >= 0)` |
| `Situacao` | `situacao` | `varchar(10)` | `Ativo` ou `Inativo`; novo produto nasce `Ativo` (FR-007) | `NOT NULL`, `CHECK (situacao IN ('Ativo','Inativo'))` |
| `CriadoEm` | `criado_em` | `timestamptz` | definido na criação | `NOT NULL` |
| `AtualizadoEm` | `atualizado_em` | `timestamptz` | atualizado a cada alteração, inclusive inativação e reativação (FR-022) | `NOT NULL` |
| `CriadoPor` | `criado_por` | `varchar(450)` | id do operador que cadastrou, gravado só na criação (FR-022) | `NOT NULL`, FK para a tabela de usuários do Identity (`AspNetUsers.Id`), `ON DELETE RESTRICT` |
| `AlteradoPor` | `alterado_por` | `varchar(450)` | id do operador da última alteração, incluindo inativação e reativação (FR-022) | `NOT NULL`, FK para `AspNetUsers.Id`, `ON DELETE RESTRICT` |

**Campo derivado (não armazenado)**: `PrecoExibicao`, texto no formato `R$ 1.234,56`, gerado pela Api
a partir de `Preco` com a cultura `pt-BR` (FR-024). Nenhuma coluna guarda esse texto.

### Índices

- `ux_produtos_sku` — `UNIQUE (sku)`. Garante FR-003 no banco, inclusive em corrida (clarificação Q2).
- `ix_produtos_listagem_ativos` — `(nome) WHERE situacao = 'Ativo'`. Índice parcial para a listagem padrão (FR-012, SC-006).

### Transições de situação

```text
            cadastrar
  (novo) ───────────────► Ativo ◄──────────────┐
                           │  │                │
                 inativar  │  │ reativar       │
                           ▼  │                │
                        Inativo ───────────────┘
```

- `Ativo → Inativo`: operador inativa (FR-010). Dados preservados (FR-011).
- `Inativo → Ativo`: operador reativa (FR-017). Dados preservados.
- `Inativo → Inativo` (inativar de novo): nenhuma alteração; a API responde 200 com o produto atual e não gera evento (premissa de Edge Cases).
- Edição de produto `Inativo` é recusada (Edge Cases).

### Regras de domínio

1. `Produto.Criar(nome, sku, preco, estoque)` valida todos os campos e lança `DadosInvalidosException` com o nome do campo.
2. `Sku` é value object: normaliza na construção; formato inválido lança `DadosInvalidosException`.
3. `Produto.Atualizar(...)` só é permitido com `Situacao == Ativo`.
4. Cada mudança relevante registra um evento de domínio (`ProdutoCriado` ou `ProdutoAtualizado`) no agregado. O evento é publicado pelo outbox.
5. Unicidade de SKU é checada por `ProdutoService` via `IProdutoRepository.ExisteSkuAsync(sku, idIgnorado)`. A checagem é repetida pelo índice único no commit.
6. Autoria: cada operação recebe `operadorId` vindo da Api. Cadastro define `CriadoPor` e `AlteradoPor` com esse id. Edição, inativação e reativação definem `AlteradoPor` e atualizam `AtualizadoEm`. Nenhuma operação recebe `operadorId` vazio.
7. Banco indisponível: a falha vem da Infrastructure como `BancoIndisponivelException`. O domínio não trata esse caso. Nenhum estado parcial é gravado, porque a transação faz rollback (FR-023).

## Entidade: OutboxMensagem

Origem: padrão Transactional Outbox (research.md, item 3).

| Campo | Coluna | Tipo | Regra |
|-------|--------|------|-------|
| `Id` | `id` | `uuid` | `messageId` do evento; chave de idempotência do consumidor |
| `Tipo` | `tipo` | `varchar(100)` | `produto.criado` ou `produto.atualizado` (routing key) |
| `AgregadoId` | `agregado_id` | `uuid` | id do produto |
| `Payload` | `payload` | `jsonb` | envelope do evento (ver contracts/eventos-produto.md) |
| `OcorridoEm` | `ocorrido_em` | `timestamptz` | momento do evento no domínio |
| `ProcessadoEm` | `processado_em` | `timestamptz NULL` | preenchido após publisher confirm |
| `Tentativas` | `tentativas` | `integer` | incrementado a cada falha de publicação |
| `UltimoErro` | `ultimo_erro` | `text NULL` | mensagem da última falha |

### Índices

- `ix_outbox_pendentes` — `(ocorrido_em) WHERE processado_em IS NULL`. Usado pelo worker.

### Ciclo de vida

```text
gravada (processado_em NULL, tentativas 0)
   │  worker publica + publisher confirm
   ▼
processada (processado_em preenchido)

gravada → falha → tentativas++ → nova tentativa
            └── tentativas >= MaxTentativas → fica parada (inspeção manual)
```

## Identity (ASP.NET Core Identity)

Tabelas padrão `AspNetUsers`, `AspNetRoles`, etc., no mesmo banco. Usadas apenas para o login do operador (FR-015). Sem perfis nesta versão (clarificação Q1). A sessão é um cookie de autenticação com expiração deslizante de 8 h (clarificação de sessão). Não há tabela de sessão.

## Migrations

- Uma migration inicial cria `produtos` (com FKs de `criado_por` e `alterado_por` para `AspNetUsers`), `outbox_mensagens` e as tabelas do Identity. As FKs exigem que as tabelas do Identity sejam criadas na mesma migration ou antes.
- Os `CHECK` e índices parciais são definidos na configuração do EF Core com `HasCheckConstraint` e `HasIndex(...).HasFilter(...)`.
- A migration nunca é editada depois de commitada; mudanças viram nova migration.
