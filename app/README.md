# app/ — Projeto 03 (reaproveitamento do Projeto 01)

> **Esta pasta é intencionalmente vazia de código.** Não há duplicação de código-fonte aqui.

## Fonte de verdade do código

O código-fonte da aplicação **AWS Database Lab Store** vive em um único lugar:

```text
aws-database-evolution-labs/01-rds-resilience/app/
```

O Projeto 03 (DynamoDB) **reutiliza a mesma aplicação** do Projeto 01 (e do Projeto 02). Isso atende ao Requisito 13.7 (reaproveitar o máximo, sem recriar aplicação, layout ou componentes) e ao Requisito 9.1 (o projeto reaproveita a aplicação e é acessível via `dynamodb-lab.inhesta.net`).

## Por que não copiamos o código?

Copiar `app/` para dentro deste diretório criaria cópias do mesmo código, que precisariam ser mantidas em sincronia. Assim como no Projeto 02, mantemos **uma única cópia** em `01-rds-resilience/app/` e documentamos apenas o **delta** do Projeto 03.

## O que muda no Projeto 03 (o delta)

Diferente do Projeto 02 (onde a mudança foi **só de configuração**), o Projeto 03 tem **um delta de código pequeno e localizado**: o **backend do carrinho** passa de relacional para DynamoDB. O resto da aplicação (produtos, clientes, pedidos, imagens, layout) permanece **exatamente igual**.

### 1) Configuração (`.env` na EC2)

| Variável | Projeto 02 (Aurora) | Projeto 03 (DynamoDB) |
|---|---|---|
| `CART_MODE` | `relational` | **`dynamodb`** |
| `DYNAMODB_TABLE` | — | **`carrinho-lab`** (nome da tabela do carrinho) |
| `AWS_REGION` / `DYNAMODB_REGION` | — | região da tabela (ex.: `sa-east-1`) |
| `DB_WRITER_ENDPOINT` / `DB_READER_ENDPOINT` | endpoints do relacional | **mantêm** o relacional para produtos/clientes/pedidos (ver orientação de custo abaixo) |
| `LAB_PROJECT` | `Projeto 02 — AWS Aurora Resilience Lab` | `Projeto 03 — AWS DynamoDB Recovery Lab` |
| `LAB_ARCH_VERSION` | `Projeto 02 — Aurora MySQL (Cluster + Reader)` | `Projeto 03 — Carrinho no DynamoDB (carrinho-lab)` |

As variáveis de S3 (`STORAGE_MODE=s3`, `S3_BUCKET`, `S3_REGION`) permanecem iguais — as imagens continuam no mesmo bucket.

### 2) Código — implementação do backend DynamoDB em `cart.py`

O `cart.py` (em `01-rds-resilience/app/cart.py`) **já foi projetado** para trocar de backend sem tocar nas rotas Flask nem nos templates. A interface pública é estável:

- `add_item(cliente_id, produto_id, quantidade)`
- `get_cart(cliente_id)`
- `remove_item(cliente_id, produto_id)`
- `clear_cart(cliente_id)`

O modo `CART_MODE=dynamodb` está **implementado** (Tarefa 37 concluída). Os quatro métodos acima leem/gravam na tabela `carrinho-lab` via `boto3`, mantendo a mesma assinatura do backend relacional. Detalhes da implementação (em `01-rds-resilience/app/cart.py`):

- O recurso DynamoDB é criado de forma **preguiçosa (lazy)** em `_tabela()`, de modo que apenas importar o módulo **não exige credenciais AWS** (importante para o modo relacional e para testes locais).
- `add_item` incrementa a `quantidade` via `UpdateItem` (`ADD`) e **desnormaliza** `nome_produto` e `preco` a partir do banco relacional; grava `atualizado_em` em ISO-8601 (UTC).
- `get_cart` faz `Query` pela PK `cliente_id` e converte os `Decimal` do DynamoDB para `int`/`float` esperados pelos templates.
- `remove_item` usa `DeleteItem` pela chave composta; `clear_cart` apaga todos os itens da partição com `batch_writer`.
- Como `cliente_id`/`produto_id` são `String` no DynamoDB, os ids são convertidos para `str` na gravação e de volta para `int` na leitura.

> O atributo `imagem` **não** é gravado no item do carrinho: o template `templates/carrinho.html` exibe apenas `nome_produto`, `quantidade`, `preco` e o subtotal. Se um dia for necessário, pode ser buscado no relacional em `get_cart`.

> **Importante:** a implementação de `cart.py` (backend DynamoDB) continua sendo feita **na fonte de verdade** `01-rds-resilience/app/cart.py`, não com uma cópia dentro desta pasta. Assim, todos os projetos continuam apontando para o mesmo arquivo.

## Como executar o Projeto 03

1. Reutilize o mesmo código de `01-rds-resilience/app/` na EC2 (a mesma pasta `/opt/aws-database-lab/` já implantada).
2. Crie a tabela `carrinho-lab` no DynamoDB (Tarefa 36). O backend DynamoDB em `cart.py` já está implementado (Tarefa 37).
3. Ajuste o `.env` na EC2 (`CART_MODE=dynamodb`, `DYNAMODB_TABLE=carrinho-lab`, região) e mantenha o relacional para produtos/clientes/pedidos.
4. Reinicie a aplicação.

Nenhum `template/` ou `static/` precisa ser alterado ou copiado — o delta de código do Projeto 03 fica contido no backend do carrinho em `cart.py`.
