# Contrato de Eventos: Produto (RabbitMQ)

Origem: FR-013 e decisão de outbox em research.md (itens 3, 4 e 6).

## Broker e roteamento

| Item | Valor |
|------|-------|
| Exchange | `cadastro.produtos` |
| Tipo | `topic` |
| Durável | sim |
| Routing key `produto.criado` | criação de produto (FR-007) |
| Routing key `produto.atualizado` | edição, inativação e reativação |
| Entrega | mensagem persistente, publisher confirms ativos |
| `content-type` | `application/json` |

Consumidores criam a própria fila e fazem binding na exchange com a routing key de interesse.
A fila não é declarada pela API.

## Headers da mensagem

| Header | Valor |
|--------|-------|
| `messageId` | GUID da linha em `outbox_mensagens`. Chave de idempotência. |
| `tipo` | `produto.criado` ou `produto.atualizado` |
| `schemaVersion` | `1` |

## Envelope (corpo JSON)

```json
{
  "messageId": "3f2a8c1e-7b4d-4e2a-9c1f-5a6b7c8d9e0f",
  "tipo": "produto.criado",
  "schemaVersion": 1,
  "ocorridoEm": "2026-10-27T14:03:22Z",
  "agregadoId": "9b1d2e3f-4a5b-4c6d-8e7f-0a1b2c3d4e5f",
  "dados": {
    "id": "9b1d2e3f-4a5b-4c6d-8e7f-0a1b2c3d4e5f",
    "nome": "Camiseta Básica",
    "sku": "CAM-BAS-001",
    "preco": 49.90,
    "estoque": 120,
    "situacao": "Ativo"
  }
}
```

`dados` contém o estado completo do produto no momento do evento. O consumidor não precisa
consultar a API para atualizar o próprio cache.

## Garantias e regras para consumidores

- **Pelo menos uma vez**: o mesmo `messageId` pode chegar mais de uma vez, por reentrega após falha de rede. Consumidores MUST descartar duplicatas pelo `messageId`.
- **Ordem**: não garantida entre produtos diferentes. Para o mesmo produto, consumidores usam `ocorridoEm` ou o estado completo de `dados` para decidir a versão mais recente.
- **Compatibilidade**: adicionar campos em `dados` é mudança compatível. Remover ou renomear campos exige `schemaVersion: 2`.
- **Atraso**: o evento sai após o commit do produto, com latência de até um intervalo de polling do worker (configurável em `Outbox__IntervaloPolling`).

## Eventos gerados por operação

| Operação (spec) | Routing key | Observação |
|-----------------|-------------|------------|
| Cadastrar (US1) | `produto.criado` | uma vez por produto |
| Editar (US3) | `produto.atualizado` | só se a edição for aceita |
| Inativar (US4) | `produto.atualizado` | `situacao` = `Inativo` |
| Reativar (FR-017) | `produto.atualizado` | `situacao` = `Ativo` |
| Inativar produto já inativo | nenhum | nenhuma alteração de estado |
| Edição recusada (SKU repetido, preço negativo) | nenhum | nada é gravado |
