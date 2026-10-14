# Histórias de Usuário — Kaikan Cachoeira (versão autossuficiente para Jira)

## Como usar este arquivo

- Cada história (seções `# H1` a `# H7`) pode ser copiada **inteira** para um card do Jira. Nenhuma depende de outra seção, de outro arquivo ou de códigos como "PT-02": os padrões de tela e de API que antes eram transversais estão escritos dentro de cada história, no bloco **"Padrões desta história"**.
- Cada história termina com **"Sugestão de subtarefas"**, que divide o trabalho em Banco de dados, Backend (um por endpoint) e Frontend (um por tela ou modal).
- Quando uma história usa um endpoint que pertence a outra, o contrato necessário aparece em **"Endpoints consumidos de outras histórias"**, com o essencial para implementar sem abrir a outra. A dona do endpoint continua sendo a história indicada.
- Convenções que valem para todo o sistema e estão repetidas dentro de cada história: toda tabela tem `criado_em` e `atualizado_em` (na API, `criadoEm` e `atualizadoEm`); o parâmetro de texto das pesquisas se chama `q`; as listas paginadas seguem o padrão de paginação descrito na história que as usa.
- As validações confirmadas estão incorporadas ao texto. Pontos técnicos ainda não definidos estão listados ao final.
- Estrutura de cada história: Objetivo → Padrões desta história → Detalhes técnicos (Banco de dados, Backend, Frontend) → Regras → Protótipo → Premissas → Sugestão de subtarefas.

## Visão geral das histórias

| # | História | Permissão exigida |
|---|---|---|
| H1 | Autenticação | nenhuma |
| H2 | Gestão de eventos | `EVENTOS_MENU` |
| H3 | Gestão de estoque | `ESTOQUE_MENU` |
| H4 | Controle de doações | `DOACOES_MENU` |
| H5 | Controle de vendas e caixa | `VENDAS_MENU` (vendas) e `CAIXA_MENU` (caixa) |
| H6 | Relatórios financeiros | `RELATORIOS_MENU` |
| H7 | Dashboards gerenciais | `DASHBOARD_MENU` |

Não existe tela de gestão de usuários. O sistema é usado por um grupo seleto, com **uma conta compartilhada por cargo**, criada por migration. Há dois cargos: Administrador e Operador. Por isso não se registra qual usuário fez cada operação.

---

# H1 — Autenticação

## Objetivo

**Eu**, como usuário do sistema  
**Quero** uma tela de login e de recuperação de senha  
**Para** acessar a minha conta do sistema e poder recuperar minha senha caso eu esqueça.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de validação de formulário**
- O front valida os campos antes de chamar a API.
- Campo inválido recebe borda vermelha e a mensagem do erro logo abaixo.
- Erros de validação devolvidos pela API (HTTP 400) são exibidos no campo correspondente.

**Padrão de API**
- Erros seguem o corpo 
  ```json
  {
   "codigo":  string,
   "mensagem":  string,
   "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 404 não encontrado, 409 conflito.
- Os endpoints `/auth/*` não exigem JWT, exceto onde indicado.
- Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Criar as tabelas: `cargo`, `permissao`, `cargo_permissao`, `usuario`, `token_senha`, `refresh_token`.
- Todas as tabelas desta história têm `criado_em` e `atualizado_em` (`NOT NULL`; `atualizado_em` começa igual a `criado_em` e é atualizado pelo backend a cada alteração). Na API, `criadoEm` e `atualizadoEm`.
- Carga inicial (migration): as permissões abaixo, os dois cargos (Administrador e Operador) e uma conta por cargo com `status = 'pendente'`.

| Permissão | Dá acesso a |
|---|---|
| `EVENTOS_MENU` | Eventos, detalhes, cardápio e histórico de vendas |
| `ESTOQUE_MENU` | Estoque, categorias, lotes e histórico de movimentações |
| `DOACOES_MENU` | Doações |
| `VENDAS_MENU` | Registro de venda |
| `CAIXA_MENU` | Caixa e fiados |
| `RELATORIOS_MENU` | Relatórios por evento |
| `DASHBOARD_MENU` | Dashboards |
| `EVENTOS_LEITURA` | Listar e consultar eventos (`GET /eventos`), inclusive a partir de outros módulos, sem editar |
| `PRODUTOS_LEITURA` | Consultar produtos a partir de outros módulos, sem editar |
| `CARDAPIO_EVENTO_LEITURA` | Consultar cardápio de eventos a partir de outros módulos, sem editar |

Cargos iniciais validados:

| Cargo | Permissões |
|---|---|
| Administrador | todas |
| Operador | `VENDAS_MENU`, `CAIXA_MENU`, `EVENTOS_LEITURA`, `PRODUTOS_LEITURA`, `CARDAPIO_EVENTO_LEITURA` |

### Backend

**Criar endpoint: `POST /auth/login`**
- Corpo: 
  ```json
  {
     "email":  string,
     "senha":  string
  }
  ```

  - `email` e `senha` não podem ser nulos ou em branco. `email` usa a anotação `@Email`.
- Busca o `usuario` pelo email e confere a senha com o hash. A conta precisa estar com `status = 'ativo'`.
- Sucesso (200): cria o `accessToken` (JWT de curta duração) e o `refreshToken`. Guarda só o **hash** do refresh em `refresh_token`.
  - Resposta: 
    ```json
    {
       "accessToken":  string,
       "refreshToken":  string,
       "usuario":  {
         "nome":  string,
         "cargo":  string,
         "permissoes":  [
          string
        ]
      }
    }
    ```

- Falha (401): 
  ```json
  {
     "codigo":  "CREDENCIAIS_INVALIDAS",
     "mensagem":  "Usuário ou senha inválidos"
  }
  ```
  . A resposta é a mesma para email inexistente, senha errada ou conta não ativa.

**Criar endpoint: `POST /auth/refresh`**
- Corpo: 
  ```json
  {
     "refreshToken":  string
  }
  ```

- Valida que o hash existe, não está revogado e não expirou.
- Revoga o token usado e emite um novo par de tokens (rotação).
- Falha (401) se inválido, revogado ou expirado.

**Criar endpoint: `POST /auth/logout`**
- Corpo: 
  ```json
  {
     "refreshToken":  string
  }
  ```
  . Marca `revogado = true`. Retorna 204.

**Criar endpoint: `POST /auth/recuperacao-senha`**
- Corpo: 
  ```json
  {
     "email":  string
  }
  ```
  . Não pode ser nulo ou em branco.
- Se existir `usuario` com esse email:
  - cria um registro em `token_senha` (64 caracteres, `usado = false`, `expira_em` em exatamente 1 hora);
  - invalida os tokens anteriores não usados do mesmo usuário;
  - envia um email com o link `{url-do-front}/redefinicao-senha?token={token}`.
- **Sempre retorna 202**, exista o email ou não, para não revelar quais emails estão cadastrados.

**Criar endpoint: `POST /auth/redefinicao-senha`**
- Corpo: 
  ```json
  {
     "token":  string,
     "novaSenha":  string
  }
  ```

- `novaSenha`: não nula, entre 8 e 255 caracteres.
- O token deve existir, estar com `usado = false` e `expira_em` no futuro. Caso contrário, 400 com `codigo = "TOKEN_INVALIDO"`.
- Sucesso (204):
  - grava o hash da nova senha e marca o token como `usado`;
  - revoga todos os `refresh_token` do usuário;
  - se `status = 'pendente'`, muda para `ativo` (a primeira configuração de senha é feita por este link de recuperação).

### Frontend

**Tela de "Login"** (padrão de tela)
- Campo "Email": não nulo ou em branco.
- Campo "Senha": não nulo ou em branco.
- Botão "Esqueci minha senha":
  - redireciona para "Recuperação de Senha";
  - preenche o campo de email dessa tela com o valor digitado no Login, se houver.
- Botão "Entrar":
  - valida os campos (padrão de validação) e chama `POST /auth/login`;
  - em sucesso, guarda os tokens e redireciona para "Início";
  - caso contrário, mostra "Usuário ou senha inválidos".

**Tela de "Esqueci minha senha"** (padrão de tela)
- Campo "Email": não nulo ou em branco.
- Botão "Recuperar minha senha": chama `POST /auth/recuperacao-senha` e mostra "Caso o email esteja cadastrado no sistema, será enviado um link para recuperar sua senha".
- Botão de voltar: retorna para o "Login".

**Tela de "Redefinição de senha"** (padrão de tela)
- Lê o `token` da URL. Sem token, mostra o modal de erro de token inválido.
- Campo "Senha": não nulo ou em branco, de 8 a 255 caracteres.
- Campo "Confirme sua senha": não nulo ou em branco, igual a "Senha".
- Botão "Redefinir sua senha": chama `POST /auth/redefinicao-senha`.
  - Sucesso: modal de sucesso com o botão "Voltar à página de Login", que redireciona para o "Login".
  - Erro de token inválido: modal de erro com o botão "Tente novamente", que volta para "Esqueci minha senha".
  - Erro do servidor: modal de erro com o botão "Voltar à página de Login".

**Sessão (comportamento global do front)**
- Ao receber 401 em uma chamada autenticada, o front tenta `POST /auth/refresh` uma vez e repete a chamada. Se falhar, limpa a sessão e redireciona para o "Login".
- O menu e as rotas mostram só as telas das permissões retornadas no login.
- Botão "Sair": chama `POST /auth/logout` e redireciona para o "Login".

## Regras
- Nenhuma permissão é exigida nas telas e endpoints de autenticação.
- A senha nunca é armazenada nem logada em texto puro.

## Protótipo
- Existe.

## Premissas
- As tabelas estão criadas no banco de dados.
- Cargos, permissões e contas são criados por migration. Não há cadastro de usuários pela interface.

## Sugestão de subtarefas
- BD: migrations das 6 tabelas e carga inicial de permissões, cargos e contas.
- BE: `POST /auth/login`.
- BE: `POST /auth/refresh` e `POST /auth/logout`.
- BE: `POST /auth/recuperacao-senha` (inclui envio de email).
- BE: `POST /auth/redefinicao-senha`.
- FE: tela de Login.
- FE: tela "Esqueci minha senha".
- FE: tela "Redefinição de senha" com os modais.
- FE: sessão (refresh automático, guarda de rotas por permissão, botão Sair).

---

# H2 — Gestão de eventos

## Objetivo

**Eu**, enquanto administrador  
**Quero** registrar, listar com filtros, pesquisar e editar os eventos da associação (inclusive reuniões internas), reabrir eventos encerrados e montar o cardápio de cada evento com as quantidades previstas  
**Para** ter controle sobre datas, produtos, custos e lucro de cada evento.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de pesquisa e filtros** (telas "Eventos", "Produtos do evento" e "Histórico de vendas")
- Campo de pesquisa e botão de pesquisar chamam o endpoint de listagem da tela, passando o texto no parâmetro `q` e os filtros selecionados.
- Botão "Filtrar" com o painel fechado: muda o ícone e mostra abaixo as opções de filtro.
- Botão "Filtrar" com o painel aberto: muda o ícone e esconde as opções, **sem limpar** os filtros já aplicados.

**Padrão de lista**
- Enquanto carrega, mostra ícone de carregamento.
- Se não houver registros, mostra a mensagem de lista vazia indicada na tela.
- Em caso de erro, mostra um toast de erro e renderiza na lista a mensagem de erro indicada na tela.
- Em caso de sucesso, renderiza um Card para cada registro retornado.

**Padrão de paginação** (listas paginadas desta história)
- Parâmetros `page` (começa em 0; o frontend exibe +1) e `size` (padrão 20, máximo 100).
- Resposta: 
  ```json
  {
     "itens":  [
      ...
    ],
     "page":  number,
     "size":  number,
     "totalItens":  number,
     "totalPaginas":  number
  }
  ```

- Frontend: botões "Anterior" e "Próxima" e o texto "Página X de Y". "Anterior" fica desabilitado na primeira página e "Próxima" na última. Não há "carregar mais".

**Padrão de modal de ação** (criar, editar, excluir, confirmar)
- Ao confirmar, o texto do botão é trocado por um ícone de carregamento.
- Em caso de erro, mostra um modal de erro e **não fecha nem limpa** o modal da ação.
- Em caso de sucesso, mostra um modal de sucesso, fecha o modal da ação e limpa seus campos.

**Padrão de validação de formulário**
- O front valida os campos antes de chamar a API.
- Campo inválido recebe borda vermelha e a mensagem do erro logo abaixo.
- Erros de validação devolvidos pela API (HTTP 400) são exibidos no campo correspondente.

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão `EVENTOS_MENU`, exceto `GET /eventos`, que exige `EVENTOS_LEITURA`. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 403 sem permissão, 404 não encontrado, 409 conflito (duplicidade ou regra de negócio violada).
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Criar as tabelas: `evento`, `status_evento`, `produto_evento`, `custo_evento`. Todas têm `criado_em` e `atualizado_em`.
- `evento` tem `nome`, `observacoes`, `tipo_evento` (`EVENTO` ou `REUNIAO`), `status_id`, `inicio_previsto_em`, `fim_previsto_em`, `iniciado_em` e `encerrado_em`. `iniciado_em` é preenchido ao iniciar o evento. `encerrado_em` representa apenas o encerramento vigente.
- Só pode haver um evento do tipo `EVENTO` com status `EM_ANDAMENTO` por vez. Além da validação no backend, o banco garante isso com um índice único parcial.
- `produto_evento` é o item do cardápio do evento. Tem `evento_id`, `origem` (`ESTOQUE` ou `PRODUZIDO`), `produto_id` (só em `ESTOQUE`; um produto não se repete no mesmo evento), `preco_venda`, `quantidade_prevista` (só em `ESTOQUE`), `custo_total_ingredientes` (só em `PRODUZIDO`), `desconto` (desconto por unidade, 0 quando não há desconto), `desconto_aplicado_em` (nulo quando não há desconto) e o snapshot dos campos históricos do produto (`nome`, `descricao`, `categoria` e `foto_url`, além de outros necessários à apresentação histórica). A `categoria` é guardada no snapshot como **nome (texto)**, sem o id da categoria: editar ou excluir uma categoria não altera cardápios já montados. O snapshot preserva a apresentação histórica; o ID mantém o vínculo necessário a estoque e movimentações.
- Item `PRODUZIDO` é um produto feito na hora com ingredientes (ex.: tempurá). Não existe no estoque, não tem quantidade prevista nem foto, não gera movimentação de estoque e guarda só o custo total dos ingredientes do evento.
- `custo_evento` guarda os custos adicionais do evento: `evento_id`, `nome`, `descricao` (opcional) e `valor` (maior que 0). Ex.: limpeza por faxineira contratada e pagamento a quem ajudou. Contas de luz, água, gás e similares são custo mensal e não entram no custo do evento (fora do escopo).
- As tabelas `venda` e `item_venda` (da história de vendas e caixa) são lidas aqui pelos endpoints de histórico de vendas.
- Carga inicial (migration): 3 status: `NAO_INICIADO`, `EM_ANDAMENTO` e `ENCERRADO`.

### Backend

**Criar endpoint: `GET /eventos`**
- Exige a permissão `EVENTOS_LEITURA` (exceção à permissão `EVENTOS_MENU` do padrão de API desta história).
- Parâmetros: `q` (nome), `statusId`, `tipoEvento`, `dataInicio`, `dataFim` (período sobre `inicio_previsto_em`).
- Retorna 
  ```json
  [
    {
       id,
       nome,
       tipoEvento,
       status:  {
         id,
         codigo,
         descricao
      },
       dataHoraInicio,
       dataHoraEncerramento,
       iniciadoEm,
       encerradoEm
    }
  ]
  ```
  . O evento `EM_ANDAMENTO` vem sempre primeiro; os demais, do mais recente para o mais antigo.

**Criar endpoint: `GET /eventos/{id}`**
- Retorna os dados do evento, `observacoes` e o resumo `{ quantidadeProdutos, quantidadeVendas, totalVendido, totalCustosAdicionais }`. 404 se não existir.

**Criar endpoint: `POST /eventos`**
- Corpo: 
  ```json
  {
     "nome":  string,
     "observacoes":  string,
     "tipoEvento":  "EVENTO" | "REUNIAO",
     "dataHoraInicio":  Instant,
     "dataHoraEncerramento":  Instant
  }
  ```

  - `nome`: não nulo ou em branco, máximo de 150 caracteres.
  - `dataHoraEncerramento` não pode ser anterior a `dataHoraInicio`.
  - `observacoes`: opcional.
- Cria o evento com status `NAO_INICIADO`. Retorna 201 com `{ id, nome, tipoEvento, status, dataHoraInicio, dataHoraEncerramento, criadoEm, atualizadoEm }`.

**Criar endpoint: `PUT /eventos/{id}`**
- Corpo igual ao do `POST /eventos`. 404 se não existir.
- A edição é permitida em qualquer status, inclusive datas e observações.
- Retorna 200 com o evento atualizado.

**Criar endpoint: `PATCH /eventos/{id}`**
- Corpo: 
  ```json
  {
     "status":  number,
     "custos":  [
      {
         "nome":  string,
         "descricao":  string,
         "valor":  number
      }
    ]
  }
  ```
  , em que `status` é o id do novo status. `custos` é opcional e só é aceito na transição para `ENCERRADO`. Retorna 204. 404 se o evento não existir.
- Transições permitidas:
  - `NAO_INICIADO` → `EM_ANDAMENTO`: preenche `iniciado_em`. Retorna 409 se já houver outro evento do tipo `EVENTO` em andamento.
  - `EM_ANDAMENTO` → `ENCERRADO`: preenche `encerrado_em` e grava, na mesma transação, os `custos` informados em `custo_evento`. Se qualquer custo for inválido, nada é encerrado.
  - `EM_ANDAMENTO` → `NAO_INICIADO` (desfazer início): só se não houver vendas nem lançamentos de caixa; limpa `iniciado_em`. Caso contrário, 409.
  - `ENCERRADO` → `EM_ANDAMENTO` (reabertura): permite reabrir **qualquer** evento encerrado, independentemente de existirem vendas, lançamentos de caixa, valores a receber (fiados), doações ou movimentações de estoque. Limpa `encerrado_em` e preserva `iniciado_em`. Retorna 409 se já houver outro evento do tipo `EVENTO` em andamento.
- Qualquer outra transição retorna 409.
- Só pode haver um evento do tipo `EVENTO` em andamento por vez. A verificação e a mudança de status ocorrem na mesma transação e são atômicas, evitando que chamadas concorrentes causem transições inconsistentes. Reuniões não entram nessa regra.
- A reabertura não cria um novo evento e não apaga, duplica ou recria vendas, lançamentos de caixa, valores a receber, doações, movimentações de estoque nem custos adicionais. Preços e snapshots históricos não são recalculados. Continuam valendo as validações próprias de cada operação, inclusive estoque suficiente (estoque nunca negativo) e regras de caixa.
- Não há histórico explícito de reaberturas (sem tabela de transições de estado). Para encerrar de novo, usa-se a mesma transição `EM_ANDAMENTO` → `ENCERRADO`, com suas validações normais.

**Criar endpoint: `GET /eventos/{id}/custos`**
- Lista os custos adicionais do evento: `[{ id, nome, descricao, valor, criadoEm, atualizadoEm }]`. 404 se o evento não existir.

**Criar endpoint: `POST /eventos/{id}/custos`**
- Corpo: 
  ```json
  {
     "nome":  string,
     "descricao":  string,
     "valor":  number
  }
  ```

  - `nome`: não nulo ou em branco, máximo de 150 caracteres.
  - `descricao`: opcional, máximo de 255 caracteres.
  - `valor`: maior que 0.
- Vale para eventos do tipo `EVENTO` e `REUNIAO`, com status `EM_ANDAMENTO` ou `ENCERRADO`. Com `NAO_INICIADO`, retorna 409. 404 se o evento não existir.
- Retorna 201 com o custo criado.

**Criar endpoint: `PUT /eventos/{id}/custos/{custoId}`**
- Corpo e validações iguais ao do `POST /eventos/{id}/custos`, com a mesma regra de status. 404 se o evento ou o custo não existirem. Retorna 200 com o custo atualizado.

**Criar endpoint: `DELETE /eventos/{id}/custos/{custoId}`**
- 404 se o evento ou o custo não existirem. Retorna 204.

**Criar endpoint: `DELETE /eventos/{id}`**
- Só exclui evento `NAO_INICIADO`, sem doações, movimentações ou vendas vinculadas. Caso contrário, 409.
- Remove na mesma transação os `produto_evento` e os `custo_evento` do evento, sem depender de `ON DELETE CASCADE`. Retorna 204.

**Criar endpoint: `GET /eventos/{id}/produtos`**
- Lista o cardápio do evento. Parâmetros: `q`, `categoria` (nome da categoria guardado no snapshot; aceita mais de uma).
- Retorna `[{ id, origem, produtoId, nome, descricao, categoria, fotoUrl, precoVenda, desconto, descontoAplicadoEm, precoComDesconto, quantidadePrevista, quantidadeVendida, restantePrevisto, custoTotalIngredientes }]`, em que `categoria` é o nome (texto) guardado no snapshot e `precoComDesconto` = `precoVenda` − `desconto`.
  - `quantidadeVendida` = soma de `item_venda.quantidade` das vendas não canceladas.
  - `restantePrevisto` = `quantidadePrevista` − `quantidadeVendida`, sem piso em 0. Se a venda superar o previsto, o valor fica negativo.
  - Em itens `PRODUZIDO`, `produtoId`, `quantidadePrevista` e `restantePrevisto` são nulos; em itens `ESTOQUE`, `custoTotalIngredientes` é nulo.

**Criar endpoint: `GET /eventos/{id}/produtos/categorias`**
- Retorna os nomes distintos (sem repetição) das categorias guardadas no snapshot do cardápio do evento, como lista de textos. Exemplo: `["Bebida", "Comida"]`.

**Criar endpoint: `POST /eventos/{id}/produtos`**
- Só permitido com o evento `NAO_INICIADO`. 404 se o evento não existir. Para tipo `REUNIAO`, retorna 409 (reunião não tem cardápio).
- Corpo: 
  ```json
  {
     "origem":  "ESTOQUE" | "PRODUZIDO",
     "produtoId":  number,
     "nome":  string,
     "descricao":  string,
     "categoria":  string,
     "precoVenda":  number,
     "quantidadePrevista":  number,
     "custoTotalIngredientes":  number
  }
  ```

  - `origem = ESTOQUE`: `produtoId` (produto existente e ativo, sem repetir no evento), `precoVenda` (maior ou igual a 0) e `quantidadePrevista` (maior que 0) são obrigatórios. `nome`, `descricao`, `categoria` e `fotoUrl` vêm do produto e formam o snapshot.
  - `origem = PRODUZIDO`: `nome` (obrigatório, máximo de 150 caracteres, único no evento), `precoVenda` (maior ou igual a 0) e `custoTotalIngredientes` (maior ou igual a 0, custo total dos ingredientes no evento) são obrigatórios. `descricao` é opcional. `categoria` é opcional e, se informada, deve ser o nome de uma categoria existente (400 caso contrário). Não recebe `produtoId` nem `quantidadePrevista`.
- Retorna 201 com todos os campos do snapshot: `{ id, eventoId, origem, produtoId, nome, descricao, categoria, fotoUrl, precoVenda, quantidadePrevista, custoTotalIngredientes, desconto, descontoAplicadoEm, criadoEm, atualizadoEm }`.

**Criar endpoint: `POST /eventos/{id}/produtos/{id}/foto`**
- Upload multipart, opcional. Guarda a chave de armazenamento em `foto_url` e retorna a URL de acesso. 404 se o produto não existir.

**Criar endpoint: `PUT /eventos/{id}/produtos/{produtoEventoId}`**
- Só permitido com o evento `NAO_INICIADO`. 404 se não existir.
- Corpo em itens `ESTOQUE`: 
  ```json
  {
     "precoVenda":  number,
     "quantidadePrevista":  number
  }
  ```
  . Em itens `PRODUZIDO`:
  ```json
  {
     "nome":  string,
     "descricao":  string,
     "categoria":  string,
     "precoVenda":  number,
     "custoTotalIngredientes":  number
  }
  ```
  , com as mesmas validações do `POST`. O snapshot de itens `ESTOQUE` não é alterado.
- Depois que o evento começa, o preço-base não muda; descontos são aplicados pelos endpoints de desconto da história de vendas e caixa.
- Retorna 200 com o item atualizado.

**Criar endpoint: `DELETE /eventos/{id}/produtos/{produtoEventoId}`**
- Só permitido com o evento `NAO_INICIADO` e se o item nunca foi vendido. Caso contrário, 409. Retorna 204.

**Criar endpoint: `GET /eventos/{id}/vendas`**
- Histórico de vendas do evento, paginado (padrão de paginação), da mais recente para a mais antiga.
- Parâmetros: `q` (nome do produto), `dataHoraInicio`, `dataHoraFim` e `cancelada` (booleano). Se `cancelada` estiver ausente, mostra todas as vendas; se for `true`, só as canceladas; se for `false`, só as ativas (não canceladas).
- Retorna `[{ id, criadoEm, desconto, precoTotal, quantidadeItens, cancelada }]` dentro do envelope de paginação.

**Criar endpoint: `GET /eventos/{id}/vendas/{vendaId}`**
- Retorna 
  ```json
  {
     id,
     criadoEm,
     itens:  [
      {
         produtoEventoId,
         nome,
         quantidade,
         precoUnitario,
         descontoUnitario
      }
    ],
     desconto,
     precoTotal,
     cancelada,
     canceladaEm
  }
  ```
  . 404 se a venda não for do evento.

### Endpoints consumidos de outras histórias
- `GET /produtos` (dona: gestão de estoque). Parâmetros: `q`, `categoriaId` (aceita mais de uma categoria). Retorna 
  ```json
  [
    {
       id,
       nome,
       categoria:  {
         id,
         nome
      },
       fotoUrl,
       quantidadeAtual
    }
  ]
  ```
  . Usado para consultar produtos ao adicionar itens `ESTOQUE` ao cardápio.
- `POST /eventos/{id}/vendas/{vendaId}/cancelamento` (dona: vendas e caixa). Sem corpo. Retorna 204. Retorna 409 se a venda já estiver cancelada ou se o evento estiver `ENCERRADO`. Estorna o estoque dos itens de origem `ESTOQUE`. Exige `VENDAS_MENU`.

### Frontend

**Tela de "Eventos"** (padrão de tela)
- Campo de pesquisa e botão de pesquisar (padrão de pesquisa): chamam `GET /eventos`. Filtros: status, tipo e período.
- Lista de eventos (padrão de lista): chama `GET /eventos`. Vazio: "Não há eventos cadastrados". Erro: "Houve um erro ao buscar eventos". O evento em andamento aparece no topo da lista e com destaque visual (cor própria e etiqueta "Em andamento").
- Botão "Criar Evento": abre o modal de criação (padrão de modal de ação e de validação):
  - "Nome": não nulo ou em branco, máximo de 150 caracteres.
  - Select "Tipo": Evento ou Reunião.
  - "Data/hora de início" e "Data/hora de encerramento": obrigatórios, encerramento não anterior ao início.
  - "Observações": opcional.
  - Botão "Criar Evento": chama `POST /eventos`. Erro: "Houve um erro ao criar o evento". Sucesso: "Evento criado com sucesso".
- Card de evento: mostra nome, tipo, período e status, com os botões:
  - "Ver detalhes": abre a tela "Detalhes do Evento".
  - "Produtos do evento": abre a tela "Produtos do evento" (só para tipo Evento).
  - "Histórico de vendas": abre a tela "Histórico de vendas" (só para tipo Evento).
  - "Ver relatório": redireciona para o "Relatório do evento" (tela da história de relatórios financeiros). Só aparece se o usuário tiver `RELATORIOS_MENU`.
  - "Iniciar evento": aparece com status `NAO_INICIADO`. Pede confirmação e chama `PATCH /eventos/{id}` com o status `EM_ANDAMENTO`. Se já houver evento do tipo Evento em andamento, o botão fica desabilitado e informa o motivo; a decisão final é do backend (409).
  - "Desfazer início": aparece com status `EM_ANDAMENTO`. Pede confirmação e chama `PATCH /eventos/{id}` com o status `NAO_INICIADO`. Em 409, mostra o motivo no modal de erro.
  - "Encerrar evento": aparece com status `EM_ANDAMENTO`. Abre o modal "Encerrar evento", que pede confirmação e tem a lista opcional "Custos adicionais" (cada custo com nome, descrição e valor; botões "Adicionar custo" e remover custo da lista). Se o evento já tiver custos registrados (ex.: depois de uma reabertura), eles são mostrados (`GET /eventos/{id}/custos`) e só os novos são enviados. Ao confirmar, chama `PATCH /eventos/{id}` com o status `ENCERRADO` e os `custos` novos. Erro: "Houve um erro ao encerrar o evento". Sucesso: "Evento encerrado com sucesso".
  - "Reabrir evento": aparece com status `ENCERRADO`. Pede confirmação informando que o evento voltará ao status "Em andamento" e que as operações desse status serão habilitadas (a confirmação não afirma que dados serão apagados ou recriados) e chama `PATCH /eventos/{id}` com o status `EM_ANDAMENTO`. Enquanto a requisição está em andamento, a ação fica desabilitada. Se já houver evento do tipo Evento em andamento, o botão fica desabilitado e informa o motivo. Após o sucesso confirmado pela API, atualiza o status e mostra as ações do status "Em andamento". Em caso de erro (ex.: 409), mostra o motivo no modal de erro e mantém o status exibido consistente com o servidor.
  - "Editar evento": abre o modal de edição, com a mesma lógica do "Criar Evento", preenchido com os dados atuais. Chama `PUT /eventos/{id}`. Erro: "Houve um erro ao atualizar o evento". Sucesso: "Evento atualizado com sucesso".
  - "Excluir evento": aparece com status `NAO_INICIADO`. Pede confirmação e chama `DELETE /eventos/{id}`.

**Tela de "Detalhes do Evento"** (padrão de tela)
- Mostra os dados do evento e o resumo (`GET /eventos/{id}`): produtos, vendas, total vendido e total de custos adicionais. Erro: "Houve um erro ao buscar o evento".
- Seção "Custos adicionais" (eventos `EM_ANDAMENTO` ou `ENCERRADO`): lista os custos (`GET /eventos/{id}/custos`) e o total. Vazio: "Não há custos adicionais neste evento". Erro: "Houve um erro ao buscar os custos adicionais". Botão "Adicionar custo" e, em cada custo, "Editar" e "Excluir" (com confirmação).
- Modal "Custo adicional" (padrão de modal de ação e de validação): "Nome" (obrigatório, máximo de 150 caracteres), "Descrição" (opcional, máximo de 255 caracteres) e "Valor" (só números positivos maiores que 0). Chama `POST /eventos/{id}/custos` ou `PUT /eventos/{id}/custos/{custoId}`. Erro: "Houve um erro ao salvar o custo". Sucesso: "Custo salvo com sucesso". A exclusão chama `DELETE /eventos/{id}/custos/{custoId}`; erro: "Houve um erro ao excluir o custo".

**Tela de "Produtos do evento"** (padrão de tela)
- Campo de pesquisa e filtro (por nome da categoria) chamam `GET /eventos/{id}/produtos`.
- Lista de produtos (padrão de lista): vazio "Não há produtos neste evento"; erro "Houve um erro ao buscar produtos do evento".
- Botão "Adicionar produto": só com o evento `NAO_INICIADO`. Abre o modal "Adicionar produto ao evento" (padrão de modal de ação e de validação), com duas opções:
  - "Do estoque": botão de "Selecionar produto", que abre um modal com busca dos produtos do estoque para selecionar (obrigatório, com busca em `GET /produtos`), "Preço de venda" (maior ou igual a 0) e "Quantidade prevista" (maior que 0).
  - "Produzido no evento": "Upload de foto" (opcional), "Nome" (obrigatório, máximo de 150 caracteres), "Descrição" (opcional), select "Categoria" (opcional, lista as categorias existentes), "Preço de venda" (maior ou igual a 0) e "Custo total dos ingredientes" (maior ou igual a 0).
  - Botão "Adicionar": chama `POST /eventos/{id}/produtos`. Erro: "Houve um erro ao adicionar o produto". Sucesso: "Produto adicionado com sucesso".
- Card de produto: mostra nome, preço, quantidade prevista, vendida e restante, com os botões:
  - Itens produzidos no evento mostram a etiqueta "Produzido no evento" e "—" em previsto e restante.
  - "Ver detalhes": abre o modal com o snapshot (nome, descrição, categoria, foto e, nos produzidos, o custo total dos ingredientes).
  - "Editar": só com o evento `NAO_INICIADO`. Em itens do estoque, edita preço e quantidade prevista; em itens produzidos, edita nome, descrição, categoria, preço e custo total dos ingredientes. Chama `PUT /eventos/{id}/produtos/{produtoEventoId}`.
  - "Remover": só com o evento `NAO_INICIADO` e item nunca vendido. Pede confirmação e chama `DELETE /eventos/{id}/produtos/{produtoEventoId}`.

**Tela de "Histórico de vendas"** (padrão de tela, de pesquisa e de paginação)
- Chama `GET /eventos/{id}/vendas`. Filtros: período e situação ("Todas", "Ativas" ou "Canceladas"), que se traduzem no parâmetro `cancelada` (ausente, `false` ou `true`). Vazio: "Não há vendas neste evento". Erro: "Houve um erro ao buscar o histórico de vendas".
- Card de venda: mostra data e hora, quantidade de itens, desconto, total e a etiqueta "Cancelada" quando for o caso, com o botão "Ver detalhes", que abre o modal de detalhes (`GET /eventos/{id}/vendas/{vendaId}`).
- Botão de cancelar venda: aparece para venda não cancelada quando o usuário tiver `VENDAS_MENU` e o evento **não** estiver `ENCERRADO`. Pede confirmação e chama `POST /eventos/{id}/vendas/{vendaId}/cancelamento`. Em 409, mostra o motivo no modal de erro. O cancelamento é definitivo: não existe reativação de venda.

## Regras
- O usuário deve ter a permissão `EVENTOS_MENU` para acessar as telas e usar as funcionalidades da história. A exceção é `GET /eventos`, que exige `EVENTOS_LEITURA`.
- Evento do tipo `REUNIAO` representa só despesa: não tem cardápio, vendas nem caixa. Os custos entram como movimentações de estoque vinculadas ao evento e como custos adicionais.
- Só pode haver um evento do tipo `EVENTO` em andamento por vez. Isso vale para iniciar e para reabrir. Telas que operam sobre o evento em andamento (registro de venda e caixa) não têm seletor de evento.
- Venda cancelada deixa de contar em quantidade vendida, receita, relatórios e dashboards. Uma venda não pode ser cancelada se o evento estiver `ENCERRADO`; reabrir o evento volta a permitir o cancelamento.
- Evento `ENCERRADO` pode ser reaberto pelo `PATCH /eventos/{id}`, voltando a `EM_ANDAMENTO`, mesmo com registros operacionais. Os dados registrados antes do encerramento (vendas e itens, cancelamentos, descontos, caixa, valores a receber, doações, movimentações de estoque e custos adicionais) permanecem associados ao mesmo evento e preservados.
- Não existe histórico explícito de reaberturas. Sem ele, o sistema não garante reconstruir todas as datas de encerramento e reabertura anteriores.
- `encerrado_em` representa apenas o encerramento vigente: é limpo ao reabrir e preenchido no próximo encerramento. A data do encerramento anterior não é preservada. `iniciado_em` não é alterado.
- Evento reaberto passa a ser tratado como `EM_ANDAMENTO` em telas, relatórios e filtros.
- Os custos adicionais (limpeza, pagamento a quem ajudou etc.) são informados ao encerrar o evento e podem ser editados e excluídos depois, com o evento `EM_ANDAMENTO` ou `ENCERRADO`. Contas de luz, água, gás e similares são custo mensal e ficam fora do escopo.
- Produtos produzidos no evento só podem ser incluídos com o evento `NAO_INICIADO`, não têm quantidade definida, não são cadastrados no estoque e não geram movimentação de estoque. O custo total dos ingredientes é editável até o evento começar.
- A lista de eventos mostra o evento em andamento no topo, com destaque.

## Protótipo
- Existe, exceto a ação "Reabrir evento" e sua confirmação, o modal "Encerrar evento" com custos adicionais, a seção "Custos adicionais", o cadastro de produto produzido no evento e o destaque do evento em andamento, a definir.

## Premissas
- As tabelas estão criadas no banco de dados.
- Eventos duram 1 dia ou 2 dias seguidos (sábado e domingo) e acontecem sempre dentro do Kaikan. Cada um é um único registro de evento.
- Os produtos do estoque já estão cadastrados na tela de estoque. Produtos produzidos no evento são criados direto no cardápio do evento.

## Sugestão de subtarefas
- BD: migrations de `evento` (com índice único parcial para um evento em andamento), `status_evento`, `produto_evento` (com `origem`, desconto e custo dos ingredientes) e `custo_evento`, e carga dos 3 status.
- BE: `GET /eventos` (permissão `EVENTOS_LEITURA` e ordenação) e `GET /eventos/{id}`.
- BE: `POST /eventos` e `PUT /eventos/{id}`.
- BE: `PATCH /eventos/{id}` (máquina de transição de status, reabertura, regra de um evento em andamento e custos ao encerrar).
- BE: CRUD de `/eventos/{id}/custos`.
- BE: `DELETE /eventos/{id}`.
- BE: CRUD de `/eventos/{id}/produtos` (itens do estoque e produzidos) e `/produtos/categorias`.
- BE: `GET /eventos/{id}/vendas` (com filtro `cancelada` e paginação) e `GET /eventos/{id}/vendas/{vendaId}`.
- FE: tela "Eventos" (lista com destaque do evento em andamento, pesquisa, filtros, card e ações).
- FE: ação "Reabrir evento" no card, com modal de confirmação.
- FE: modal de criar e editar evento.
- FE: modal "Encerrar evento" com custos adicionais.
- FE: tela "Detalhes do Evento" com a seção e o modal de custos adicionais.
- FE: tela "Produtos do evento" e modais (adicionar do estoque ou produzido, detalhes, editar, remover).
- FE: tela "Histórico de vendas" (filtros, paginação) e modal de detalhes com cancelamento.

---

# H3 — Gestão de estoque

## Objetivo

**Eu**, enquanto administrador  
**Quero** cadastrar os produtos, gerenciar as categorias deles e registrar as movimentações de entrada e saída, cada uma classificada por tipo  
**Para** ter o histórico e o registro de todas as movimentações do estoque, saber o que sobrou e ser avisado do que está faltando.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de pesquisa e filtros**
- Campo de pesquisa e botão de pesquisar chamam o endpoint de listagem da tela, passando o texto no parâmetro `q` e os filtros selecionados.
- Botão "Filtrar" com o painel fechado: muda o ícone e mostra abaixo as opções de filtro.
- Botão "Filtrar" com o painel aberto: muda o ícone e esconde as opções, **sem limpar** os filtros já aplicados.

**Padrão de lista**
- Enquanto carrega, mostra ícone de carregamento.
- Se não houver registros, mostra a mensagem de lista vazia indicada na tela.
- Em caso de erro, mostra um toast de erro e renderiza na lista a mensagem de erro indicada na tela.
- Em caso de sucesso, renderiza um Card para cada registro retornado.

**Padrão de paginação** (listas paginadas desta história)
- Parâmetros `page` (começa em 0; o frontend exibe +1) e `size` (padrão 20, máximo 100).
- Resposta: 
  ```json
  {
     "itens":  [
      ...
    ],
     "page":  number,
     "size":  number,
     "totalItens":  number,
     "totalPaginas":  number
  }
  ```

- Frontend: botões "Anterior" e "Próxima" e o texto "Página X de Y". "Anterior" fica desabilitado na primeira página e "Próxima" na última. Não há "carregar mais".

**Padrão de modal de ação** (criar, editar, excluir, movimentar)
- Ao confirmar, o texto do botão é trocado por um ícone de carregamento.
- Em caso de erro, mostra um modal de erro e **não fecha nem limpa** o modal da ação.
- Em caso de sucesso, mostra um modal de sucesso, fecha o modal da ação e limpa seus campos.

**Padrão de validação de formulário**
- O front valida os campos antes de chamar a API.
- Campo inválido recebe borda vermelha e a mensagem do erro logo abaixo.
- Erros de validação devolvidos pela API (HTTP 400) são exibidos no campo correspondente.

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão `ESTOQUE_MENU`. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 403 sem permissão, 404 não encontrado, 409 conflito (duplicidade ou regra de negócio violada).
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Criar as tabelas: `categoria`, `produto`, `estoque`, `tipo_movimentacao`, `movimentacao_estoque`. Todas têm `criado_em` e `atualizado_em`.
- `categoria` tem: `nome` (único, garantido também no banco, comparando sem diferenciar maiúsculas/minúsculas e sem espaços externos) e `descricao` (opcional, com o limite máximo do modelo de dados existente, o mesmo da `descricao` de `produto`).
- `produto` tem: `nome` (único), `descricao`, `categoria`, `foto_url`, `quantidade_atual`, `estoque_minimo`, `ativo` (para exclusão lógica quando houver histórico). `quantidade_atual` é um valor derivado: a soma de `estoque.quantidade_atual` dos lotes do produto, atualizada na mesma transação de cada movimentação. O produto não tem unidade de medida nem custo médio.
- `estoque` representa um **lote** de um produto e tem: `produto_id`, `quantidade_inicial`, `quantidade_atual`, `custo_unitario`, `custo_total` e `criado_em`. `custo_total` é o valor pago pelo lote inteiro (`quantidade_inicial` × `custo_unitario`), definido na entrada do lote. Lote sem custo informado (doação e ajuste de entrada) tem `custo_unitario` e `custo_total` nulos. Lote zerado permanece como histórico.
- `tipo_movimentacao` tem: `codigo`, `descricao`, `direcao` (entrada ou saída), `exige_custo` (só se aplica a entradas), indicador de tipo gerado pelo sistema, indicador de ativo.
- `movimentacao_estoque` guarda `estoque_id` (lote, obrigatório), a quantidade **com sinal**, `custo_unitario`, `evento_id`, `observacoes`, `criado_em` e as chaves opcionais `item_venda_id` e `doacao_id`. Nas entradas, `custo_unitario` é o custo informado (ou nulo). Nas saídas e vendas, é o custo do lote consumido.
- Carga inicial (migration): categorias iniciais (ex.: Bebida, Comida) e os tipos de movimentação. Depois da carga, as categorias são gerenciadas pelo Menu de categorias.
- Tipos de movimentação usados no sistema: `REABASTECIMENTO`, `CONSUMO_INTERNO`, `PERDA`, `AJUSTE_ENTRADA`, `AJUSTE_SAIDA` (manuais) e `VENDA`, `DOACAO`, `ESTORNO_VENDA` (gerados pelo sistema).
  - `PERDA`: baixa de produto perdido (estragou, quebrou, venceu). O motivo é informado no campo opcional "Observações".
  - `AJUSTE_ENTRADA` e `AJUSTE_SAIDA`: correção de divergência entre a contagem física e o estoque do sistema (acerto de inventário), sem relação com compra, venda ou doação. O ajuste de entrada cria um lote sem custo.

### Backend

**Criar endpoint: `GET /produtos`**
- Parâmetros: `q` (nome), `categoriaId` (aceita mais de uma categoria, repetindo o parâmetro: `categoriaId=1&categoriaId=2`; o produto aparece se pertencer a qualquer uma delas), `abaixoDoMinimo` (boolean).
- Retorna 
  ```json
  [
    {
       id,
       nome,
       descricao,
       categoria:  {
         id,
         nome
      },
       fotoUrl,
       quantidadeAtual,
       estoqueMinimo
    }
  ]
  ```
  , em que `quantidadeAtual` é a soma dos lotes.

**Criar endpoint: `GET /produtos/{id}`**
- 404 com corpo vazio se não existir. Se existir, retorna o mesmo objeto da listagem.

**Criar endpoint: `POST /produtos`**
- Corpo: 
  ```json
  {
     "nome":  string,
     "descricao":  string,
     "categoriaId":  number,
     "estoqueMinimo":  number
  }
  ```

  - `nome`: não nulo ou em branco, máximo de 150 caracteres.
  - `estoqueMinimo`: opcional, maior ou igual a 0.
  - `descricao` e `categoriaId`: opcionais.
- 409 se já existir produto com o mesmo `nome`.
- Se não existir, insere com `quantidade_atual = 0` e sem lotes, e retorna 201 com o produto e o id. Todo produto pode ser movimentado.

**Criar endpoint: `POST /produtos/{id}/foto`**
- Upload multipart, opcional. Guarda a chave de armazenamento em `foto_url` e retorna a URL de acesso. 404 se o produto não existir.

**Criar endpoint: `PUT /produtos/{id}`**
- Corpo igual ao do `POST /produtos`. 404 se não existir. 409 se o novo nome já for de outro produto. Retorna 204.
- Não altera `quantidade_atual` nem os lotes: só movimentações alteram.

**Criar endpoint: `DELETE /produtos/{id}`**
- 404 com corpo vazio se não existir.
- Se o produto não tiver vínculos históricos ou operacionais que precisem ser preservados, permite exclusão definitiva.
- Se houver vínculo com eventos/cardápios, vendas, doações ou lotes ou movimentações de estoque, faz exclusão lógica (`ativo = false`), preservando dados e referências históricas. Produto inativo não pode ser selecionado para novos cadastros, mas continua disponível para consultas históricas. Retorna 204.

**Criar endpoint: `GET /categorias`**
- Retorna 200 com `[{ id, nome, descricao }]`, ordenada por `nome`, para o Menu de categorias, o select do formulário e os filtros. Lista vazia segue o padrão de resposta já adotado pela API.
- A pesquisa por texto é feita no frontend sobre a lista carregada (a lista é pequena); o endpoint não recebe parâmetro de pesquisa.

**Criar endpoint: `POST /categorias`**
- Corpo: 
  ```json
  {
     "nome":  string,
     "descricao":  string
  }
  ```

  - `nome`: não nulo ou em branco, máximo de 255 caracteres. Espaços no início e no fim são removidos.
  - `descricao`: opcional, com o limite máximo do modelo de dados existente (o mesmo da `descricao` de `produto`).
- 409 se já existir categoria com o mesmo `nome` (comparação sem diferenciar maiúsculas/minúsculas e sem espaços externos, aplicada igualmente no backend e no banco).
- Retorna 201 com a categoria criada: `{ id, nome, descricao }`.

**Criar endpoint: `PUT /categorias/{id}`**
- Corpo igual ao do `POST /categorias`, com as mesmas validações de nome, normalização e unicidade. 404 se não existir. 409 se o novo nome já for de outra categoria. Retorna 200 com a categoria atualizada: `{ id, nome, descricao }`.
- Não altera snapshots históricos de produtos, vendas, eventos ou outros registros.

**Criar endpoint: `DELETE /categorias/{id}`**
- 404 se não existir.
- Se houver pelo menos um produto vinculado à categoria, ou outro vínculo que impeça a exclusão, rejeita com 409 e a categoria permanece inalterada. Exemplo: `{ "codigo": "CATEGORIA_COM_PRODUTOS", "mensagem": "Não é possível excluir esta categoria porque existem produtos vinculados a ela." }`.
- Não exclui produtos nem os desvincula automaticamente. Cardápios de eventos guardam só o nome da categoria (snapshot) e não impedem a exclusão.
- A verificação e a exclusão ocorrem na mesma transação, protegidas contra concorrência e respeitando as restrições de integridade do banco.
- Sem vínculos, exclui a categoria e retorna 204.

**Criar endpoint: `GET /tipos-movimentacao`**
- Retorna os tipos ativos que podem ser usados manualmente no modal "Movimentar Estoque": `[{ id, codigo, descricao, direcao, exigeCusto }]`.
- Os tipos gerados pelo sistema (`VENDA`, `DOACAO`, `ESTORNO_VENDA`) não são listados nem aceitos em `POST /produtos/{id}/movimentacoes`.

**Criar endpoint: `POST /produtos/{id}/movimentacoes`**
- Corpo: 
  ```json
  {
     "tipoMovimentacaoId":  number,
     "quantidade":  number,
     "custoUnitario":  number,
     "eventoId":  number,
     "observacoes":  string
  }
  ```

  - `quantidade`: sempre positiva (o sentido vem do tipo).
  - `custoUnitario`: só se aplica a entradas. Obrigatório e maior que 0 se `tipo.exigeCusto` (hoje, `REABASTECIMENTO`); nos demais tipos é ignorado. Nas saídas, o custo vem do lote consumido.
  - `eventoId`: opcional.
  - `observacoes`: opcional. Em `PERDA`, registra o motivo da perda.
  - `CONSUMO_INTERNO` exige `eventoId` de um evento do tipo `REUNIAO`.
- Erros: 404 se o produto não existir. 400 se o tipo não existir, estiver inativo ou for gerado pelo sistema.
- Entrada, na mesma transação:
  - cria um lote (`estoque`) com `quantidade_inicial` = `quantidade_atual` = `quantidade`, `custo_unitario` = `custoUnitario` (nulo se o tipo não exigir custo) e `custo_total` = `quantidade` × `custoUnitario` (nulo se sem custo). Entrada nunca soma a um lote existente;
  - grava a movimentação de entrada (quantidade positiva) com o `estoque_id` do novo lote e o `custo_unitario`;
  - atualiza `produto.quantidade_atual`.
- Saída, na mesma transação:
  - valida que a soma dos saldos dos lotes cobre a quantidade. Se não cobrir, rejeita (409) sem alterar nenhum lote nem o histórico. Vale para `CONSUMO_INTERNO`, `PERDA` e `AJUSTE_SAIDA`;
  - consome os lotes com saldo do mais antigo para o mais novo (`criado_em` crescente, desempate por `id`), travando os lotes nessa ordem para evitar concorrência, e reduz `estoque.quantidade_atual`;
  - grava uma movimentação por lote consumido (quantidade negativa, `estoque_id` e `custo_unitario` do lote);
  - atualiza `produto.quantidade_atual`.
- Retorna 201 com 
  ```json
  {
     movimentacoes:  [
      ...
    ],
     quantidadeAtual
  }
  ```
  . O estoque nunca pode ficar negativo; o backend valida essa regra mesmo quando o frontend também valida.

**Criar endpoint: `GET /produtos/{id}/movimentacoes`**
- Histórico paginado (padrão de paginação) de todos os lotes do produto, do mais recente para o mais antigo.
- Parâmetros: `tipoMovimentacaoId`, `eventoId`, `dataInicio`, `dataFim`.
- Retorna `[{ id, estoqueId, tipo, quantidade, custoUnitario, evento, observacoes, criadoEm }]` dentro do envelope de paginação.

**Criar endpoint: `GET /produtos/{id}/estoques`**
- Lista os lotes do produto, do mais antigo para o mais novo, inclusive os zerados: `[{ id, quantidadeInicial, quantidadeAtual, custoUnitario, custoTotal, criadoEm }]`. 404 se o produto não existir.

**Criar endpoint: `GET /estoques/{id}/movimentacoes`**
- Histórico paginado (padrão de paginação) das movimentações de um lote, do mais recente para o mais antigo: `[{ id, tipo, quantidade, custoUnitario, evento, observacoes, criadoEm }]` dentro do envelope de paginação. 404 se o lote não existir.

### Endpoints consumidos de outras histórias
- `GET /eventos?tipoEvento=REUNIAO` (dona: gestão de eventos). Retorna `[{ id, nome, tipoEvento, status, dataHoraInicio, dataHoraEncerramento }]`. Usado no select "Evento" do modal "Movimentar Estoque". Exige `EVENTOS_LEITURA`.

### Frontend

**Tela de "Estoque"** (padrão de tela)
- Campo de pesquisa e botão de pesquisar (padrão de pesquisa): chamam `GET /produtos`. Filtros: categoria (seleção múltipla) e "Somente abaixo do mínimo".
- Botão "Menu de categorias": fica junto aos controles de Estoque e abre o modal "Menu de categorias".
- Botão "Criar produto": abre o modal de criação (padrão de modal de ação e de validação):
  - Upload de imagem: opcional.
  - "Nome": não nulo ou em branco, máximo de 150 caracteres.
  - "Descrição": opcional.
  - Select "Categoria": opcional.
  - "Estoque mínimo": opcional, maior ou igual a 0.
  - Botão "Criar produto": valida os campos e chama `POST /produtos` (e `POST /produtos/{id}/foto` se houver imagem). Erro: "Houve um erro ao criar o produto". Sucesso: "Produto criado com sucesso".
- Lista de produtos (padrão de lista): chama `GET /produtos`. Vazio: "Não há produtos cadastrados". Erro: "Houve um erro ao buscar estoque".
- Card de produto: mostra nome, categoria, quantidade atual e a etiqueta "Estoque baixo" (quantidade atual menor ou igual ao mínimo). Botões:
  - "Ver detalhes": abre o modal "Detalhes do produto".
  - "Movimentar Estoque": abre o modal "Movimentar Estoque".
  - "Editar": abre o modal "Edição de produto".
  - "Excluir": abre o modal de confirmação de exclusão.

**Modal "Detalhes do produto"**
- Mostra os dados já carregados na lista (basta filtrar por id a lista de `GET /produtos`) e duas listas:
  - Movimentações do produto, de todos os lotes, exibidas por padrão, sem precisar escolher um lote (`GET /produtos/{id}/movimentacoes`, com paginação "Anterior" e "Próxima"): cada movimentação mostra o lote a que pertence. Carregando com ícone, vazio "Sem movimentações", erro "Houve um erro ao buscar as movimentações".
  - Lotes (`GET /produtos/{id}/estoques`): carregando com ícone, vazio "Sem lotes", erro "Houve um erro ao buscar os lotes". Cada lote mostra data de entrada, quantidade inicial, quantidade atual, custo unitário e custo total ("Sem custo" quando nulos) e o botão "Ver movimentações", que mostra as movimentações só daquele lote (`GET /estoques/{id}/movimentacoes`, com paginação): carregando com ícone, vazio "Sem movimentações", erro "Houve um erro ao buscar as movimentações".
- Botão "Movimentar Estoque": fecha este modal e abre o "Movimentar Estoque".

**Modal "Movimentar Estoque"** (padrão de modal de ação e de validação)
- Select "Tipo da movimentação" (`GET /tipos-movimentacao`): obrigatório.
- "Custo unitário": aparece só para entradas cujo tipo exija custo (hoje, `REABASTECIMENTO`). Obrigatório quando aparece. Só aceita números positivos. Nas saídas não aparece: o custo é o do lote consumido, definido pelo sistema.
- Select "Evento": opcional. Para `CONSUMO_INTERNO`, é obrigatório e só lista eventos do tipo Reunião.
- "Quantidade a movimentar": obrigatório, só números positivos.
- "Observações": opcional. Quando o tipo for `PERDA`, o rótulo passa a "Observações (motivo da perda)".
- "Quantidade após a movimentação": mostra, antes de confirmar, a quantidade atual do produto mais a quantidade (entradas) ou menos a quantidade (saídas). O cálculo é feito no frontend com a quantidade atual já carregada e se atualiza ao digitar a quantidade ou trocar o tipo. Se o resultado ficar negativo, mostra um aviso e desabilita a confirmação. O backend continua validando o saldo.
- Botão "Cancelar": fecha o modal.
- Botão "Movimentar Estoque": chama `POST /produtos/{id}/movimentacoes`. Erro: "Houve um erro ao movimentar o produto". Sucesso: "Movimentação realizada com sucesso". Se o saldo for insuficiente, mostra o erro retornado e não confirma a movimentação.

**Modal "Edição de produto"**
- Mesma lógica e campos do "Criar produto", preenchidos com os dados atuais, chamando `PUT /produtos/{id}`. Erro: "Houve um erro ao atualizar o produto". Sucesso: "Produto atualizado com sucesso".

**Modal de confirmação de exclusão**
- Botão "Confirmar exclusão": chama `DELETE /produtos/{id}`. Enquanto carrega, mostra ícone de carregamento. Erro: "Houve um erro ao excluir o produto" (informa se o produto será desativado por possuir histórico). Sucesso: "Produto excluído com sucesso".
- Botão "Cancelar": fecha o modal.

**Modal "Menu de categorias"** (padrão de modal e de lista)
- Aberto pelo botão "Menu de categorias" da tela "Estoque". Os modais de criar, editar e excluir categoria abrem sobre ele. Chama `GET /categorias` ao abrir. Vazio: "Não há categorias cadastradas". Erro: "Houve um erro ao buscar categorias". Falha na consulta nunca é exibida como lista vazia.
- Campo de pesquisa por texto: filtra no frontend a lista já carregada, sem chamar o backend (exceção ao padrão de pesquisa, por ser lista pequena).
- Botão "Criar categoria": abre o modal "Criar categoria".
- Card de categoria: mostra nome e descrição (quando houver), com os botões:
  - "Editar": abre o modal "Editar categoria".
  - "Excluir": abre o modal de confirmação de exclusão da categoria.
- A lista é mantida no estado do frontend. Criar, editar e excluir apenas salvam no backend; o resultado da ação atualiza a lista local, sem nova consulta de busca.

**Modal "Criar categoria"** (padrão de modal de ação e de validação)
- "Nome": não nulo ou em branco, máximo de 255 caracteres. Espaços no início e no fim são removidos antes do envio.
- "Descrição": opcional, com o limite máximo do modelo de dados existente.
- Botão "Cancelar": fecha o modal sem chamar a API. Se houver dados preenchidos, pode pedir confirmação antes de descartá-los.
- Botão "Criar categoria": valida os campos e chama `POST /categorias`. Fica desabilitado enquanto a requisição está em andamento, para evitar envios repetidos. Erro: "Houve um erro ao criar a categoria" (inclui nome duplicado, 409), mantendo os dados preenchidos. Sucesso: "Categoria criada com sucesso"; o modal fecha e a categoria devolvida pela API é inserida na lista local, mantendo a ordem por nome.
  - Em falha de comunicação, informa que não foi possível confirmar o cadastro e permite nova tentativa, sem repetir a requisição automaticamente. O frontend não assume sucesso antes da resposta da API.

**Modal "Editar categoria"** (padrão de modal de ação e de validação)
- Mesma lógica e campos do "Criar categoria", preenchidos com os dados atuais, chamando `PUT /categorias/{id}`. Erro: "Houve um erro ao atualizar a categoria", mantendo os dados digitados. Sucesso: "Categoria atualizada com sucesso"; o modal fecha e a categoria devolvida pela API substitui a da lista local.

**Modal de confirmação de exclusão de categoria**
- Botão "Confirmar exclusão": chama `DELETE /categorias/{id}`. Enquanto carrega, mostra ícone de carregamento. Erro: "Houve um erro ao excluir a categoria"; em 409, informa que existem produtos vinculados e a categoria permanece inalterada. Sucesso: "Categoria excluída com sucesso"; a categoria é removida da lista local só depois da confirmação do sucesso.
- Botão "Cancelar": fecha o modal.

**Atualização dos seletores de categoria**
- Depois de criar, editar ou excluir uma categoria, o select "Categoria" dos modais "Criar produto" e "Edição de produto" e o filtro de categoria da tela de Estoque usam a mesma lista local atualizada, sem nova consulta a `GET /categorias`.
- Erros da API são comunicados sem descartar dados ainda não salvos nesses formulários.

## Regras
- O usuário deve ter a permissão `ESTOQUE_MENU` para acessar a tela e usar as funcionalidades da história, inclusive registrar movimentações.
- Todo saldo de estoque pertence a um lote. Toda entrada cria um lote novo, e toda saída (manual ou venda) consome os lotes do mais antigo para o mais novo (FIFO), gerando uma movimentação por lote consumido. Não há escolha manual de lote.
- Todo produto pode ser movimentado.
- O estoque nunca pode ficar negativo. O backend rejeita saídas e vendas que excedam a soma dos saldos dos lotes.
- `produto.quantidade_atual` é sempre a soma dos saldos dos lotes do produto.
- O custo pertence ao lote. A entrada com custo (`REABASTECIMENTO`) cria um lote com custo; as entradas sem custo (`DOACAO` e `AJUSTE_ENTRADA`) criam lote com custo nulo. O sistema não calcula custo médio. Nas saídas e vendas, o custo da movimentação é o do lote consumido.
- O estorno de venda devolve a quantidade ao lote de origem, mesmo que ele esteja zerado.
- A edição de quantidade de uma doação de reabastecimento atualiza a movimentação `DOACAO` original e o lote vinculado, na mesma transação. É rejeitada (409) se a nova quantidade ficar abaixo do que já foi consumido do lote.
- O motivo de uma perda é registrado no campo opcional "Observações" da movimentação `PERDA`.
- Categoria com produtos vinculados não pode ser excluída. O backend bloqueia a exclusão mesmo que o frontend tenha desabilitado o botão, e o sistema nunca exclui nem desvincula produtos como consequência da exclusão de uma categoria.
- O nome da categoria é obrigatório, único e tem no máximo 255 caracteres; a descrição é opcional, com o limite máximo do modelo de dados existente.
- Criar, editar ou excluir categoria não reescreve snapshots históricos de produtos, vendas, eventos ou outros registros. O snapshot de `produto_evento` guarda a categoria pelo nome (texto).

## Protótipo
- Existe, exceto o Menu de categorias e seus modais, a lista de lotes no modal "Detalhes do produto", a pré-visualização da quantidade no modal "Movimentar Estoque" e o filtro de categoria com seleção múltipla, a definir.

## Premissas
- As tabelas do módulo (inclusive `estoque`) serão criadas já no modelo de lotes. Não há dados nem código existentes, portanto não há migração de dados.
- Todos os produtos do estoque são alimentos ou bebidas prontos para venda. Ingredientes e itens não comestíveis (ex.: limpeza e prêmios) não fazem parte do estoque. Produtos feitos na hora com ingredientes (ex.: tempurá) existem só no cardápio do evento, sem passar pelo estoque.
- As categorias iniciais são carregadas por migration e, depois, gerenciadas pelo Menu de categorias (criar, editar e excluir).

## Sugestão de subtarefas
- BD: migrations das 5 tabelas (com `descricao` e unicidade do nome em `categoria`, e `estoque` com os campos de lote) e carga de categorias e tipos de movimentação.
- BE: `GET /produtos` (filtro por várias categorias) e `GET /produtos/{id}`.
- BE: `POST /produtos`, `PUT /produtos/{id}` e `POST /produtos/{id}/foto`.
- BE: `DELETE /produtos/{id}`.
- BE: `GET /categorias` (com `descricao`) e `GET /tipos-movimentacao`.
- BE: `POST /categorias` e `PUT /categorias/{id}` (normalização e unicidade do nome; ambos devolvem a categoria).
- BE: `DELETE /categorias/{id}` (bloqueio com produtos vinculados).
- BE: `POST /produtos/{id}/movimentacoes` (criação de lote na entrada, consumo FIFO na saída, validação de saldo não negativo).
- BE: `GET /produtos/{id}/movimentacoes`, `GET /produtos/{id}/estoques` e `GET /estoques/{id}/movimentacoes` (com paginação).
- FE: tela "Estoque" (lista, pesquisa, filtros e card).
- FE: modal de criar e editar produto.
- FE: modal "Detalhes do produto" (movimentações de todos os lotes, lotes e movimentações por lote).
- FE: modal "Movimentar Estoque" (com a quantidade após a movimentação).
- FE: modal de confirmação de exclusão.
- FE: botão "Menu de categorias" e modal "Menu de categorias" (lista e pesquisa no frontend).
- FE: modais "Criar categoria" e "Editar categoria".
- FE: modal de confirmação de exclusão de categoria.
- FE: atualização local dos seletores de categoria nos formulários de produto.

---

# H4 — Controle de doações

## Objetivo

**Eu**, enquanto administrador  
**Quero** registrar, editar e listar as doações recebidas, identificando o doador e o valor da ficha entregue como agradecimento  
**Para** ter um registro sobre as doações e o seu custo para a associação.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de pesquisa e filtros**
- Campo de pesquisa e botão de pesquisar chamam o endpoint de listagem da tela, passando o texto no parâmetro `q` e os filtros selecionados.
- Botão "Filtrar" com o painel fechado: muda o ícone e mostra abaixo as opções de filtro.
- Botão "Filtrar" com o painel aberto: muda o ícone e esconde as opções, **sem limpar** os filtros já aplicados.

**Padrão de lista**
- Enquanto carrega, mostra ícone de carregamento.
- Se não houver registros, mostra a mensagem de lista vazia indicada na tela.
- Em caso de erro, mostra um toast de erro e renderiza na lista a mensagem de erro indicada na tela.
- Em caso de sucesso, renderiza um Card para cada registro retornado.

**Padrão de paginação** (listas paginadas desta história)
- Parâmetros `page` (começa em 0; o frontend exibe +1) e `size` (padrão 20, máximo 100).
- Resposta: 
  ```json
  {
     "itens":  [
      ...
    ],
     "page":  number,
     "size":  number,
     "totalItens":  number,
     "totalPaginas":  number
  }
  ```

- Frontend: botões "Anterior" e "Próxima" e o texto "Página X de Y". "Anterior" fica desabilitado na primeira página e "Próxima" na última. Não há "carregar mais".

**Padrão de modal de ação** (registrar, editar)
- Ao confirmar, o texto do botão é trocado por um ícone de carregamento.
- Em caso de erro, mostra um modal de erro e **não fecha nem limpa** o modal da ação.
- Em caso de sucesso, mostra um modal de sucesso, fecha o modal da ação e limpa seus campos.

**Padrão de validação de formulário**
- O front valida os campos antes de chamar a API.
- Campo inválido recebe borda vermelha e a mensagem do erro logo abaixo.
- Erros de validação devolvidos pela API (HTTP 400) são exibidos no campo correspondente.

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão `DOACOES_MENU`. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 403 sem permissão, 404 não encontrado, 409 conflito (duplicidade ou regra de negócio violada).
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Criar a tabela `doacao` com: `evento_id`, `tipo` (`dinheiro`, `produto` ou `reabastecimento`), `valor`, `forma_pagamento`, `produto_id` (só em `reabastecimento`), `nome` (descrição do item, só em `produto`), `quantidade`, `unidade_medida` (só em `produto`: `un`, `kg` ou `L`), `doador`, `valor_ficha`, `criado_em` e `atualizado_em`. `criado_em` é gerado pelo sistema e não pode ser editado.
- A tabela `movimentacao_estoque` (história de estoque) recebe uma coluna `doacao_id` opcional, usada pela movimentação `DOACAO`. Só doações do tipo `reabastecimento` geram movimentação e lote.

### Backend

**Criar endpoint: `GET /doacoes`**
- Parâmetros: `q` (doador ou nome do item), `tipo` e `eventoId`. `tipo` e `eventoId` aceitam mais de um valor ao mesmo tempo (parâmetro repetido, ex.: `tipo=dinheiro&tipo=produto`), e os filtros se combinam (o registro precisa casar com algum tipo e com algum evento informados). Não há filtro por período.
- Resposta paginada (padrão de paginação), da doação mais recente para a mais antiga: `[{ id, eventoId, tipo, valor, formaPagamento, produtoId, nome, quantidade, unidadeMedida, doador, valorFicha, criadoEm, atualizadoEm }]` dentro do envelope de paginação.

**Criar endpoint: `POST /doacoes`**
- Corpo:
  ```json
  {
     "eventoId":  number,
     "tipo":  "dinheiro" | "produto" | "reabastecimento",
     "valor":  number,
     "formaPagamento":  string,
     "produtoId":  number,
     "nome":  string,
     "quantidade":  number,
     "unidadeMedida":  string,
     "doador":  string,
     "valorFicha":  number
  }
  ```
  - `eventoId`: obrigatório, de um evento que exista e não esteja `ENCERRADO`.
  - `tipo`: deve ser `dinheiro`, `produto` ou `reabastecimento`.
  - Se `tipo = dinheiro`: `valor` (maior que 0) e `formaPagamento` (`PIX`, `DEBITO`, `CREDITO` ou `DINHEIRO`) são obrigatórios.
  - Se `tipo = produto`: `nome` (descrição do item, obrigatório, máximo de 150 caracteres, ex.: "arroz" ou "bolo de cenoura"), `quantidade` (maior que 0) e `unidadeMedida` (`un`, `kg` ou `L`) são obrigatórios. Não informa produto do cadastro, não segue as regras de estoque (pode ser ingrediente) e **não gera movimentação nem lote**.
  - Se `tipo = reabastecimento`: `produtoId` (produto existente e ativo, que segue as regras de estoque) e `quantidade` (maior que 0) são obrigatórios. 
  - `doador`: não nulo ou em branco, máximo de 150 caracteres.
  - `valorFicha`: opcional, maior ou igual a 0. É o valor da ficha de agradecimento entregue e entra como custo do evento.
- Insere a doação. Se `tipo = reabastecimento`, gera, na mesma transação, um lote novo sem custo (`custo_unitario` e `custo_total` nulos, `quantidade_inicial` e `quantidade_atual` iguais à quantidade) e a movimentação de entrada `DOACAO` em `movimentacao_estoque`, com `doacao_id` e `estoque_id`, atualizando `produto.quantidade_atual`. Nos demais tipos, não há movimentação de estoque.
- Retorna 201 com a doação, o id e `criadoEm`.

**Criar endpoint: `PUT /doacoes/{id}`**
- Corpo igual ao do `POST /doacoes`. 404 se não existir.
- A edição é permitida em qualquer status do evento. `tipo` é imutável, e `criadoEm` é gerado pelo sistema e não pode ser alterado.
- Doação em dinheiro: permite alterar `eventoId`, `doador`, `valor`, `formaPagamento` e `valorFicha`.
- Doação de produto: permite alterar `eventoId`, `doador`, `nome`, `quantidade`, `unidadeMedida` e `valorFicha`.
- Doação de reabastecimento: permite alterar `eventoId`, `doador`, `quantidade` e `valorFicha`. `produtoId` é imutável.
- Se a quantidade de uma doação de reabastecimento mudar, atualiza, na mesma transação, a movimentação `DOACAO` original e o lote vinculado (`quantidade_inicial` passa a ser a nova quantidade e `quantidade_atual` é ajustada pela diferença), além de `produto.quantidade_atual`. Se a nova quantidade ficar abaixo do que já foi consumido do lote, rejeita com 409 e nada é alterado. Não cria movimentação adicional de ajuste ou estorno.
- Retorna 204.

### Endpoints consumidos de outras histórias
- `GET /eventos?statusId=<NAO_INICIADO e EM_ANDAMENTO>` (dona: gestão de eventos). Retorna `[{ id, nome, status, dataHoraInicio }]`. Usado no select "Evento" e no filtro de evento.
- `GET /produtos?q=` (dona: gestão de estoque). Retorna `[{ id, nome }]`. Usado no select "Produto" do tipo reabastecimento.
- `POST /produtos` (dona: gestão de estoque). Corpo mínimo: 
  ```json
  {
     "nome":  string
  }
  ```
  . Retorna 201 com o produto criado. Usado pela opção "Criar produto" dentro do modal de doação, no tipo reabastecimento.
- Essas leituras exigem `EVENTOS_LEITURA` e `PRODUTOS_LEITURA`, sem conceder permissões de edição.

### Frontend

**Tela de "Doações"** (padrão de tela)
- Campo de pesquisa e botão de pesquisar (padrão de pesquisa): chamam `GET /doacoes`. Filtros: tipo e evento, ambos com seleção múltipla. Não há filtro de período.
- Lista de doações (padrão de lista): chama `GET /doacoes`. Vazio: "Não há doações cadastradas". Erro: "Houve um erro ao buscar doações". Cada Card mostra doador, tipo, valor ou item, valor da ficha e evento (sem data), com o botão "Editar". A lista é paginada (padrão de paginação), com os botões "Anterior" e "Próxima".
- Botão "Registrar Doação": abre o modal "Registrar Doação" (padrão de modal de ação e de validação):
  - Select "Evento": obrigatório, só eventos não encerrados.
  - Opções de tipo "Dinheiro", "Produto" e "Reabastecimento": ao selecionar uma, mostra só os campos dela e oculta os das outras.
  - Tipo "Dinheiro": "Valor" (só números positivos maiores que 0) e select "Forma de pagamento" (obrigatório).
  - Tipo "Produto": "Descrição do item" (obrigatório, máximo de 150 caracteres), "Quantidade" (só números positivos maiores que 0) e select "Unidade de medida" (`un`, `kg` ou `L`, obrigatório).
  - Tipo "Reabastecimento": select "Produto" (obrigatório, com busca) e "Quantidade" (só números positivos maiores que 0). O select oferece "Criar produto" somente se o usuário também tiver permissão de edição de estoque (`ESTOQUE_MENU`); o produto criado segue as regras de estoque.
  - "Doador": não nulo ou em branco, máximo de 150 caracteres.
  - "Valor da ficha de agradecimento": opcional, maior ou igual a 0.
  - Botão "Registrar": chama `POST /doacoes`. Erro: "Houve um erro ao registrar a doação". Sucesso: "Doação registrada com sucesso".
- Modal "Editar Doação": mesma lógica do modal de registro, com "Tipo" e "Produto" desabilitados. A quantidade fica habilitada nos tipos "Produto" e "Reabastecimento"; no "Produto", a descrição do item e a unidade de medida também são editáveis. No "Reabastecimento", se a quantidade ficar abaixo do que já foi consumido do lote, mostra o erro retornado (409). `criadoEm` não é editável. Chama `PUT /doacoes/{id}`. Erro: "Houve um erro ao atualizar a doação". Sucesso: "Doação atualizada com sucesso".

## Regras
- O usuário deve ter a permissão `DOACOES_MENU` para acessar a tela e usar as funcionalidades da história.
- Só doações do tipo `reabastecimento` alteram o estoque (criam lote sem custo e a movimentação `DOACAO`). Doações de dinheiro e de produto não geram movimentação.

## Protótipo
- Existe, exceto o tipo "Reabastecimento" e o ajuste do tipo "Produto" (descrição e unidade de medida), a definir.

## Premissas
- Existe um protótipo.
- A tabela `doacao` está criada no banco de dados.
- O doador é um texto livre (o cadastro de associados está fora do escopo).
- Todas as doações são destinadas a eventos.
- A doação do tipo "Produto" não passa pelo estoque e pode ser ingrediente. A do tipo "Reabastecimento" só aceita produtos que seguem as regras de estoque (alimentos ou bebidas prontos para venda).

## Sugestão de subtarefas
- BD: migration de `doacao` (com `unidade_medida` e o tipo `reabastecimento`) e coluna `doacao_id` em `movimentacao_estoque`.
- BE: `GET /doacoes` (filtros múltiplos de tipo e evento, resposta paginada).
- BE: `POST /doacoes` (inclui lote sem custo e movimentação `DOACAO` apenas no tipo `reabastecimento`).
- BE: `PUT /doacoes/{id}` (atualização do lote e da movimentação com a validação do consumido).
- FE: tela "Doações" (lista paginada, pesquisa, filtros e card sem data).
- FE: modal "Registrar Doação" (alternância entre dinheiro, produto e reabastecimento, criar produto na hora).
- FE: modal "Editar Doação".

---

# H5 — Controle de vendas e caixa

## Objetivo

**Eu**, enquanto operador do baiten  
**Quero** registrar as vendas (com desconto e baixa automática no estoque) e os lançamentos de caixa (venda e devolução de fichas, formas de pagamento e fiados) durante o evento em andamento  
**Para** ter dados detalhados das vendas e do desempenho do baiten e o histórico de tudo o que foi vendido.

A "abertura e fechamento de caixa" acontece com o início e o encerramento do evento: o caixa só aceita lançamentos com o evento `EM_ANDAMENTO` (a quitação de fiado é a exceção).
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de pesquisa e filtros**
- Campo de pesquisa e botão de pesquisar chamam o endpoint de listagem da tela, passando o texto no parâmetro `q` e os filtros selecionados.
- Botão "Filtrar" com o painel fechado: muda o ícone e mostra abaixo as opções de filtro.
- Botão "Filtrar" com o painel aberto: muda o ícone e esconde as opções, **sem limpar** os filtros já aplicados.

**Padrão de lista**
- Enquanto carrega, mostra ícone de carregamento.
- Se não houver registros, mostra a mensagem de lista vazia indicada na tela.
- Em caso de erro, mostra um toast de erro e renderiza na lista a mensagem de erro indicada na tela.
- Em caso de sucesso, renderiza um Card para cada registro retornado.

**Padrão de paginação** (listas paginadas desta história)
- Parâmetros `page` (começa em 0; o frontend exibe +1) e `size` (padrão 20, máximo 100).
- Resposta: 
  ```json
  {
     "itens":  [
      ...
    ],
     "page":  number,
     "size":  number,
     "totalItens":  number,
     "totalPaginas":  number
  }
  ```

- Frontend: botões "Anterior" e "Próxima" e o texto "Página X de Y". "Anterior" fica desabilitado na primeira página e "Próxima" na última. Não há "carregar mais".

**Padrão de ação** (confirmar venda, aplicar desconto, registrar no caixa, quitar fiado)
- Ao confirmar, o texto do botão é trocado por um ícone de carregamento.
- Em caso de erro, mostra a mensagem de erro indicada e **não limpa** os campos preenchidos.
- Em caso de sucesso, mostra a mensagem de sucesso indicada e limpa os campos.

**Padrão de validação de formulário**
- O front valida os campos antes de chamar a API.
- Campo inválido recebe borda vermelha e a mensagem do erro logo abaixo.
- Erros de validação devolvidos pela API (HTTP 400) são exibidos no campo correspondente.

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão da tela: `VENDAS_MENU` para vendas e `CAIXA_MENU` para caixa e fiados. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 403 sem permissão, 404 não encontrado, 409 conflito (duplicidade ou regra de negócio violada).
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Criar as tabelas: `venda`, `item_venda`, `pagamento_caixa`, `fiado`. Todas têm `criado_em` e `atualizado_em`.
  - `venda`: `evento_id`, `desconto` (desconto dado na venda inteira, em valor), `preco_total`, `cancelada_em`.
  - `item_venda`: `venda_id`, `produto_evento_id`, `quantidade`, `preco_unitario` (snapshot de `produto_evento.preco_venda`) e `desconto_unitario` (snapshot de `produto_evento.desconto` vigente na hora da venda).
  - `pagamento_caixa`: `evento_id`, `tipo` (`VENDA_FICHA`, `DEVOLUCAO_FICHA` ou `QUITACAO_FIADO`), `valor`, `forma_pagamento`, `fiado_id` (só em `QUITACAO_FIADO`).
  - `fiado`: `pagamento_caixa_id` (lançamento da venda de fichas fiada), `nome_devedor`, `valor`, `observacoes`, `quitado`, `quitado_em` e `forma_pagamento` (forma usada na quitação; nula enquanto o fiado está pendente).
- A baixa do estoque usa a tabela `movimentacao_estoque` (tipo `VENDA`, com `item_venda_id`, `evento_id` e `estoque_id`, uma movimentação por lote consumido) e o estorno usa o tipo `ESTORNO_VENDA`, no mesmo lote de origem. Itens de origem `PRODUZIDO` (produzidos no evento) não geram movimentação.

### Backend

**Criar endpoint: `POST /eventos/{id}/vendas`**
- Corpo:
  ```json
  {
     "valorFinal":  number,
     "itens":  [
      {
         "produtoEventoId":  number,
         "quantidade":  number
      }
    ]
  }
  ```
- O evento deve existir, ser do tipo `EVENTO` e estar `EM_ANDAMENTO`. Caso contrário, 409.
- `itens`: ao menos um. Cada `produtoEventoId` deve pertencer ao evento, sem repetição. `quantidade` maior que 0.
- `valorFinal`: opcional. Se informado, deve ser maior ou igual a 0 e menor ou igual à soma dos itens (já com o desconto unitário vigente de cada produto). O `desconto` da venda é a soma dos itens menos o `valorFinal`, calculado e salvo pelo backend. Se ausente, o desconto da venda é 0.
- Não há desconto por item no corpo: o desconto de cada produto é o `desconto` vigente em `produto_evento`, aplicado antes da venda pelo endpoint de desconto.
- Na mesma transação:
  - cria a `venda` e os `item_venda`, gravando `preco_unitario` e `desconto_unitario` como snapshot de `produto_evento.preco_venda` e `produto_evento.desconto`;
  - calcula `preco_total` = soma de (`quantidade` × (`preco_unitario` − `desconto_unitario`)) − `desconto` da venda;
  - para cada item de origem `ESTOQUE`, consome os lotes do produto do mais antigo para o mais novo (FIFO), gerando uma movimentação de saída `VENDA` por lote consumido, com `item_venda_id`, `evento_id`, `estoque_id` e o `custo_unitario` do lote, reduzindo `estoque.quantidade_atual` e atualizando `produto.quantidade_atual`. Itens de origem `PRODUZIDO` não validam nem baixam estoque e não geram movimentação.
- A venda é rejeitada (409), sem ser criada, se os lotes de qualquer item de origem `ESTOQUE` não cobrirem a quantidade vendida. O estoque nunca pode ficar negativo. A quantidade vendida acima da prevista não bloqueia a venda se houver saldo real suficiente.
- Retorna 201 com a venda: `{ id, criadoEm, desconto, precoTotal }`.

**Criar endpoint: `PATCH /eventos/{id}/produtos/{produtoEventoId}/desconto`**
- Aplica desconto a um produto do cardápio. Corpo: 
  ```json
  {
     "novoPreco":  number
  }
  ```
  . Exige `VENDAS_MENU`.
- O evento deve estar `EM_ANDAMENTO`, senão 409. 404 se o item não for do evento. 409 se o produto já tiver desconto aplicado (para mudar o valor, remove-se o desconto antes).
- `novoPreco`: obrigatório, maior ou igual a 0 e menor ou igual a `preco_venda` (400 caso contrário).
- Grava `desconto` = `preco_venda` − `novoPreco` (por unidade) e `desconto_aplicado_em` = agora. Se `novoPreco` for igual a `preco_venda`, o desconto fica 0 e nada é gravado.
- Vendas já registradas não mudam; o desconto vale para as vendas feitas a partir daí.
- Retorna 200 com `{ id, precoVenda, desconto, descontoAplicadoEm, precoComDesconto }`.

**Criar endpoint: `DELETE /eventos/{id}/produtos/{produtoEventoId}/desconto`**
- Remove o desconto: zera `desconto` e `desconto_aplicado_em`. Exige `VENDAS_MENU`.
- O evento deve estar `EM_ANDAMENTO`, senão 409. 404 se o item não for do evento. Se não houver desconto aplicado, não altera nada.
- Retorna 200 com `{ id, precoVenda, desconto, descontoAplicadoEm, precoComDesconto }`.

**Criar endpoint: `POST /eventos/{id}/vendas/{vendaId}/cancelamento`**
- Sem corpo. 404 se a venda não for do evento. 409 se já estiver cancelada. 409 se o evento estiver `ENCERRADO`: não é possível cancelar venda de evento encerrado (reabrir o evento volta a permitir).
- Na mesma transação: preenche `venda.cancelada_em` e, para cada movimentação `VENDA` da venda, gera uma movimentação de entrada `ESTORNO_VENDA` com o mesmo `item_venda_id`, no mesmo lote (`estoque_id`) e com o mesmo custo, devolvendo a quantidade ao lote de origem (mesmo que esteja zerado) e atualizando `produto.quantidade_atual`. Itens de origem `PRODUZIDO` não têm estorno. Retorna 204.
- O cancelamento é definitivo: não existe reativação de venda. Para vender de novo, registra-se uma nova venda.
- A venda cancelada deixa de contar em receita, quantidade vendida, relatórios e dashboards.

**Criar endpoint: `POST /eventos/{id}/caixa`**
- Corpo: 
  ```json
  {
     "tipo":  "VENDA_FICHA" | "DEVOLUCAO_FICHA",
     "valor":  number,
     "formaPagamento":  string,
     "nomeDevedor":  string,
     "observacao":  string
  }
  ```

- O evento deve estar `EM_ANDAMENTO`, senão 409.
- `valor`: obrigatório, maior que 0.
- `formaPagamento`: obrigatório, `PIX`, `DEBITO`, `CREDITO`, `DINHEIRO` ou `FIADO`.
- `FIADO` só é aceito em `VENDA_FICHA`. Nesse caso `nomeDevedor` é obrigatório (máximo de 150 caracteres), `observacao` é opcional (máximo de 255) e, na mesma transação, cria o `fiado` ligado ao lançamento.
- `nomeDevedor` e `observacao` são ignorados nas outras formas de pagamento.
- Devolução de fichas: restitui 100% do valor das fichas não usadas, tipicamente em `DINHEIRO`.
- Retorna 201 com 
  ```json
  {
     lancamento:  {
       id,
       tipo,
       valor,
       formaPagamento,
       nomeDevedor,
       criadoEm
    },
     resumo
  }
  ```
  , para o frontend atualizar a lista e o resumo sem nova consulta.

**Criar endpoint: `GET /eventos/{id}/caixa`**
- Histórico paginado (padrão de paginação) dos lançamentos, do mais recente para o mais antigo. Parâmetros: `tipo`, `formaPagamento`.
- Retorna `{ resumo, lancamentos }`, em que `lancamentos` segue o envelope de paginação:
  - `resumo` = `{ fichasVendidas, fichasDevolvidas, saldoFichas, porFormaPagamento, fiadosEmAberto, quantidadeFiadosEmAberto, totalRecebido }`;
  - `porFormaPagamento` = vendas de ficha que não são fiado e quitações de fiado (na forma de pagamento da quitação), menos as devoluções (na forma da devolução);
  - `totalRecebido` = soma de `porFormaPagamento`;
  - `lancamentos` = `[{ id, tipo, valor, formaPagamento, nomeDevedor, criadoEm }]`. Traz **todas as vendas de ficha** (inclusive as fiadas), as devoluções e os **pagamentos dos fiados quitados** (`tipo = QUITACAO_FIADO`, com a forma de pagamento usada na quitação). `nomeDevedor` só vem nos lançamentos ligados a fiado.

**Criar endpoint: `DELETE /eventos/{id}/caixa/{lancamentoId}`**
- Corrige registros acidentais do caixa. Exclusão física. Exige `CAIXA_MENU`.
- 404 se o lançamento não for do evento. 409 se o evento não estiver `EM_ANDAMENTO`.
- `VENDA_FICHA` em `FIADO`: exclui o lançamento e, na mesma transação, o fiado criado por ele. Se o fiado já estiver quitado, retorna 409 (primeiro é preciso excluir a quitação).
- `QUITACAO_FIADO`: exclui o lançamento e reabre o fiado (volta a pendente e limpa `quitado_em` e `forma_pagamento`), na mesma transação.
- Demais lançamentos: exclui o lançamento.
- Retorna 200 com `{ resumo }`.

**Criar endpoint: `GET /eventos/{id}/fiados`**
- Parâmetro: `situacao` (`pendente` ou `quitado`). Se ausente, retorna todos os fiados. Outro valor retorna 400.
- Retorna `[{ id, nomeDevedor, valor, observacoes, quitado, quitadoEm, formaPagamento, criadoEm }]`.

**Criar endpoint: `PATCH /eventos/{id}/fiados/{fiadoId}/quitacao`**
- Corpo: 
  ```json
  {
     "formaPagamento":  "PIX" | "DEBITO" | "CREDITO" | "DINHEIRO"
  }
  ```
  . A forma de pagamento é obrigatória; `FIADO` ou valor ausente retornam 400.
- Marca `quitado = true`, preenche `quitado_em` e `forma_pagamento` e, na mesma transação, cria o lançamento `QUITACAO_FIADO` com o valor do fiado e a forma informada. 404 se não for do evento. 409 se já estiver quitado.
- O fiado pode ser quitado mesmo com o evento encerrado.
- Retorna 200 com `{ fiado, lancamento, resumo }`.

### Endpoints consumidos de outras histórias
- `GET /eventos?statusId=<EM_ANDAMENTO>&tipoEvento=EVENTO` (dona: gestão de eventos). Retorna `[{ id, nome, status, dataHoraInicio, dataHoraEncerramento }]`, com no máximo um evento (só pode haver um em andamento). Usado para descobrir o evento em andamento. O `statusId` de `EM_ANDAMENTO` é fixo, definido por migration.
- `GET /eventos/{id}/produtos` (dona: gestão de eventos). Parâmetros: `q`, `categoria` (nome da categoria guardado no snapshot; aceita mais de uma). Retorna `[{ id, origem, produtoId, nome, descricao, categoria, fotoUrl, precoVenda, desconto, descontoAplicadoEm, precoComDesconto, quantidadePrevista, quantidadeVendida, restantePrevisto, custoTotalIngredientes }]`. O `id` é o `produtoEventoId` usado em `POST /eventos/{id}/vendas`.
- `GET /eventos/{id}/produtos/categorias` (dona: gestão de eventos). Retorna a lista de nomes (textos) das categorias do snapshot do cardápio.
- Essas leituras exigem `EVENTOS_LEITURA`, `PRODUTOS_LEITURA` e `CARDAPIO_EVENTO_LEITURA`, sem conceder permissões de edição.

### Frontend

**Tela de "Registro de venda"** (padrão de tela)
- Ao abrir, busca o evento em andamento (`GET /eventos?statusId=<EM_ANDAMENTO>&tipoEvento=EVENTO`). Como só pode haver um, não há seletor de evento. Se não houver, mostra "Nenhum evento em andamento" e desabilita a tela.
- Campo de pesquisa e botão de pesquisar (padrão de pesquisa): chamam `GET /eventos/{id}/produtos`, com `q` e as categorias selecionadas (pelo nome, no parâmetro `categoria`).
- Lista de botões de categoria: chama `GET /eventos/{id}/produtos/categorias`. Se der erro, não renderiza a lista. Se der sucesso, renderiza numa lista horizontal um botão por nome de categoria.
  - Clicar em um botão não selecionado o seleciona e filtra os produtos pelas categorias selecionadas.
  - Clicar em um selecionado o deseleciona e refiltra.
  - Sem nenhuma categoria selecionada, busca todos os produtos do evento.
  - Botão "Limpar filtro": aparece só quando há pelo menos uma categoria selecionada. Ao clicar, desmarca todas as categorias e busca todos os produtos do evento, mantendo o texto da pesquisa.
- Lista de produtos (padrão de lista): chama `GET /eventos/{id}/produtos`. Vazio: "Não há produtos neste evento". Erro: "Houve um erro ao buscar os produtos".
- Card de produto: mostra foto, nome, preço em fichas e restante previsto (só em itens do estoque). Se o produto tiver desconto, mostra o preço de venda sem desconto riscado e, ao lado, o novo preço com desconto. O card não tem o botão "Ver detalhes". Botões:
  - "Adicionar à venda": adiciona à venda e troca o botão pelos botões de aumentar e reduzir quantidade, mostrando a quantidade atual.
  - Aumentar e reduzir: alteram a quantidade em 1 unidade. Ao reduzir para 0, volta o botão "Adicionar à venda".
  - "Aplicar desconto": aparece quando o produto não tem desconto. Abre o modal "Aplicar desconto".
  - "Remover desconto": substitui o "Aplicar desconto" quando o produto tem desconto. Pede confirmação e chama `DELETE /eventos/{id}/produtos/{produtoEventoId}/desconto`. Erro: "Houve um erro ao remover o desconto". Sucesso: "Desconto removido com sucesso", e o card volta a mostrar só o preço de venda.
- Modal "Aplicar desconto" (padrão de ação e de validação): mostra o nome e o preço de venda do produto e o campo "Novo preço" (obrigatório, maior ou igual a 0 e menor ou igual ao preço de venda). Mostra, antes de confirmar, o desconto por unidade calculado (preço de venda − novo preço). Botão "Aplicar desconto": chama `PATCH /eventos/{id}/produtos/{produtoEventoId}/desconto`. Erro: "Houve um erro ao aplicar o desconto". Sucesso: "Desconto aplicado com sucesso", fecha o modal e atualiza o card com a resposta. Botão "Cancelar": fecha o modal.
- Resumo da venda:
  - Lista de produtos adicionados, cada um com aumentar e reduzir (igual ao Card), preço unitário (com o desconto do produto, se houver) e o botão "Remover produto da venda", que zera a quantidade e remove da lista. Não há campo de desconto por item.
  - Campo "Valor final da venda": opcional, maior ou igual a 0 e menor ou igual ao total calculado. Fica vazio por padrão (equivale ao total). Ao informar, mostra o "Desconto da venda" calculado (total − valor final) antes de confirmar.
  - Total da venda calculado na tela, somando quantidade × preço com desconto de cada produto.
  - Botão "Confirmar venda" (padrão de ação): chama `POST /eventos/{id}/vendas`. Erro: toast "Houve um erro ao confirmar a venda". Sucesso: toast "Venda registrada com sucesso" e remove todos os itens e limpa o valor final.

**Tela de "Caixa"** (padrão de tela)
- Evento em andamento obtido como na tela de registro de venda, sem seletor de evento.
- Resumo no topo (`GET /eventos/{id}/caixa`): fichas vendidas, devolvidas, saldo, total recebido por forma de pagamento e fiados em aberto. Tem um botão para retrair e expandir (padrão expandido; o estado só existe no frontend). Retraído, o histórico e a barra lateral "Registrar" ocupam o espaço liberado.
- Barra lateral "Registrar" (padrão de ação e de validação):
  - Dois botões de seleção única para o tipo, "Venda de fichas" e "Devolução de fichas", no mesmo estilo dos botões de categoria da tela de registro de venda. Obrigatório.
  - "Valor": obrigatório, número positivo e acima de 0.
  - Botões de seleção única para a forma de pagamento (`PIX`, `Débito`, `Crédito`, `Dinheiro` e `Fiado`), no mesmo estilo. Obrigatório. "Fiado" só aparece para venda de fichas; se o tipo mudar para devolução com "Fiado" selecionado, a seleção é desfeita.
  - Se "Fiado" for selecionado, mostra "Nome" (não nulo ou em branco, máximo de 150) e "Observação" (opcional, máximo de 255).
  - Botão "Confirmar registro": chama `POST /eventos/{id}/caixa`. Erro: toast "Houve um erro ao registrar". Sucesso: toast "Registrado com sucesso", limpa todos os campos, atualiza o resumo com o `resumo` da resposta e insere o lançamento devolvido no topo da lista do histórico, sem nova consulta ao backend.
- Aba "Histórico de lançamentos" (padrão de pesquisa, de lista e de paginação): chama `GET /eventos/{id}/caixa`. Vazio: "Não há lançamentos neste evento". Erro: "Houve um erro ao buscar o caixa". Card de lançamento mostra tipo (inclusive "Quitação de fiado"), valor, forma de pagamento, nome (quando houver) e data e hora, com o botão "Excluir".
  - Atualização local ao registrar: o lançamento mais recente fica sempre no topo. Se o usuário estiver na primeira página e o lançamento casar com os filtros ativos, é inserido no topo; se a lista passar do número de itens por página (`size`), o último registro é retirado da lista local, e `totalItens` e `totalPaginas` são recalculados. Em outra página ou com filtro que não casa, a lista não muda.
  - Botão "Excluir": pede confirmação e chama `DELETE /eventos/{id}/caixa/{lancamentoId}`. Erro: modal de erro "Houve um erro ao excluir o lançamento", com o motivo retornado (ex.: evento encerrado, fiado já quitado). Sucesso: toast "Lançamento excluído com sucesso", remove o lançamento da lista local e atualiza o resumo com a resposta. Se o lançamento era ligado a um fiado, a aba "Fiados" é recarregada na próxima vez que for aberta.
- Aba "Fiados" (padrão de lista): o botão da aba mostra entre parênteses o número de fiados pendentes (`quantidadeFiadosEmAberto` do resumo). Chama `GET /eventos/{id}/fiados`, com filtro de botões "Todos", "Pendentes" e "Quitados" (parâmetro `situacao`). Vazio: "Não há fiados neste evento". Erro: "Houve um erro ao buscar os fiados". O Card de fiado mostra nome, valor, observação, situação e, se quitado, a forma de pagamento, com o botão "Quitar fiado" (só nos pendentes).
  - "Quitar fiado": abre o modal "Quitar fiado" (padrão de ação e de validação), que mostra o nome e o valor, e botões de seleção única para a forma de pagamento (`PIX`, `Débito`, `Crédito` e `Dinheiro`, obrigatório). Botão "Confirmar quitação": chama `PATCH /eventos/{id}/fiados/{fiadoId}/quitacao`. Erro: modal de erro "Houve um erro ao quitar o fiado". Sucesso: toast "Fiado quitado com sucesso", fecha o modal, atualiza o card (ou o remove da lista, se o filtro for "Pendentes"), atualiza o resumo e insere o lançamento de quitação no topo do histórico, tudo a partir da resposta. Botão "Cancelar": fecha o modal.
- Botão "Voltar" na aba de fiados: volta para o "Histórico de lançamentos".

## Regras
- Para a tela de registro de venda, o usuário deve ter `VENDAS_MENU`. Para a tela de caixa, `CAIXA_MENU`. Aplicar e remover desconto exigem `VENDAS_MENU`.
- Não se registra quem atendeu cada venda.
- O cancelamento de venda é acionado pelo modal de detalhes do histórico de vendas (tela da história de gestão de eventos), só com o evento não encerrado. Não existe reativação de venda cancelada.
- O desconto de produto é por unidade, definido no próprio produto do cardápio (preço novo menor ou igual ao preço de venda), guarda a hora em que foi aplicado e vale para as vendas feitas depois; vendas anteriores guardam o desconto vigente na época. Não há desconto por item no resumo da venda. O desconto da venda inteira é informado pelo valor final, e o desconto é calculado e salvo pelo backend.
- Produtos produzidos no evento (`PRODUZIDO`) podem ser vendidos como os demais, mas não validam nem baixam estoque e não geram movimentação.
- O caixa só aceita lançamentos com o evento `EM_ANDAMENTO`, exceto a quitação de fiado. A exclusão de lançamentos também só é permitida com o evento `EM_ANDAMENTO`.
- A quitação de fiado gera um lançamento `QUITACAO_FIADO` no caixa. Excluir esse lançamento reabre o fiado; excluir a venda de fichas fiada exclui também o fiado, desde que ele ainda esteja pendente.
- O frontend mantém o histórico do caixa, o resumo e a lista de fiados atualizados com as respostas das ações, sem consultas extras ao backend.
- As fichas físicas não são digitalizadas, só o impacto financeiro (venda, devolução e consumo em vendas).

## Protótipo
- Existe, exceto o desconto no card (aplicar e remover), o botão "Limpar filtro", o resumo retrátil do caixa, os botões de tipo e de forma de pagamento, o modal "Quitar fiado" e a exclusão de lançamentos, a definir.

## Premissas
- Existe um protótipo.
- As tabelas estão criadas no banco de dados.
- O cardápio do evento (`produto_evento`) já foi montado antes do evento começar, com preço e quantidade prevista (itens do estoque) ou custo dos ingredientes (itens produzidos no evento).

## Sugestão de subtarefas
- BD: migrations de `venda`, `item_venda`, `pagamento_caixa` (com `QUITACAO_FIADO`) e `fiado` (com `forma_pagamento`).
- BE: `POST /eventos/{id}/vendas` (cálculo de total, valor final e baixa de estoque por lote, FIFO).
- BE: `PATCH` e `DELETE /eventos/{id}/produtos/{produtoEventoId}/desconto`.
- BE: `POST /eventos/{id}/vendas/{vendaId}/cancelamento` (estorno no lote de origem e bloqueio com evento encerrado).
- BE: `POST /eventos/{id}/caixa` (com criação do fiado) e `DELETE /eventos/{id}/caixa/{lancamentoId}`.
- BE: `GET /eventos/{id}/caixa` (resumo e lançamentos paginados, com quitações).
- BE: `GET /eventos/{id}/fiados` (filtro `situacao`) e `PATCH .../quitacao` (com forma de pagamento).
- FE: tela "Registro de venda" (busca, categorias com "Limpar filtro", cards com desconto).
- FE: modal "Aplicar desconto" e ação "Remover desconto".
- FE: resumo da venda (carrinho, valor final, confirmar).
- FE: tela "Caixa" (resumo retrátil e barra "Registrar" com botões).
- FE: aba "Histórico de lançamentos" (paginação, atualização local e exclusão).
- FE: aba "Fiados" com filtros e modal "Quitar fiado".

---

# H6 — Relatórios financeiros

## Objetivo

**Eu**, enquanto administrador  
**Quero** consultar um relatório individual por evento, com produtos vendidos, custos, doações e o lucro calculado automaticamente  
**Para** visualizar rapidamente a performance do evento e possíveis melhorias.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de pesquisa e filtros**
- Campo de pesquisa e botão de pesquisar chamam o endpoint de listagem da tela, passando o texto no parâmetro `q` e os filtros selecionados.
- Botão "Filtrar" com o painel fechado: muda o ícone e mostra abaixo as opções de filtro.
- Botão "Filtrar" com o painel aberto: muda o ícone e esconde as opções, **sem limpar** os filtros já aplicados.

**Padrão de lista**
- Enquanto carrega, mostra ícone de carregamento.
- Se não houver registros, mostra a mensagem de lista vazia indicada na tela.
- Em caso de erro, mostra um toast de erro e renderiza na lista a mensagem de erro indicada na tela.
- Em caso de sucesso, renderiza um Card para cada registro retornado.

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão `RELATORIOS_MENU`. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação, 401 não autenticado, 403 sem permissão, 404 não encontrado, 409 conflito (regra de negócio violada).
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Não cria tabelas. Os dados são calculados a partir de `evento`, `venda`, `item_venda`, `produto_evento`, `custo_evento`, `pagamento_caixa`, `fiado`, `doacao` e `movimentacao_estoque`.
- Campos usados: `venda.preco_total`, `venda.desconto` e `venda.cancelada_em`; `item_venda.quantidade`, `preco_unitario` e `desconto_unitario`; `produto_evento.origem`, `quantidade_prevista` e `custo_total_ingredientes`; `custo_evento.valor`; `movimentacao_estoque.quantidade`, `custo_unitario`, `evento_id`, `item_venda_id` e o tipo da movimentação; `doacao.valor_ficha`, `doacao.valor` e `doacao.tipo`.

### Backend

**Criar endpoint: `GET /relatorios/eventos`**
- Parâmetros: `q`, `tipoEvento`, `dataInicio`, `dataFim`. Por padrão, só eventos `ENCERRADO`.
- Retorna `[{ id, nome, tipoEvento, data, receita, custo, lucro }]`.

**Criar endpoint: `GET /eventos/{id}/relatorio`**
- Evento `ENCERRADO`: relatório final. Evento `EM_ANDAMENTO`: relatório parcial (`parcial: true`). Evento `NAO_INICIADO`: 409. 404 se não existir.
- Para evento do tipo `EVENTO`, retorna:
  - `resumo`:
    - `receitaVendas` = soma de `venda.preco_total` das vendas não canceladas (já considera os descontos de produto e o desconto da venda);
    - `descontosVendas` = soma de `venda.desconto` das vendas não canceladas (informativo: descontos dados na venda inteira, que não pertencem a nenhum produto);
    - `custoCompras` = soma de |`quantidade`| × `custo_unitario` das movimentações `REABASTECIMENTO` do evento;
    - `custoFichasAgradecimento` = soma de `doacao.valor_ficha` do evento;
    - `custoIngredientes` = soma de `produto_evento.custo_total_ingredientes` dos itens produzidos no evento;
    - `totalCustosAdicionais` = soma de `custo_evento.valor` do evento;
    - `doacoesEmDinheiro` = soma das doações do tipo `dinheiro` do evento;
    - `receitaTotal` = `receitaVendas` + `doacoesEmDinheiro`;
    - `custoTotal` = `custoCompras` + `custoFichasAgradecimento` + `custoIngredientes` + `totalCustosAdicionais`;
    - `lucro` = `receitaTotal` − `custoTotal`;
  - `produtos`: um item para cada produto do cardápio do evento (inclusive os sem venda), só com vendas não canceladas: `[{ produtoEventoId, nome, origem, quantidadePrevista, quantidadeVendida, totalDescontos, custoTotal, lucroBruto, lucroLiquido }]`:
    - `quantidadePrevista`: nula nos itens produzidos no evento;
    - `quantidadeVendida` = soma de `item_venda.quantidade`;
    - `totalDescontos` = soma de `quantidade` × `desconto_unitario` (descontos aplicados no próprio produto);
    - `custoTotal`: nos itens do estoque, soma de |`quantidade`| × `custo_unitario` das movimentações `VENDA` do produto (custo dos lotes consumidos; lote sem custo conta 0); nos itens produzidos, `custo_total_ingredientes`;
    - `lucroBruto` = total arrecadado do produto, sem descontar custos = soma de `quantidade` × (`preco_unitario` − `desconto_unitario`);
    - `lucroLiquido` = `lucroBruto` − `custoTotal`.
    - A soma do `lucroBruto` de todos os produtos menos `descontosVendas` é igual a `receitaVendas`;
  - `custosAdicionais`: `[{ nome, descricao, valor }]`;
  - `caixa`: fichas vendidas, devolvidas, total por forma de pagamento (inclui as quitações de fiado) e fiados em aberto;
  - `doacoes`: itens doados (produto e quantidade) e doações em dinheiro;
  - `movimentacoesEvento`: movimentações de consumo e perdas vinculadas ao evento, para consultar as quantidades registradas. Não existe registro separado de sobras.
- Para evento do tipo `REUNIAO`, retorna `custoTotal` = compras (`REABASTECIMENTO`) + consumo interno (`CONSUMO_INTERNO`, |`quantidade`| × `custo_unitario`) + `totalCustosAdicionais`, com a lista de itens e a lista `custosAdicionais`. Movimentações sem custo informado (ex.: lote de doação) não somam custo. Não há receita nem lucro.

### Frontend

**Tela de "Relatórios"** (padrão de tela)
- Chama `GET /relatorios/eventos` (padrão de pesquisa e de lista). Filtros: tipo do evento e período. Vazio: "Não há eventos encerrados". Erro: "Houve um erro ao buscar os relatórios".
- Card de evento: nome, data, receita, custo e lucro, com o botão "Ver relatório".

**Tela de "Relatório do evento"** (padrão de tela)
- Chama `GET /eventos/{id}/relatorio`. Carregando: ícone. Erro: toast e a mensagem "Houve um erro ao buscar o relatório do evento".
- Mostra, em seções: Resumo (cards de receita, custos e lucro, com a linha "Descontos da venda"), Produtos, Custos adicionais, Caixa, Doações e movimentações de consumo/perda do evento.
- Seção "Produtos": para cada produto, mostra custo total, lucro bruto, lucro líquido, quantidade prevista (ou "—" nos produzidos no evento), quantidade vendida e valor total dos descontos. Produtos sem venda também aparecem.
- Seção "Custos adicionais": lista nome, descrição e valor de cada custo e o total.
- Relatório parcial mostra o aviso "Evento em andamento: valores parciais".
- Para reunião, mostra só a seção de custos (compras, consumo interno e custos adicionais).
- Botão de voltar: redireciona para "Relatórios".

## Regras
- O usuário deve ter a permissão `RELATORIOS_MENU` para acessar as telas e usar as funcionalidades da história.
- Venda cancelada não conta em nenhum valor do relatório.
- O custo total do produto é o das unidades vendidas (custo dos lotes consumidos), e não as compras feitas para o evento; por isso a soma dos custos dos produtos pode diferir de `custoCompras`.

## Protótipo
- A definir.

## Premissas
- As vendas, doações, custos adicionais e movimentações relevantes de consumo e perda devem estar registrados para o relatório final refletir o evento.
- As tabelas usadas como fonte já existem (criadas nas histórias de eventos, estoque, doações e vendas e caixa).

## Sugestão de subtarefas
- BE: `GET /relatorios/eventos`.
- BE: `GET /eventos/{id}/relatorio` para evento do tipo `EVENTO` (resumo, produtos, custos adicionais, caixa e doações).
- BE: `GET /eventos/{id}/relatorio` para evento do tipo `REUNIAO`.
- FE: tela "Relatórios" (lista, filtros e card).
- FE: tela "Relatório do evento" (seções, tabela de produtos, custos adicionais e relatório parcial).

---

# H7 — Dashboards gerenciais

## Objetivo

**Eu**, enquanto administrador  
**Quero** visualizar dashboards com indicadores financeiros, de vendas e de estoque  
**Para** acompanhar rapidamente o desempenho financeiro, de vendas e de estoque dos eventos.
## Padrões desta história

**Padrão de tela**
- Toda tela é responsiva (celular, tablet e computador) e está de acordo com o protótipo.

**Padrão de bloco de indicador**
- Cada bloco (card, gráfico ou lista) carrega de forma independente dos demais.
- Carregando: mostra ícone de carregamento só no bloco.
- Erro: mostra toast e a mensagem "Houve um erro ao buscar este indicador" só no bloco.
- Sem dados: mostra "Sem dados no período".

**Padrão de API**
- Todo endpoint exige JWT válido e a permissão `DASHBOARD_MENU`. Sem token, 401. Sem permissão, 403.
- Erros seguem o corpo 
  ```json
  {
     "codigo":  string,
     "mensagem":  string,
     "campos":  [
      {
         "campo":  string,
         "mensagem":  string
      }
    ]
  }
  ```

- Códigos HTTP: 400 validação (ex.: `dataFim` antes de `dataInicio`), 401 não autenticado, 403 sem permissão.
- Valores monetários e quantidades usam `BigDecimal` (nunca `Double`). Datas e horas trafegam em ISO-8601 (`Instant`).

## Detalhes técnicos

### Banco de dados
- Não cria tabelas. Os indicadores são consultas sobre `evento`, `venda`, `item_venda`, `produto_evento`, `custo_evento`, `doacao`, `produto`, `estoque` e `movimentacao_estoque`.
- Campos usados: `venda.preco_total` e `venda.cancelada_em`; `item_venda.quantidade`, `preco_unitario` e `desconto_unitario`; `produto_evento.custo_total_ingredientes`; `custo_evento.valor`; `produto.quantidade_atual` e `estoque_minimo`; `estoque.quantidade_atual` e `custo_unitario`; custos de compra e fichas de agradecimento, calculados como no relatório por evento.

### Backend

Todos os endpoints aceitam `dataInicio` e `dataFim` (padrão: últimos 6 meses) e consideram só eventos do tipo `EVENTO`. Vendas canceladas não são contadas.

**Criar endpoint: `GET /dashboard/resumo`**
- Retorna `{ totalEventos, receitaTotal, custoTotal, lucroTotal }`, com as fórmulas:
  - `receitaTotal` = soma de `venda.preco_total` das vendas não canceladas + soma de `doacao.valor` das doações em dinheiro;
  - `custoTotal` = compras (`REABASTECIMENTO` dos eventos, |`quantidade`| × `custo_unitario`) + soma de `doacao.valor_ficha` + soma de `produto_evento.custo_total_ingredientes` + soma de `custo_evento.valor` (custos adicionais);
  - `lucroTotal` = `receitaTotal` − `custoTotal`.

**Criar endpoint: `GET /dashboard/eventos`**
- Retorna a série `[{ eventoId, nome, data, receita, custo, lucro }]` por evento, em ordem cronológica. Permite comparar o evento atual com o anterior.

**Criar endpoint: `GET /dashboard/produtos-mais-vendidos`**
- Parâmetro: `limite` (padrão 10).
- Retorna `[{ produtoId, nome, quantidade, receita, participacao, participacaoAcumulada }]`, ordenado por receita (base da curva ABC). A `receita` do produto é o total arrecadado dele, já com o desconto unitário.

**Criar endpoint: `GET /dashboard/estoque`**
- Retorna 
  ```json
  {
     valorEstoque,
     produtosAbaixoDoMinimo:  [
      ...
    ],
     produtosNegativos:  [
      ...
    ]
  }
  ```

- `valorEstoque` = soma de `estoque.quantidade_atual` × `estoque.custo_unitario` dos lotes com custo informado.
- Este endpoint não usa o filtro de período (reflete o estoque atual).

### Frontend

**Tela de "Dashboard"** (padrão de tela e de bloco de indicador)
- Filtro de período. Mudar o período recarrega todos os blocos que usam período.
- Cards de resumo: eventos, receita, custo e lucro (`GET /dashboard/resumo`).
- Gráfico de barras: receita, custo e lucro por evento (`GET /dashboard/eventos`).
- Gráfico de barras horizontais: produtos mais vendidos (`GET /dashboard/produtos-mais-vendidos`).
- Lista de alertas de estoque: produtos abaixo do mínimo (`GET /dashboard/estoque`), com link para a tela de "Estoque" quando o usuário tiver `ESTOQUE_MENU`.
- Card "Valor em estoque" (`GET /dashboard/estoque`).

## Regras
- O usuário deve ter a permissão `DASHBOARD_MENU` para acessar a tela e usar as funcionalidades da história.
- As fórmulas de receita, custo e lucro são as mesmas do relatório por evento, incluindo as doações em dinheiro na receita do evento.

## Protótipo
- A definir.

## Premissas
- Os eventos têm vendas, doações, custos adicionais e movimentações de estoque registrados para os indicadores refletirem a realidade.

## Sugestão de subtarefas
- BE: `GET /dashboard/resumo`.
- BE: `GET /dashboard/eventos`.
- BE: `GET /dashboard/produtos-mais-vendidos`.
- BE: `GET /dashboard/estoque`.
- FE: tela "Dashboard" com filtro de período e cards de resumo.
- FE: gráfico de receita, custo e lucro por evento.
- FE: gráfico de produtos mais vendidos.
- FE: alertas de estoque e card "Valor em estoque".

---

# Fora do escopo e pendências

## Fora do escopo por decisão do time
- Gestão de associados e anuidades (cadastro, titular e dependente, cobrança de inadimplentes).
- Controle de público de fora do evento (pagantes e visitantes).
- Custos mensais do baiten (contas de luz, água, gás e similares): não entram no custo do evento.

## Não cobertos por nenhuma história (decidir)
- Gestão de usuários, cargos e permissões pela interface (hoje por migration).

## Validações confirmadas incorporadas nesta versão
- Convenções gerais: toda tabela tem `criado_em` e `atualizado_em`; o parâmetro de pesquisa por texto se chama `q`; listas paginadas usam o padrão de paginação (`page`, `size` e envelope com `itens`, `totalItens` e `totalPaginas`), com botões "Anterior" e "Próxima"; "pedido" foi renomeado para "venda" em tabelas, endpoints e telas; `unidade_medida` foi removida de `produto` e `produto_evento` (permanece só em `doacao`, no tipo "produto").
- H1: validade do token de recuperação de senha de exatamente 1 hora; primeira configuração de senha pelo link de recuperação; apenas dois cargos, Administrador (tudo) e Operador (vendas e caixa); permissões de leitura separadas das permissões de edição; `PEDIDOS_LEITURA` e `ESTOQUE_MOVIMENTACAO` não existem.
- H2: alteração de datas permitida em todos os status conforme validações de cadastro; desfazer início permitido quando não houver vendas nem lançamentos de caixa, independentemente de doações ou movimentações de estoque; remoção do fluxo de registro de sobras; cardápio não pode ser criado/editado durante `EM_ANDAMENTO`; cancelamento de venda não permitido com o evento encerrado, e sem reativação.
- H2 (reabertura): feita pelo `PATCH /eventos/{id}` (`ENCERRADO` → `EM_ANDAMENTO`); qualquer evento `ENCERRADO` pode ser reaberto, mesmo com registros operacionais; dados operacionais anteriores preservados; sem histórico explícito de reaberturas; `iniciado_em` preservado; `encerrado_em` limpo ao reabrir e preenchido no próximo encerramento.
- H2 (um evento por vez): só pode haver um evento do tipo `EVENTO` em andamento, validado no backend e no banco; vale para iniciar e reabrir; reuniões ficam fora da regra; registro de venda e caixa não têm seletor de evento; a lista de eventos mostra o evento em andamento no topo, com destaque; `GET /eventos` exige `EVENTOS_LEITURA`.
- H2 (custos adicionais): tabela `custo_evento` (nome, descrição e valor), informada ao encerrar o evento e editável depois, para eventos e reuniões; contas de luz, água, gás e similares ficam fora (custo mensal).
- H2 (cardápio): item do estoque (com quantidade prevista) ou produzido no evento (com custo total dos ingredientes, sem quantidade, sem estoque e sem movimentação), este último só incluído antes do evento começar; `POST /eventos/{id}/produtos` devolve todos os campos do snapshot; `GET /eventos/{id}/vendas` filtra por `cancelada` (ausente = todas, `true` = canceladas, `false` = ativas).
- H3 (categorias): CRUD de categorias acessado pelo botão "Menu de categorias" na tela de Estoque, em um modal com o mesmo nome; nome obrigatório com máximo de 255 caracteres; descrição opcional; nome duplicado rejeitado sem diferenciar maiúsculas e minúsculas; exclusão bloqueada quando houver produtos vinculados; pesquisa feita no frontend; criar, editar e excluir só salvam no backend e atualizam a lista local com a resposta.
- H3: estoque nunca negativo; saídas são rejeitadas quando não houver saldo suficiente; exclusão definitiva somente sem dependências históricas/operacionais; produtos com histórico são desativados, preservando referências; `GET /produtos` filtra por mais de uma categoria.
- H3 (escopo do estoque): todos os produtos são alimentos ou bebidas prontos para venda; ingredientes e itens não comestíveis ficam fora do estoque; todo produto pode ser movimentado (sem `controla_estoque`).
- H3 (lotes e FIFO): cada entrada cria um lote (`estoque`) com `quantidade_inicial`, `quantidade_atual`, `custo_unitario` e `custo_total` (valor pago pelo lote inteiro); saídas e vendas consomem os lotes do mais antigo para o mais novo; `produto.quantidade_atual` é a soma dos lotes; não há custo médio, o custo pertence ao lote; o modal de detalhes mostra as movimentações de todos os lotes e a lista de lotes com as movimentações de cada um; sem migração, pois não há dados existentes.
- H3 (movimentação): `PERDA` registra o motivo no campo opcional "Observações"; `AJUSTE_ENTRADA` e `AJUSTE_SAIDA` corrigem divergências de contagem; o modal mostra a quantidade após a movimentação antes de confirmar; movimentar estoque exige `ESTOQUE_MENU`; o tipo `PRODUCAO_EVENTO` foi removido.
- H4: doações editáveis em qualquer status do evento; `criadoEm` imutável; três tipos: dinheiro (inalterado), produto (sem regras de estoque, pode ser ingrediente, unidades `un`, `kg` ou `L`, sem movimentação) e reabastecimento (produto existente ou criado seguindo as regras de estoque, cria lote sem custo e movimentação `DOACAO`); em doação de dinheiro podem ser editados evento, doador, valor, forma de pagamento e valor da ficha; em doação de produto, evento, doador, descrição, quantidade, unidade e valor da ficha; em reabastecimento, evento, doador, quantidade e valor da ficha, sem reduzir abaixo do consumido do lote; tipo e produto imutáveis; a movimentação `DOACAO` e o lote são atualizados junto com a quantidade; a listagem não filtra por período, aceita vários tipos e eventos, é paginada e o card não mostra data.
- H5 (vendas): a venda gera movimentação `VENDA` por lote consumido e o cancelamento gera `ESTORNO_VENDA` no lote de origem; a reativação foi removida; produtos produzidos no evento não geram movimentação.
- H5 (descontos): o desconto de produto é por unidade, definido no próprio card (novo preço menor ou igual ao preço de venda) e removível por "Remover desconto"; guarda a hora de aplicação; o resumo da venda não tem desconto por item; o desconto da venda é informado pelo valor final, e o desconto é calculado e salvo pelo backend.
- H5 (caixa e fiados): o histórico do caixa é paginado e traz vendas de ficha, devoluções e quitações de fiado; `DELETE /eventos/{id}/caixa/{lancamentoId}` corrige registros acidentais (excluir venda fiada pendente exclui o fiado; excluir a quitação reabre o fiado); `fiado` tem `forma_pagamento`; a quitação exige a forma de pagamento e gera um lançamento `QUITACAO_FIADO`; `GET /eventos/{id}/fiados` filtra por `situacao` (pendente ou quitado; ausente = todos); o resumo do caixa é retrátil e o frontend atualiza lista e resumo com as respostas, sem consultas extras.
- H6 e H7: doações em dinheiro integram a receita e o cálculo do lucro do evento; o custo do evento inclui os custos adicionais e os ingredientes dos produtos produzidos no evento; o relatório mostra, por produto, custo total, lucro bruto, lucro líquido, quantidade prevista, quantidade vendida e total de descontos.
- Escopo financeiro: removida a história de fluxo financeiro geral; o sistema mantém apenas informações financeiras relacionadas aos eventos.
- Produto no cardápio: referência por ID mais snapshot dos campos históricos; alterações no cadastro geral não alteram o snapshot do evento; a categoria é guardada no snapshot pelo nome (texto), não pelo id.

## Pontos técnicos ainda a definir na implementação
- Os campos exatos do snapshot de `produto_evento`, além dos campos históricos explicitamente listados (recomendação: nenhum além de `nome`, `descricao`, `categoria` pelo nome e `foto_url`).
