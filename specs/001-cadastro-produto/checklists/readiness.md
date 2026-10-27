# Readiness Checklist: Cadastro de Produtos

**Purpose**: Validar a qualidade dos requisitos (spec, plano e contratos) antes de gerar as tasks
**Created**: 2026-10-27
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md) | [contracts](../contracts/)

**Note**: Checklist personalizado gerado por `/speckit-checklist`. Foco padrão: lacunas entre o spec e o plano (mensageria, paginação, segurança) e requisitos de exceção. Profundidade padrão, audiência: revisor de PR.
**Review Ownership**: Artefato de revisão de qualidade de requisitos, de responsabilidade do revisor. Marque `[x]` apenas quando o revisor julgar o critério satisfeito.
**Marker Semantics**: `[x]` significa critério de qualidade de requisito revisado e satisfeito. Não significa que a implementação está pronta.

## Requirement Completeness

- [ ] CHK001 Os requisitos de publicação de eventos (criação e atualização de produto) estão definidos no spec, ou existem só no plano? [Gap, Spec §Requirements]
- [ ] CHK002 A garantia de entrega "pelo menos uma vez" do evento está definida como requisito, ou só aparece no contrato de eventos? [Gap, Contracts]
- [ ] CHK003 Os parâmetros da listagem (tamanho padrão de página, máximo e ordenação) estão no spec ou só no plano? [Gap, Spec §FR-012]
- [ ] CHK004 A ordenação da listagem padrão está definida? [Gap, Spec §FR-012]
- [ ] CHK005 Está especificado como consultar um produto inativo por id, citado em US4 cenário 2 como "por outro meio"? [Gap, Spec §US4]
- [ ] CHK006 Os requisitos de validade do token, renovação e logout estão definidos para FR-015? [Gap, Spec §FR-015]
- [ ] CHK007 A criação do primeiro operador (seed de desenvolvimento) tem requisito no spec? [Gap]
- [ ] CHK008 As mensagens de erro exibidas ao operador em cada recusa (SKU, preço, estoque, nome, login) estão definidas? [Completeness, Spec §FR-004, FR-005, FR-006, FR-015]

## Requirement Clarity

- [ ] CHK009 "Preço não negativo com até duas casas decimais" define moeda e formato de exibição de forma verificável? [Clarity, Spec §Key Entities]
- [ ] CHK010 "SKU sem diferenciar maiúsculas" define se acentos e caracteres fora de ASCII são aceitos? [Ambiguity, Spec §FR-016]
- [ ] CHK011 "Edição recusada mantém os valores anteriores" define o que o operador vê na tela após a recusa? [Clarity, Spec §FR-009]
- [ ] CHK012 "SKU reservado" para produto inativo define se o SKU pode ser liberado depois, ou se fica bloqueado para sempre? [Ambiguity, Spec §Assumptions]
- [ ] CHK013 "Listagem padrão" define se inclui qualquer filtro além da situação ativa? [Clarity, Spec §FR-012]

## Requirement Consistency

- [ ] CHK014 A premissa "conta própria, sem perfis" em Assumptions é consistente com FR-015 e com a clarificação Q1? [Consistency]
- [ ] CHK015 A regra de reativação (FR-017) aparece de forma consistente em US4, Edge Cases e Assumptions? [Consistency]
- [ ] CHK016 Os eventos gerados por inativar e reativar (`produto.atualizado`, decisão do plano) estão previstos no spec? [Conflict, Plan §Decisões]
- [ ] CHK017 A regra "SKU único contra ativos e inativos" é consistente com US3 cenário 2 e com SC-002? [Consistency]
- [ ] CHK018 A ação de inativar em SC-004 e em US4 descreve o mesmo fluxo? [Consistency, Spec §SC-004]

## Acceptance Criteria Quality

- [ ] CHK019 SC-001 (cadastro em menos de 1 minuto) define o que conta como cadastro concluído? [Measurability, Spec §SC-001]
- [ ] CHK020 SC-005 (nenhum produto perde dados) define quais campos são comparados? [Measurability, Spec §SC-005]
- [ ] CHK021 SC-002 (100% das tentativas recusadas) define como é medido o caso de corrida entre dois operadores? [Measurability, Spec §SC-002]
- [ ] CHK022 SC-006 (2 s com 10.000 produtos) define as condições de medição, como carga e proporção de inativos? [Measurability, Spec §SC-006]

## Scenario Coverage

- [ ] CHK023 Há requisitos de comportamento quando o banco ou o broker estão indisponíveis durante o cadastro ou a edição? [Coverage, Gap]
- [ ] CHK024 Há requisitos para token expirado durante uma edição? [Coverage, Exception Flow]
- [ ] CHK025 Há requisitos para a listagem quando a página pedida está além do total de produtos ativos? [Coverage, Edge Case]

## Edge Case Coverage

- [ ] CHK026 Os limites de tamanho de página (padrão 50, máximo 200) estão no spec? [Gap, Plan §Research 7]
- [ ] CHK027 Há requisito para inativar e editar o mesmo produto ao mesmo tempo? [Coverage, Gap]
- [ ] CHK028 Há requisito para a falha de publicação que deixa o evento parado após o número máximo de tentativas? [Edge Case, Gap, Contracts]

## Non-Functional Requirements

- [ ] CHK029 Os requisitos de armazenamento seguro de senha do operador estão definidos? [Gap, Spec §FR-015]
- [ ] CHK030 Os requisitos de observabilidade (log das operações do operador e falhas do outbox) estão definidos? [Gap]
- [ ] CHK031 Há requisito de registro de autoria (quem cadastrou ou alterou cada produto), dado que o login foi exigido por Q1? [Gap, Spec §Clarifications]

## Dependencies & Assumptions

- [ ] CHK032 A escolha de RabbitMQ e do padrão outbox está registrada como premissa no spec, ou só no plano? [Dependency, Gap]
- [ ] CHK033 A premissa de volume de 10.000 produtos foi validada contra o plano de paginação? [Assumption, Plan §Research 7]
- [ ] CHK034 As decisões pendentes do plano (exceção para `Microsoft.AspNetCore.Mvc.Testing` e tensão da porta de repositório) estão rastreadas como pendências? [Traceability, Plan §Complexity Tracking]

## Ambiguities & Conflicts

- [ ] CHK035 FR-010 não define o que acontece ao inativar um produto já inativo, mas Edge Cases define. Há conflito ou falta de rastreio? [Conflict, Spec §FR-010, Edge Cases]
- [ ] CHK036 Os termos "situação", "status" e "ativo" são usados de forma consistente entre spec, data-model e contratos? [Terminology]

## Notes

- Itens desmarcados precisam de atualização do spec ou do plano, ou de decisão explícita do revisor.
- Este checklist não é alterado por `/speckit-implement`. Marcadores são de responsabilidade do revisor.
