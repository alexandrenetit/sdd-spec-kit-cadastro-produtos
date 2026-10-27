# Feature Specification: Cadastro de Produtos

**Feature Branch**: `001-cadastro-produto`

**Created**: 2026-10-27

**Status**: Draft

**Input**: User description: "Cadastro de produtos para uma loja. O operador cadastra, edita, lista e inativa produtos com nome, SKU único, preço e estoque. Produto inativo não aparece na listagem padrão. Não pode existir preço negativo nem SKU repetido."

## Clarifications

### Session 2026-10-27

- Q: Quem usa o sistema precisa fazer login antes de cadastrar ou editar produtos? → A: Sim, login com conta de operador, sem perfis diferentes
- Q: Se dois operadores salvarem o mesmo SKU ao mesmo tempo, o que deve acontecer? → A: O primeiro salvamento vence; o segundo recebe a mensagem de SKU já em uso

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Cadastrar produto (Priority: P1)

O operador da loja informa nome, SKU, preço e estoque de um novo produto e o salva. O produto passa a existir no catálogo como ativo.

**Why this priority**: sem cadastro não há catálogo. É a base de todas as outras histórias.

**Independent Test**: o operador cadastra um produto com dados válidos e confirma que ele foi salvo com os valores informados.

**Acceptance Scenarios**:

1. **Given** o operador está na tela de cadastro, **When** informa nome, SKU, preço e estoque válidos e salva, **Then** o produto é salvo como ativo e seus dados são exibidos sem alteração.
2. **Given** já existe um produto com o SKU `ABC-001`, **When** o operador tenta cadastrar outro produto com o mesmo SKU, **Then** o sistema recusa o cadastro, informa que o SKU já está em uso e não salva nenhum registro.
3. **Given** o operador está na tela de cadastro, **When** informa preço negativo, **Then** o sistema recusa o cadastro, informa que o preço não pode ser negativo e não salva nenhum registro.
4. **Given** o operador deixa nome, SKU, preço ou estoque em branco, **When** tenta salvar, **Then** o sistema recusa o cadastro e indica o campo obrigatório que falta.

---

### User Story 2 - Listar produtos ativos (Priority: P1)

O operador abre a listagem padrão de produtos e vê apenas os produtos ativos do catálogo.

**Why this priority**: a listagem é a forma principal de consulta do catálogo e define a regra de visibilidade de produtos inativos.

**Independent Test**: com produtos ativos e inativos cadastrados, abrir a listagem padrão e conferir que só os ativos aparecem.

**Acceptance Scenarios**:

1. **Given** existem 3 produtos ativos e 2 inativos, **When** o operador abre a listagem padrão, **Then** são exibidos somente os 3 produtos ativos, com nome, SKU, preço e estoque.
2. **Given** não existe nenhum produto ativo, **When** o operador abre a listagem padrão, **Then** o sistema exibe uma mensagem informando que não há produtos cadastrados.
3. **Given** um produto ativo aparece na listagem, **When** o operador o inativa (User Story 4), **Then** ele deixa de aparecer na listagem padrão sem perda de dados.

---

### User Story 3 - Editar produto (Priority: P2)

O operador altera nome, SKU, preço ou estoque de um produto ativo existente.

**Why this priority**: corrige erros de cadastro e mantém o catálogo atualizado, mas o catálogo já é útil sem edição.

**Independent Test**: editar um produto ativo, salvar e confirmar que os novos valores aparecem na listagem.

**Acceptance Scenarios**:

1. **Given** um produto ativo existe, **When** o operador altera o preço para um valor válido e salva, **Then** o novo preço passa a ser exibido na listagem.
2. **Given** existem os produtos A (SKU `ABC-001`) e B (SKU `XYZ-002`), **When** o operador tenta mudar o SKU de B para `ABC-001`, **Then** o sistema recusa a alteração, informa que o SKU já está em uso e mantém o SKU original de B.
3. **Given** um produto ativo com SKU `ABC-001`, **When** o operador salva a edição mantendo o mesmo SKU, **Then** o sistema aceita a alteração, pois o SKU não está em uso por outro produto.
4. **Given** o operador informa preço negativo na edição, **When** tenta salvar, **Then** o sistema recusa a alteração e mantém o preço anterior.

---

### User Story 4 - Inativar produto (Priority: P2)

O operador inativa um produto que não deve mais ser vendido ou listado. O produto deixa de aparecer na listagem padrão, mas seus dados são preservados.

**Why this priority**: permite retirar produtos do catálogo sem apagar histórico. Depende da listagem (User Story 2) para ter efeito visível.

**Independent Test**: inativar um produto ativo e confirmar que ele sai da listagem padrão e que seus dados continuam guardados.

**Acceptance Scenarios**:

1. **Given** um produto ativo aparece na listagem, **When** o operador o inativa, **Then** o produto some da listagem padrão e o sistema confirma a inativação.
2. **Given** um produto inativo, **When** o operador consulta o produto pelo identificador (FR-021), **Then** o nome, SKU, preço e estoque permanecem iguais aos de antes da inativação.
3. **Given** um produto inativo, **When** o operador tenta cadastrar um novo produto com o mesmo SKU, **Then** o sistema recusa o cadastro, pois o SKU continua em uso.
4. **Given** um produto inativo por engano, **When** o operador o reativa, **Then** o produto volta a aparecer na listagem padrão com nome, SKU, preço e estoque iguais aos de antes da inativação.

---

### Edge Cases

- Preço zero: aceito, pois não é negativo.
- Estoque zero: aceito, representa produto sem unidades disponíveis.
- SKU com diferença só de maiúsculas/minúsculas ou espaços nas bordas (`abc-001 ` vs `ABC-001`): tratado como o mesmo SKU e recusado.
- Preço com mais de duas casas decimais (ex.: `10,999`): recusado; o preço deve ter no máximo duas casas decimais.
- Estoque negativo ou fracionado (ex.: `2,5` unidades): recusado.
- Inativar um produto que já está inativo: nenhuma mudança de estado; o sistema informa que o produto já está inativo.
- Editar um produto inativo: não permitido nesta versão.
- Nome composto só de espaços: tratado como campo em branco e recusado.
- Produto inativado por outro operador enquanto a edição está em andamento: a edição é recusada com aviso de que o produto foi inativado, e nada é gravado (FR-009).
- Banco de dados indisponível no momento de salvar: a operação é recusada com mensagem de indisponibilidade e nada é gravado (FR-023).
- Dois operadores salvam o mesmo SKU ao mesmo tempo: o primeiro salvamento é aceito e o segundo é recusado com a mensagem de SKU já em uso, sem registro duplicado.
- Q: Qual o volume de produtos que o catálogo deve suportar nesta versão? → A: Até 10.000 produtos (ativos e inativos somados)
- Q: Qual é o tamanho máximo do nome e quais caracteres o SKU pode ter? → A: Nome até 150 caracteres; SKU até 30 caracteres, apenas letras, números e hífen
- Q: Um produto inativado por engano deve poder voltar a ficar ativo? → A: Sim, o operador pode reativar um produto inativo
- Q: Depois de quanto tempo sem uso o operador precisa fazer login de novo? → A: 8 horas de inatividade, renovadas a cada uso
- Q: Quando um produto é cadastrado ou editado, o sistema deve guardar qual operador fez a alteração? → A: Sim, guardar quem criou e quem fez a última alteração
- Q: Se o banco de dados estiver indisponível ao salvar, o que o sistema deve fazer? → A: Recusar a operação com mensagem de indisponibilidade; nada é gravado
- Q: Se um produto é inativado enquanto outro operador edita, o que acontece ao salvar a edição? → A: Recusar a edição e informar que o produto foi inativado; nada é gravado
- Q: Em qual formato o preço deve aparecer para o operador? → A: Real brasileiro com símbolo, formato R$ 1.234,56

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST permitir cadastrar um produto informando nome, SKU, preço e estoque.
- **FR-002**: O sistema MUST exigir nome, SKU, preço e estoque em todo cadastro e edição.
- **FR-003**: O sistema MUST garantir que cada SKU seja único entre todos os produtos, ativos ou inativos, desconsiderando diferenças de maiúsculas/minúsculas e espaços nas bordas.
- **FR-004**: O sistema MUST recusar cadastro ou edição com SKU já usado por outro produto e informar o motivo.
- **FR-005**: O sistema MUST recusar preço negativo em cadastro e edição.
- **FR-006**: O sistema MUST recusar estoque negativo ou fracionado em cadastro e edição.
- **FR-007**: O sistema MUST salvar todo produto novo como ativo.
- **FR-008**: O operador MUST poder editar nome, SKU, preço e estoque de um produto ativo.
- **FR-009**: Uma edição recusada MUST manter os valores anteriores do produto.
- **FR-010**: O operador MUST poder inativar um produto ativo.
- **FR-011**: A inativação MUST preservar todos os dados do produto.
- **FR-012**: A listagem padrão MUST exibir somente produtos ativos.
- **FR-013**: A listagem padrão MUST exibir nome, SKU, preço e estoque de cada produto listado.
- **FR-014**: Quando não houver produtos ativos, a listagem padrão MUST informar que não há produtos cadastrados.
- **FR-015**: O sistema MUST exigir login com conta de operador para qualquer consulta, cadastro, edição ou inativação de produto. Acesso sem login MUST ser recusado e redirecionado para o login. A sessão expira após 8 horas de inatividade, e cada uso renova esse prazo.
- **FR-016**: O nome MUST ter no máximo 150 caracteres. O SKU MUST ter no máximo 30 caracteres e conter apenas letras, números e hífen. Valores fora dessas regras MUST ser recusados com mensagem indicando o campo.
- **FR-017**: O operador MUST poder reativar um produto inativo. Após a reativação, o produto volta à listagem padrão com os mesmos dados.
- **FR-018**: Cada cadastro aceito MUST gerar um evento de criação de produto. Cada edição aceita, inativação e reativação MUST gerar um evento de atualização de produto. Edições recusadas e inativação de produto já inativo NÃO geram evento.
- **FR-019**: Os eventos de produto MUST ser publicados em um message broker. A alteração do produto e o evento correspondente MUST ser registrados juntos: nenhuma alteração aceita pode ficar sem evento, e a indisponibilidade temporária do broker MUST NOT causar perda de evento. Quando o broker voltar, os eventos pendentes MUST ser publicados. A entrega é pelo menos uma vez, então um mesmo evento pode chegar mais de uma vez ao consumidor.
- **FR-020**: A listagem padrão MUST ser paginada. A página tem 50 produtos por padrão e no máximo 200 por requisição. Os produtos MUST aparecer em ordem alfabética pelo nome.
- **FR-021**: O operador MUST poder consultar qualquer produto pelo seu identificador, ativo ou inativo, para ver seus dados sem depender da listagem padrão.
- **FR-022**: Cada produto MUST registrar o operador que o criou e o operador que fez a última alteração (cadastro, edição, inativação ou reativação), junto com a data de cada um desses momentos.
- **FR-023**: Se o banco de dados estiver indisponível durante cadastro, edição, inativação ou reativação, o sistema MUST recusar a operação com mensagem de indisponibilidade. Nenhuma alteração parcial MAY ser gravada, e o operador MUST poder tentar de novo depois.
- **FR-024**: Todo preço exibido ao operador MUST usar real brasileiro com símbolo e duas casas decimais, no formato R$ 1.234,56 (ponto como separador de milhar e vírgula como separador decimal). O preço é gravado sem formatação.

### Key Entities

- **Produto**: item vendido pela loja. Atributos: nome (texto obrigatório), SKU (identificador único), preço (valor monetário não negativo com até duas casas decimais), estoque (quantidade inteira não negativa), situação (ativo ou inativo), operador que criou e operador da última alteração (FR-022).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: O operador cadastra um produto válido em menos de 1 minuto.
- **SC-002**: 100% das tentativas de cadastro ou edição com SKU repetido ou preço negativo são recusadas e não alteram o catálogo.
- **SC-003**: 100% das consultas à listagem padrão não exibem produtos inativos.
- **SC-004**: O operador inativa um produto em menos de 30 segundos a partir da listagem.
- **SC-005**: Nenhum produto inativado perde dados após a inativação.
- **SC-006**: A listagem padrão exibe a primeira página em menos de 2 segundos com 10.000 produtos cadastrados.
- **SC-007**: Em um teste de indisponibilidade do broker, 100% das alterações aceitas durante a falha geram evento publicado após o broker voltar, sem perda e sem evento para alteração recusada.

## Assumptions

- Cada operador usa conta própria com login; não há perfis diferentes (ex.: administrador) nesta versão.
- Não há tela ou filtro para consultar produtos inativos; a listagem padrão é a única listagem.
- Produto inativo mantém seu SKU reservado, para evitar que um SKU seja reutilizado por engano.
- SKU é comparado sem diferenciar maiúsculas/minúsculas e sem espaços nas bordas.
- Preço e estoque aceitam zero.
- A listagem padrão tem 50 produtos por página, no máximo 200, ordenados por nome de A a Z (decisão de default, registrada no FR-020).- Moeda única: real brasileiro (BRL). Não há conversão de moeda nesta versão.
- O volume máximo é de 10.000 produtos somados, ativos e inativos.
- Não há integração com vendas, compras ou outros sistemas nesta versão; movimentação de estoque ocorre apenas por edição manual.
