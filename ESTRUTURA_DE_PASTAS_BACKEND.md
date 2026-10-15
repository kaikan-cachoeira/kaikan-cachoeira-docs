# Estrutura de Pastas do Backend

Este documento define como o código do `kaikan-cachoeira-api` é organizado. O objetivo é que qualquer pessoa do time saiba **onde criar** e **onde procurar** uma classe, e que os módulos não virem uma teia de dependências circulares.

## Princípio: pacote por domínio, camadas dentro de cada domínio

Em vez de agrupar por tipo técnico no projeto inteiro (`controller/`, `service/`, `repository/` com tudo misturado), o código é agrupado **por domínio de negócio**. Cada domínio tem as suas próprias camadas dentro dele.

**Por quê:**

- Cada história de usuário (H1 a H7) corresponde a um ou dois domínios, então um card do Jira mexe em poucas pastas.
- Reduz conflitos de merge: cada pessoa trabalha no seu domínio.
- As dependências entre módulos ficam visíveis (quem importa quem).

**Custo aceito:** o Java não permite esconder classes entre subpacotes (`controller`, `service`...). As regras de dependência da seção abaixo são acordo do time, e não imposição do compilador.

## Estrutura

Pacote raiz: `com.kaikancachoeira.api`

```
src/main/java/com/kaikancachoeira/api/
├── ApiApplication.java
│
├── common/                      # transversal, sem regra de negócio
│   ├── config/                  # OpenApiConfig, CorsConfig, etc.
│   ├── exception/               # ApiException, GlobalExceptionHandler, ErroResponse
│   └── dto/                     # PaginaResponse<T> (itens, page, size, totalItens, totalPaginas)
│
├── auth/                        # H1
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── entity/                  # Usuario, Cargo, Permissao, TokenSenha, RefreshToken
│   ├── dto/
│   └── security/                # SecurityConfig, JwtService, filtro JWT
│
├── evento/                      # H2: evento, status e custos adicionais
├── cardapio/                    # H2 + H5: produto_evento, snapshot, desconto
├── estoque/                     # H3: produto, categoria, lote, movimentação
├── doacao/                      # H4
├── venda/                       # H5: venda e item_venda
├── caixa/                       # H5: lançamentos do caixa e fiados
├── relatorio/                   # H6
└── dashboard/                   # H7
```

Cada domínio de negócio repete o mesmo miolo:

```
evento/
├── controller/    EventoController, CustoEventoController
├── service/       EventoService, CustoEventoService
├── repository/    EventoRepository, CustoEventoRepository
├── entity/        Evento, StatusEvento, CustoEvento, TipoEvento (enum)
└── dto/           CriarEventoRequest, EventoResponse, ...
```

Pastas só são criadas quando há classe para colocar (por exemplo, `relatorio` e `dashboard` não têm `entity/`, pois não criam tabelas).

Recursos:

```
src/main/resources/
├── application.properties
├── application-dev.properties
├── application-prod.properties
└── db/migration/                # migrations (ver seção "Migrations")
```

Os testes espelham a estrutura: `src/test/java/com/kaikancachoeira/api/<dominio>/<camada>/`.

## Responsabilidade de cada camada

| Camada | Responsabilidade | Não faz |
|---|---|---|
| `controller` | Recebe HTTP, valida o corpo (`@Valid`), checa a permissão (`@PreAuthorize`), chama o service e devolve DTO. | Regra de negócio, acesso a repository. |
| `service` | Regra de negócio e `@Transactional`. É a **porta de entrada do domínio** para outros domínios. | Conhecer HTTP (`ResponseEntity`, status). |
| `repository` | Interfaces Spring Data JPA e queries. | Regra de negócio. |
| `entity` | Entidades JPA e enums do domínio. | Ser devolvida pela API. |
| `dto` | Records de request e response. | Conter lógica além de conversão simples. |

Convenções de nome: `<Nome>Controller`, `<Nome>Service`, `<Nome>Repository`, entidade no singular (`Evento`), DTOs como `<Acao><Nome>Request` e `<Nome>Response`. Nomes de domínio e de classes de negócio em português sem acento, igual às tabelas e aos endpoints das histórias.

## De quem é cada endpoint

**O domínio dono é o dono dos dados, não o prefixo da URL.** Vários endpoints de `/eventos/{id}/...` não ficam em `evento`:

| Endpoint | Fica em |
|---|---|
| `/eventos`, `/eventos/{id}`, `/eventos/{id}/custos` | `evento` |
| `/eventos/{id}/produtos`, `.../produtos/{id}/desconto` | `cardapio` |
| `/eventos/{id}/vendas`, `.../vendas/{id}/cancelamento` | `venda` |
| `/eventos/{id}/caixa`, `/eventos/{id}/fiados` | `caixa` |
| `/eventos/{id}/relatorio`, `/relatorios/eventos` | `relatorio` |
| `/produtos`, `/categorias`, `/produtos/{id}/movimentacoes`, `/estoques/{id}/movimentacoes` | `estoque` |
| `/doacoes` | `doacao` |
| `/dashboard/*` | `dashboard` |

Por que `cardapio` é separado de `evento`: `produto_evento` tem snapshot, origem (`ESTOQUE` ou `PRODUZIDO`) e desconto com regras próprias. Dentro de `evento` ele dominaria o pacote.

## Regras de dependência

### 1. Direção permitida

Um domínio só pode depender dos domínios "abaixo" dele:

```
auth        → (nenhum)
evento      → (nenhum)
estoque     → evento
cardapio    → evento, estoque
doacao      → evento, estoque
venda       → evento, cardapio, estoque
caixa       → evento
relatorio   → leitura de todos
dashboard   → relatorio
```

Ciclo entre domínios é proibido. Se precisar de uma dependência "para cima", reveja onde a regra está ou use a regra 4.

### 2. Só se fala com outro domínio pelo `service`

De fora do domínio, a lógica é chamada apenas pelas classes de `service` (e os DTOs que elas recebem e devolvem). **Nunca** se usa o `repository` de outro domínio. A única razão para importar uma `entity` de outro domínio é a referência `@ManyToOne` permitida na regra 3.

Exemplo: ao registrar uma venda, `VendaService` chama um método de `EstoqueService` que consome os lotes em FIFO. Isso acontece dentro da mesma transação, pois `@Transactional` propaga entre services.

### 3. Relacionamentos entre entidades

- **Dentro do mesmo domínio**, relacionamentos JPA normais (`@ManyToOne`, `@OneToMany`). Em relações N:N, usa-se uma entidade para a tabela associativa com `@ManyToOne` para cada lado. O único caso hoje é `cargo` ↔ `permissao`, por meio de `CargoPermissao` (que precisa ser entidade por ter `id`, `criado_em` e `atualizado_em`).
- **Entre domínios, "para baixo"** (na direção permitida pela regra 1) pode usar `@ManyToOne` com `fetch = LAZY`. Exemplos: `Doacao → Evento`, `Doacao → Produto`, `ProdutoEvento → Evento`, `ItemVenda → ProdutoEvento`. Como a direção de dependência já é sem ciclos, isso não cria ciclo.
- **Entre domínios, "para cima"**, só por ID (`Long`). Os casos são `MovimentacaoEstoque.itemVendaId` e `MovimentacaoEstoque.doacaoId`: o estoque não pode importar `venda` nem `doacao`, que já dependem dele. A foreign key continua existindo no banco; só o mapeamento JPA é por ID.

**Boas práticas JPA para evitar N+1:**

- `@ManyToOne` vem `EAGER` por padrão. Declare sempre `fetch = FetchType.LAZY`.
- Quando a resposta precisar dos dados da relação (ex.: nome do evento na lista de doações), traga na própria query com `JOIN FETCH` ou `@EntityGraph`.
- `JOIN FETCH` de `@ManyToOne` funciona com paginação. `JOIN FETCH` de **coleção** (`@OneToMany`) junto com `Pageable` faz o Hibernate paginar em memória; nesse caso, carregue a coleção em uma segunda query.

### 4. Leitura agregada cruzada é permitida, escrita nunca

Alguns casos precisam ler dados de outro domínio sem criar ciclo:

- `relatorio` e `dashboard` leem tudo.
- `cardapio` precisa de `quantidadeVendida` (vem de `item_venda`), mas `venda` já depende de `cardapio`.
- `evento` precisa saber se há vendas ou lançamentos de caixa para desfazer o início ou reabrir, e montar o resumo do evento.

Nesses casos, o `repository` do domínio que lê pode ter uma query (JPQL ou nativa, juntando por ID) que passa por tabelas de outro domínio. Regras: somente leitura, devolvendo projeção ou DTO (nunca a entidade do outro domínio), com um comentário dizendo de onde vem o dado. **Toda alteração nos dados de um domínio passa pelo `service` dele.**

### 5. Uma única fonte para as fórmulas financeiras

Receita, custo e lucro (H6) são as mesmas usadas pelo dashboard (H7). As fórmulas ficam em um service de `relatorio` (ex.: `ResultadoFinanceiroService`), e `dashboard` as consome. Não se reimplementa a conta no dashboard.

### 6. Controller não devolve entidade

Toda resposta e todo corpo de requisição é um DTO (de preferência `record`). Isso protege o contrato da API contra mudanças no modelo e evita `LazyInitializationException`.

## Onde fica o que é transversal

- **Formato de erro** (`codigo`, `mensagem`, `campos`) e o `@RestControllerAdvice`: `common/exception`.
- **Envelope de paginação**: `common/dto/PaginaResponse`.
- **`criado_em` e `atualizado_em`**: declarados em cada entidade, com as anotações do Hibernate que preenchem o valor sozinhas:

  ```java
  @CreationTimestamp
  @Column(name = "criado_em", nullable = false, updatable = false)
  private Instant criadoEm;

  @UpdateTimestamp
  @Column(name = "atualizado_em", nullable = false)
  private Instant atualizadoEm;
  ```

  Atenção: `@UpdateTimestamp` não dispara em `UPDATE` em massa feito por `@Modifying @Query`. Nesse caso, `atualizado_em = now()` vai no próprio SQL.
- **JWT, filtro e configuração de segurança**: `auth/security`.
- **Permissões**: nos controllers, com `@PreAuthorize("hasAuthority('EVENTOS_MENU')")`. Exceções documentadas nas histórias, como `GET /eventos`, que exige `EVENTOS_LEITURA`.
- **Controle de concorrência** (um evento em andamento por vez, saldo de lote): no service dono da regra, com o respaldo do banco (índice único parcial, lock de lote).

## Migrations

Ficam em `src/main/resources/db/migration`, uma por mudança, com nome sequencial por data/versão. A carga inicial (permissões, cargos, contas, status de evento, categorias e tipos de movimentação) também é feita por migration. A ordem das migrations segue a direção de dependência da regra 1 (as tabelas de `evento` antes das de `cardapio`, por exemplo).

## Onde coloco...?

| Preciso de... | Coloco em |
|---|---|
| Uma regra de negócio nova de venda | `venda/service` |
| Uma query que soma vendas para o relatório | `relatorio/repository` (regra 4) |
| Um enum de forma de pagamento usado por doação e caixa | No domínio que mais o usa; o outro importa o enum via DTO do service (se virar problema, vai para `common`) |
| Uma exceção de negócio ("estoque insuficiente") | `common/exception` (classe genérica) com o código específico no handler, ou dentro do domínio se só ele usa |
| Um conversor entidade → DTO | Método estático no próprio DTO ou classe `<Nome>Mapper` no `dto/` do domínio |
| Algo que não sei onde colocar | Perguntar no grupo antes de criar uma pasta nova |

## Evolução

Se um domínio crescer demais, divide-se por subdomínio mantendo a mesma estrutura. O candidato óbvio é `estoque` (produto, categoria e movimentação), que hoje concentra três contextos. Mudanças nesta organização são discutidas com o time e atualizadas neste documento na mesma PR.