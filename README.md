# Desafio técnico SIG 2000 — Importação de pedidos B2B com reserva de estoque

**Desenvolvedor(a) Delphi Pleno ou Sênior**

Uma distribuidora recebe vendas por um portal B2B. O portal usa seus próprios códigos para identificar clientes e produtos. O ERP mantém os cadastros internos e controla o estoque. Desenvolva uma aplicação **Delphi com interface gráfica e banco Firebird** que permita ao usuário importar uma venda, consultar o motivo de uma rejeição, cadastrar um cliente, ajustar o estoque e tentar novamente pelo histórico.

Imagine um pedido de quatro unidades de um produto com apenas três disponíveis. O usuário vê a pendência, registra a entrada de duas unidades e reprocessa a venda pelo histórico. O resultado deve ser um pedido confirmado, com cinco unidades físicas, quatro reservadas e uma disponível.

## Como começar e entregar

1. Crie um repositório público na sua conta do GitHub e copie para ele os arquivos deste projeto-base. Desenvolva e entregue sua solução nesse repositório.
2. Em um banco Firebird dedicado ao desafio, na versão de sua escolha, execute o [script de estrutura fornecido](sql/01_estrutura.sql) usando o dialeto SQL 3.
3. No mesmo banco, execute uma vez a [carga inicial fornecida pela SIG 2000](sql/02_carga_inicial.sql). Ela inclui os 20 clientes e os 61 produtos presentes nos pedidos externos 1 a 20. O pedido 2 permite testar uma importação direta com saldo suficiente. O pedido 1 exercita a reposição do produto 501, também solicitado nos pedidos 8 e 12.
4. Inclua o projeto Delphi e o código-fonte no seu repositório. Documente como compilar a aplicação, configurar a conexão com o Firebird e reproduzir os testes. Informe as versões do Delphi e do Firebird utilizadas e as dependências do projeto.
5. Envie à SIG 2000 o link do seu repositório. Explique brevemente como a solução protege o saldo sob concorrência, impede duplicação de pedidos e preserva o histórico após um rollback.

## Integração com o portal

Cada carrinho da DummyJSON representa um pedido fechado. Importe um pedido por vez, pelo número inteiro positivo informado pelo usuário. Use a URL base `https://dummyjson.com`.

A DummyJSON já fornece contas fictícias para o desafio. Use a conta de demonstração abaixo ou consulte os campos `username` e `password` de uma mesma conta na [lista pública de usuários](https://dummyjson.com/users). [Documentação oficial do login](https://dummyjson.com/docs/auth)

Para iniciar a sessão, envie o seguinte JSON por `POST` para `https://dummyjson.com/auth/login` com o cabeçalho `Content-Type: application/json`.

```json
{
  "username": "emilys",
  "password": "emilyspass"
}
```

| Ação | Método e rota | Contrato |
|---|---|---|
| Iniciar sessão | `POST /auth/login` | Envie JSON com `username` e `password` de demonstração e utilize o `accessToken` e o `refreshToken` retornados. |
| Consultar venda | `GET /auth/carts/{id}` | Envie `Authorization: Bearer <accessToken>`. Para o pedido 1, use `GET /auth/carts/1`. |
| Renovar sessão | `POST /auth/refresh` | Se a busca retornar 401, envie JSON com `refreshToken`, atualize os tokens e repita a consulta uma vez. |

A renovação integra a mesma tentativa de importação. Se ela falhar ou a consulta repetida receber outro 401, registre falha de autenticação. Configure a senha localmente na aplicação e mantenha os tokens de sessão fora do repositório e do histórico. [Credenciais de demonstração e autenticação](https://dummyjson.com/docs/auth) · [Recursos autenticados](https://dummyjson.com/docs#auth) · [Carrinhos](https://dummyjson.com/docs/carts)

| Campo recebido | Destino e regra |
|---|---|
| `id` | Número externo da venda em `PEDIDO.COD_PORTAL`. |
| `userId` | Localize `CLIENTE.COD_PORTAL` e grave o ID interno em `PEDIDO.CLIENTE_ID`. |
| `products[].id` | Localize `PRODUTO.COD_PORTAL` e grave o ID interno em `PEDIDO_ITEM.PRODUTO_ID`. |
| `products[].quantity` | Quantidade da linha e quantidade a reservar. |
| `products[].price` | Preço unitário do item. |
| `products[].discountedTotal` | Valor líquido da linha inteira, incluindo quantidade e desconto. |

Numere os itens a partir de 1, na ordem recebida. Grave valores monetários com duas casas decimais. O total do pedido é a soma dos valores líquidos das linhas.

## Funcionalidades

### Importar e consultar um pedido

O usuário informa um número inteiro positivo e aciona **Importar pedido**. A aplicação segue estas etapas.

1. Registre a tentativa e verifique se o número externo já foi importado. Se já existir, informe `JA_IMPORTADO` e preserve o pedido e suas reservas.
2. Consulte o pedido no portal e localize no Firebird o cliente e os produtos pelos respectivos códigos externos.
3. Valide o saldo disponível de **todos os produtos**. Se o mesmo produto aparecer em várias linhas, some as quantidades solicitadas antes de conferir o saldo.
4. Se houver saldo para a venda inteira, grave o cabeçalho, os itens e as reservas **na mesma transação**. A confirmação aumenta a quantidade reservada. A quantidade física só muda quando o usuário registra uma entrada ou saída.
5. Apresente o resultado ao usuário. Para um pedido confirmado, permita consultar o número externo, o cliente, os produtos, as quantidades, os preços unitários, os valores líquidos dos itens e o total do pedido.

**Cliente ausente no Firebird.** Se o `userId` do pedido não existir em `CLIENTE.COD_PORTAL`, encerre a tentativa como `REJEITADO_CADASTRO`. Informe o número do pedido e o código externo do cliente. Preserve pedidos, itens e reservas. Nessa tentativa, ofereça a ação **Cadastrar cliente**.

**Produto ausente no Firebird.** Antes de validar o estoque, procure cada `products[].id` do pedido em `PRODUTO.COD_PORTAL`. Se algum código estiver ausente, rejeite **o pedido inteiro** com o status `REJEITADO_CADASTRO`. O cadastro de produtos é mantido no ERP. A importação deve deixar esse cadastro intacto, sem criar o produto automaticamente. Preserve os saldos e registre apenas a tentativa, sem gravar cabeçalho, itens ou reservas.

Mostre no histórico o número do pedido e o código externo de cada produto ausente. Exemplo de mensagem para o usuário do ERP: “Pedido 3 rejeitado. Produto do portal 9999 sem cadastro. Após vincular esse código a um produto no ERP e preparar seu estoque, use **Tentar novamente**.” Ao reprocessar, consulte novamente os cadastros e valide o saldo de todos os itens. Para demonstrar a correção cadastral, o avaliador pode usar SQL.

Em caso de falha na gravação, reverta toda a transação da venda e registre o resultado no histórico da tentativa.

### Consultar e cadastrar clientes

Na área **Clientes**, o usuário deve conseguir consultar os clientes cadastrados e iniciar um cadastro. Exiba o nome, o código interno do ERP e o código do portal para que ele possa conferir o vínculo.

1. Ao abrir **Cadastrar cliente** pelo histórico de uma rejeição, preencha o código do portal com o `userId` recebido. Ao iniciar o cadastro pela área **Clientes**, permita que o usuário informe esse código.
2. Solicite o nome do cliente (obrigatório, até 100 caracteres) e um código de portal inteiro positivo. Ao salvar, atribua um ID interno, grave o registro em `CLIENTE` e apresente-o na consulta. Se o código do portal já estiver em uso, informe a duplicidade e preserve o cadastro existente.
3. Depois de salvar, o usuário deve poder selecionar a rejeição original no histórico e acionar **Tentar novamente**. Localize o cliente recém-cadastrado pelo código do portal e prossiga com as validações da venda. Se o pedido for confirmado, grave o ID interno em `PEDIDO.CLIENTE_ID` e vincule a nova tentativa à rejeição original.

### Consultar e ajustar estoque

Permita localizar o produto pelo código interno, código do portal ou descrição. Exiba estoque físico, reservado e disponível. **Disponível = físico − reservado.** Confirmar uma venda aumenta a reserva e preserva o saldo físico.

Para uma entrada ou saída, exija quantidade inteira positiva e motivo. A entrada aumenta o físico. A saída só pode ocorrer quando o disponível no momento da gravação cobre toda a quantidade. Em caso de recusa, informe a quantidade solicitada e o saldo disponível.

Grave o ajuste e seu movimento na mesma transação. O histórico de movimentos deve mostrar produto, data e hora, tipo, quantidade, motivo e saldos físicos anterior e posterior.

### Histórico e reprocessamento

Cada ação **Importar pedido** ou **Tentar novamente** cria uma tentativa persistida com número externo, início e status `EM_ANDAMENTO`. Ao finalizar, registre término, resultado e mensagem compreensível ao usuário.

| Resultado | Quando usar e o que informar |
|---|---|
| `IMPORTADO` | Venda e reservas confirmadas. Identifique a venda. |
| `JA_IMPORTADO` | O número externo já está gravado. Identifique o pedido existente. |
| `NAO_ENCONTRADO` | A consulta retornou 404. Informe o número consultado. |
| `FALHA_AUTENTICACAO` | O login ou a recuperação da sessão falhou. Informe a etapa. |
| `FALHA_API` | Timeout, indisponibilidade ou resposta inválida. Informe a causa disponível. |
| `REJEITADO_CADASTRO` | Cliente ou produto sem correspondência. Identifique o tipo e o código externo. |
| `ESTOQUE_INSUFICIENTE` | Identifique o produto, a quantidade solicitada e a disponível. |
| `FALHA_GRAVACAO` | Houve erro no banco. Identifique a operação que falhou. |

No histórico, o usuário seleciona uma tentativa encerrada com falha ou rejeição e aciona **Tentar novamente**. Crie uma nova tentativa para o mesmo número externo, vinculada à original, e execute novamente o fluxo. Preserve a mensagem e o resultado anteriores, inclusive quando a transação da venda for revertida.

## Estrutura fornecida

| Tabela | Campos | Regras principais |
|---|---|---|
| CLIENTE | ID, COD_PORTAL, NOME | ID interno e código externo único. |
| PRODUTO | ID, COD_PORTAL, DESCRICAO | ID interno e código externo único. |
| ESTOQUE | PRODUTO_ID, QTD_FISICA, QTD_RESERVADA | Uma linha por produto. Reserva entre zero e o saldo físico. |
| PEDIDO | ID, COD_PORTAL, CLIENTE_ID, DATA_IMPORTACAO, TOTAL_LIQUIDO | Uma venda por código externo, vinculada ao cliente interno. |
| PEDIDO_ITEM | PEDIDO_ID, SEQUENCIA, PRODUTO_ID, QUANTIDADE, PRECO_UNITARIO, TOTAL_LIQUIDO | Chave composta pelo pedido e sequência. |
| IMPORTACAO_TENTATIVA | ID, COD_PEDIDO_PORTAL, INICIO, FIM, STATUS, MENSAGEM, TENTATIVA_ORIGEM_ID | Várias tentativas por venda. Origem preenchida ao reprocessar. |
| MOVIMENTO_ESTOQUE | ID, PRODUTO_ID, DATA_HORA, TIPO, QUANTIDADE, MOTIVO, FISICO_ANTERIOR, FISICO_POSTERIOR | Uma linha por entrada ou saída efetivada. |

Os tipos, as chaves e as restrições estão no [script SQL](sql/01_estrutura.sql). Os geradores `GEN_PEDIDO`, `GEN_TENTATIVA` e `GEN_MOVIMENTO` estão disponíveis para obter os respectivos IDs com `GEN_ID(nome_do_gerador, 1)`.

## Critérios de aceite

### Demonstração pela interface

Inicie o roteiro com a estrutura e a carga inicial fornecidas pela SIG 2000, conforme os passos 2 e 3. O pedido de referência é o **pedido 1** da DummyJSON. O cliente externo 1 corresponde ao cliente interno 1001. A carga inicial deixa pedidos, itens, movimentos e tentativas vazios.

| Produto interno | Código no portal | Produto | Quantidade no pedido 1 | Estoque físico inicial | Reserva inicial |
|---|---|---|---:|---:|---:|
| 501 | 162 | Vestido azul | 4 | 3 | 0 |
| 502 | 113 | Motocicleta | 3 | 10 | 0 |
| 503 | 122 | Smartphone iPhone 6 | 3 | 10 | 0 |
| 504 | 138 | Bola de beisebol | 2 | 10 | 0 |

Execute CA01 a CA07 em sequência. No CA01, a única pendência esperada é o produto 501. No CA02, registre a entrada de duas unidades desse produto pela interface.

| ID | Ação | Resultado esperado |
|---|---|---|
| CA01 | Importar o pedido pela API autenticada. | ESTOQUE_INSUFICIENTE identifica produto, quantidade solicitada e disponível. Zero pedidos, itens e reservas. |
| CA02 | Registrar entrada de duas unidades do produto pendente, com motivo. | Físico e disponível aumentam em duas unidades. Movimento registra motivo e saldos anterior e posterior. |
| CA03 | Acionar Tentar novamente na tentativa do CA01. | Nova tentativa vinculada termina como IMPORTADO. Todos os itens são reservados. O produto ajustado fica com uma unidade disponível. |
| CA04 | Consultar o pedido. | Cliente, produtos, quantidades e valores conferem com a resposta recebida. O total corresponde à soma dos valores líquidos dos itens. |
| CA05 | Importar o mesmo número novamente. | JA_IMPORTADO. Continua existindo uma venda, com os mesmos itens e reservas. |
| CA06 | No produto ajustado, tentar saída de duas unidades e depois registrar saída de uma. | Primeira saída recusada, segunda efetivada. Disponível zero, reservas preservadas e apenas uma saída registrada. |
| CA07 | Fechar e reabrir a aplicação. | Pedido, saldos, dois movimentos e três tentativas continuam consultáveis. |

### Falhas e concorrência

Prepare cada cenário de forma independente. Respostas HTTP simuladas podem ser usadas nos testes, exercitando a mesma rotina de importação. Documente como o avaliador pode reproduzir os resultados.

| ID | Condição | Resultado esperado |
|---|---|---|
| CA08 | A busca recebe 401. Testar renovação bem-sucedida e renovação recusada. | No primeiro caso, repetir a busca uma vez e concluir a mesma tentativa. No segundo, registrar FALHA_AUTENTICACAO e preservar pedidos e saldos. |
| CA09 | Uma consulta retorna 404 e outra retorna 503. | NAO_ENCONTRADO e FALHA_API, respectivamente, com identificação do pedido. |
| CA10a | Consultar um pedido novo de cliente cadastrado com dois itens, um produto conhecido com saldo suficiente e o produto externo 9999 ausente. | O histórico registra REJEITADO_CADASTRO, o número do pedido e o código 9999. O produto continua ausente. Nenhum cabeçalho, item ou reserva é gravado, inclusive para o produto conhecido. |
| CA10b | Cadastrar o produto com COD_PORTAL = 9999, incluir sua linha de estoque com saldo suficiente e acionar Tentar novamente na tentativa do CA10a. | Uma nova tentativa vinculada termina como IMPORTADO. O pedido contém os dois itens, com as respectivas reservas. A rejeição original permanece no histórico. |
| CA10c | Em uma base com a carga inicial, simular um pedido novo com `userId = 9999` e uma unidade do produto externo 113. | A tentativa termina como REJEITADO_CADASTRO, informa o pedido e o cliente externo 9999 e oferece **Cadastrar cliente**. Pedidos, itens e saldos são preservados. |
| CA10d | Na rejeição do CA10c, abrir **Cadastrar cliente**, conferir o código 9999 já preenchido, informar o nome “Distribuidora de Teste” e salvar. | O cliente aparece na consulta com um ID interno, código do portal 9999 e o nome informado. A tentativa original continua rejeitada. |
| CA10e | Pela área **Clientes**, tentar cadastrar outro nome com o código do portal 9999. | A interface informa que o código já está em uso. Existe um único cliente com esse código e seu nome continua “Distribuidora de Teste”. |
| CA10f | Selecionar a rejeição do CA10c e acionar **Tentar novamente**, usando a mesma resposta simulada. | A nova tentativa termina como IMPORTADO e referencia a original. O pedido aponta para o ID interno do cliente criado no CA10d. Uma unidade do produto interno 502 fica reservada. |
| CA11 | Falha de banco ao inserir o segundo item de um pedido novo. | Rollback do cabeçalho, dos itens e das reservas. A tentativa permanece consultável como FALHA_GRAVACAO. |
| CA12 | Duas instâncias importam vendas diferentes, cada uma solicitando quatro unidades do mesmo produto, com cinco disponíveis. | Uma venda é confirmada e a outra termina como ESTOQUE_INSUFICIENTE. Quatro unidades reservadas e uma disponível. |
| CA13 | Duas instâncias importam o mesmo número ao mesmo tempo, com saldo suficiente para duas cópias da venda. | Uma tentativa IMPORTADO e outra JA_IMPORTADO. Uma única venda e uma única reserva das quantidades solicitadas. |
| CA14 | O usuário abre a tela de saída com cinco disponíveis. Outra instância reserva quatro antes da confirmação de uma saída de duas. | A aplicação considera o disponível atual de uma unidade, recusa a saída e preserva a reserva. |

## Avaliação

| Critério | Peso |
|---|---:|
| Integridade dos pedidos e do estoque no Firebird, inclusive sob concorrência | 30% |
| Importação, ajuste de estoque e reprocessamento | 25% |
| Organização, legibilidade e manutenção do código Delphi | 20% |
| Integração HTTP, autenticação e tratamento de falhas | 15% |
| Reprodutibilidade da entrega e clareza das mensagens ao usuário | 10% |
