# Cadastro de Produto Constitution

## Core Principles

### I. Clean Architecture com Dependência em Um Sentido

O backend MUST ser organizado em camadas concêntricas: Domain, Application,
Infrastructure e Web (apresentação). As dependências MUST apontar apenas para
dentro:

- Domain não referencia nenhum outro projeto da solução.
- Application referencia somente Domain.
- Infrastructure e Web referenciam Application e Domain; nunca o inverso.
- Código de Domain MUST NOT depender de EF Core, ASP.NET Core, IOptions ou
  qualquer biblioteca de infraestrutura.
- Violação do sentido de dependência MUST ser rejeitada em revisão de código.

Rationale: isolar regras de negócio de frameworks e do banco de dados permite
testá-las sem infraestrutura e trocar detalhes técnicos sem reescrever o domínio.

### II. Domain-Driven Design

- Entidades, value objects e agregados MUST modelar as regras de negócio do
  cadastro de produto.
- Invariantes MUST ser garantidas dentro do agregado (construtor e métodos de
  comportamento), não apenas por validação em controllers ou DTOs.
- Entidades SHOULD NOT expor setters públicos; o estado muda por operações
  com nome de domínio.
- Nomes de classes e métodos do Domain MUST usar a linguagem ubíqua do negócio.
- Um agregado é a unidade de consistência transacional. Referências a outro
  agregado MUST ser feitas por identificador.

Rationale: concentrar as regras no domínio evita que elas se espalhem e
divirjam entre camadas.

### III. Configuração Externa, Nada Fixo no Código

- Nenhum valor que varie por ambiente (connection strings, segredos, URLs,
  timeouts, chaves, credenciais) MAY aparecer no código-fonte.
- A configuração MUST ser lida em classes tipadas via IOptions<T>, com
  validação na inicialização da aplicação (ValidateOnStart).
- Segredos e valores específicos de ambiente MUST vir de variáveis de ambiente
  ou de um provedor de segredos. Nenhum segredo MUST ser commitado.
- appsettings.json MAY conter apenas padrões não sensíveis.
- Regras de negócio do domínio (ex.: tamanho máximo de um campo) NÃO são
  configuração de ambiente e MAY ser constantes no Domain.

Rationale: o mesmo binário MUST rodar em qualquer ambiente sem recompilação.

### IV. Persistência com EF Core e PostgreSQL

- O acesso a dados MUST usar EF Core com o provedor Npgsql. PostgreSQL é o
  único SGBD alvo.
- SQL MUST NOT ser montado por concatenação ou interpolação de entradas.
  Consultas MUST usar LINQ ou APIs parametrizadas do EF Core (FromSql,
  ExecuteSql com parâmetros).
- O DbContext e as configurações de mapeamento MUST ficar na camada
  Infrastructure. Domain não conhece DbContext.
- Alterações de schema MUST ser feitas por migrations versionadas no
  repositório. Alterações manuais no banco são proibidas.

Rationale: parametrização elimina injeção de SQL e migrations mantêm o schema
reproduzível entre ambientes.

### V. Teste em Toda Mudança de Código

- Toda mudança de código de produção MUST vir acompanhada de teste automatizado
  que a cubra. Correção de bug MUST incluir teste que falha antes da correção.
- Ferramentas obrigatórias: xUnit (framework de teste), Moq (dublês de
  dependência) e FluentAssertions (asserções).
- Testes de unidade MUST NOT acessar rede, banco real ou sistema de arquivos.
  Dependências externas MUST ser substituídas por Moq.
- Mocks MUST ser usados apenas em fronteiras (interfaces de dependências
  externas). Entidades e value objects do Domain MUST ser instanciados de verdade.
- Testes de persistência SHOULD rodar contra PostgreSQL real (ex.: container
  descartável), não contra provedor em memória, pois os provedores se comportam
  de forma diferente.

Rationale: teste junto da mudança impede regressão silenciosa e documenta o
comportamento esperado.

### VI. Pronto Somente com Build e Testes Verdes

- Uma tarefa MUST NOT ser considerada concluída sem `dotnet build` concluir sem
  erros e sem `dotnet test` terminar com todos os testes passando.
- Warnings de compilação SHOULD ser zero em código novo.
- Testes ignorados ou desabilitados MUST ser justificados na descrição da
  mudança.
- O resultado de build e testes MUST ser reportado com a saída real, não com
  suposições.

Rationale: estado verde é a única evidência objetiva de que a mudança funciona
e não quebrou o restante.

### VII. YAGNI

- Nenhuma abstração (interface, classe base, tipo genérico, padrão de projeto,
  camada extra) MAY ser criada sem pelo menos dois usos reais e atuais.
- Um "uso real" é código de produção consumindo a abstração, ou uma segunda
  implementação em produção. Mocks em testes NÃO contam como uso real.
- Funcionalidades não pedidas pela tarefa atual MUST NOT ser implementadas
  "para o futuro".
- Nova dependência NuGet MUST ser justificada e aprovada na revisão.

Exceção (amendment 1.1.0): uma porta de persistência, isto é, uma interface de
repositório definida no Domain e implementada na Infrastructure, é permitida
mesmo com uma única implementação de produção, quando for exigida pelo princípio I.
A porta MUST ser justificada no plano, em Complexity Tracking, citando esta exceção.
Nenhuma outra abstração recebe essa exceção.

Rationale: abstrações especulativas aumentam custo de leitura e de manutenção
sem benefício demonstrado.

## Technology Stack e Restrições Técnicas

- Runtime: .NET 10 (`net10.0`) em todos os projetos da solução.
- Persistência: EF Core com Npgsql sobre PostgreSQL.
- Configuração: Microsoft.Extensions.Options (IOptions<T>) e variáveis de
  ambiente.
- Testes: xUnit, Moq e FluentAssertions. Nenhuma outra biblioteca de teste ou
  mock MUST ser introduzida sem amendment.
- Proibido: SQL concatenado, segredos no repositório, referência de Domain a
  infraestrutura, valores fixos de ambiente no código.

## Fluxo de Desenvolvimento e Portões de Qualidade

Ordem de trabalho para cada mudança:

1. Escrever ou ajustar o teste que descreve o comportamento esperado.
2. Implementar primeiro no Domain e na Application, mantendo o teste verde.
3. Implementar Infrastructure e Web conforme necessário.
4. Rodar testes de persistência contra PostgreSQL quando houver mudança de
   mapeamento ou consulta.

Portões obrigatórios antes de concluir uma tarefa:

- Portão 1: `dotnet build` sem erros.
- Portão 2: `dotnet test` com todos os testes passando.
- Portão 3: revisão confirma sentido de dependência (I), agregados e
  invariantes (II), ausência de valores fixos (III), ausência de SQL concatenado
  (IV), testes da mudança (V) e ausência de abstração sem segundo uso (VII).

## Governance

Esta constituição prevalece sobre quaisquer outras práticas do projeto em caso
de conflito.

Procedimento de emenda:

- Toda emenda MUST ser proposta por Pull Request que altere este arquivo.
- A emenda MUST incluir justificativa, o Sync Impact Report e aprovação de pelo
  menos um mantenedor.
- Se a emenda afetar código existente, o PR MUST incluir plano de migração.

Política de versionamento (semântico):

- MAJOR: remoção ou redefinição incompatível de princípio ou restrição.
- MINOR: novo princípio ou seção, ou expansão material de orientação existente.
- PATCH: esclarecimento de redação, correção de typo ou refinamento não
  semântico.

Revisão de conformidade:

- Todo PR MUST verificar os princípios I a VII. O portão 3 do fluxo de
  desenvolvimento é o checklist mínimo.
- Violação justificada MUST ser registrada na descrição do PR, com a razão e a
  data prevista para correção.
- Complexidade adicional MUST ser justificada contra o princípio VII.

**Version**: 1.1.0 | **Ratified**: 2026-10-27 | **Last Amended**: 2026-10-27
