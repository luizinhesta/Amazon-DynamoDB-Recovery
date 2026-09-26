# NoSQL Gerenciado e Recuperação no Tempo: Carrinho no DynamoDB

> Documento de informação do **AWS DynamoDB Recovery Lab**. Conta a história da migração do **carrinho** da loja **AWS Database Lab Store** do banco relacional para o **Amazon DynamoDB** (NoSQL gerenciado), a habilitação do **Point-in-Time Recovery (PITR)** e o que se aprende para a certificação **AWS Certified Solutions Architect – Associate (SAA-C03)**. Os números marcados como *(medir)* devem ser substituídos pelos valores reais coletados na execução (evidências em `evidencias/`).
>
> **DNS:** `dynamodb-lab.inhesta.net`
> **Demais documentos:** `ARQUITETURA.md` (diagramas ANTES/DEPOIS), `IMPLANTACAO.md` (passo a passo detalhado no Console), `IMPLANTACAO-RESUMO.md` (guia enxuto de execução), `TESTES.md` (matriz de testes com resultados) e `EXCLUSAO.md` (limpeza/custo ao encerrar).

## Problema — por que NoSQL para o carrinho

Nos Projetos 01 e 02, a loja evoluiu de um monolito em EC2 para uma arquitetura desacoplada: banco relacional gerenciado (RDS → Aurora), alta disponibilidade, escala de leitura e imagens no S3. Nesse desenho, **tudo** — produtos, clientes, pedidos e também o **carrinho** — vivia no mesmo banco relacional, nas tabelas `carrinho` e `itens_carrinho`.

O carrinho, porém, tem um perfil de dados **diferente** do restante da loja. Ele é:

- **Efêmero:** o item fica no carrinho por minutos ou horas; se some, ninguém perde histórico contábil como perderia num pedido.
- **De alto volume de escrita:** cada clique de "adicionar", "alterar quantidade" ou "remover" é uma gravação. Numa loja movimentada, o carrinho grava muito mais que a tabela de pedidos.
- **De acesso por chave:** a operação natural é "traga o carrinho **deste** cliente" — uma leitura direta por chave, sem `JOIN` nem varredura.

Modelar isso no relacional **funciona**, mas força o motor transacional (ACID, `JOIN` entre `carrinho` e `itens_carrinho`, locks) a carregar um dado que não precisa dessas garantias fortes e que valoriza **latência baixa e escala** acima de tudo. É o descasamento clássico que motiva o uso de um **banco chave-valor/documento gerenciado**. Além disso, queríamos estudar **recuperação de dados granular** — voltar o carrinho para "pouco antes de alguém apagar" — de forma **gerenciada**, sem dump/restore manual.

## Arquitetura (ANTES → DEPOIS)

```text
ANTES (Projeto 02) — carrinho relacional
Usuário → Route 53 (aurora-lab.inhesta.net) → EC2 (Flask)
                                                ├── Aurora MySQL (produtos, clientes, pedidos, CARRINHO)
                                                └── imagens no Amazon S3

DEPOIS (Projeto 03) — carrinho no DynamoDB
Usuário → Route 53 (dynamodb-lab.inhesta.net) → EC2 (Flask)  [MESMA EC2, MESMA APLICAÇÃO]
                                                 ├── banco relacional (produtos, clientes, pedidos)  [RDS Single-AZ OU Aurora]
                                                 ├── Amazon DynamoDB (carrinho-lab)  → PITR habilitado
                                                 └── imagens no Amazon S3  [MESMO BUCKET]
```

O único bloco de dados que troca de tecnologia é o **carrinho**. A camada de computação (EC2), as imagens (S3) e o restante dos dados (relacional) continuam iguais.

## A decisão — mover só o carrinho (persistência poliglota)

A decisão foi **migrar apenas o carrinho** para o **Amazon DynamoDB**, mantendo produtos, clientes e pedidos no banco relacional (Req. 9.2) e **reaproveitando 100% da aplicação, EC2, S3 e observabilidade** (Req. 13.7). O código não é recriado: o único delta de código é o backend DynamoDB dentro de `cart.py`, feito na **fonte de verdade** `01-rds-resilience/app/`. É a essência da série — **cada serviço só entra quando um problema real o justifica**, e aqui o problema é o descasamento entre o perfil do carrinho e o motor relacional. Junto veio a decisão de **habilitar o PITR** na tabela do carrinho, para estudar recuperação para um ponto no tempo.

### O que é reaproveitado (fonte de verdade: Projeto 01)

| Item | Onde vive | Muda no Projeto 03? |
|---|---|---|
| Código da aplicação | `01-rds-resilience/app/` | Só `cart.py` ganha o backend DynamoDB; o resto não muda |
| Layout / front-end | `01-rds-resilience/app/` | Não |
| EC2 (`sg-ec2-lab`, deploy em `/opt/aws-database-lab/`) | Projeto 01 | Não — mesma EC2 |
| Amazon S3 (imagens, `STORAGE_MODE=s3`) | Projeto 01 | Não — mesmo bucket |
| Banco relacional (produtos, clientes, pedidos) | Projeto 01/02 | Continua servindo o relacional |
| CloudWatch / EventBridge | Projeto 01 | Adapta métricas e usa `CloudWatch Alarm → EventBridge → Logs` |
| Route 53 (`inhesta.net`) | Projeto 01 | Adiciona `dynamodb-lab.inhesta.net` para a mesma EC2 |

O código **não é duplicado**: a fonte de verdade é `01-rds-resilience/app/`. O `cart.py` escolhe o backend por `Config.CART_MODE`, então a migração é **reversível por configuração**.

## A implementação — tabela, backend, IAM e DNS

**1) Tabela `carrinho-lab`.** Criada no Console com **chave primária composta**: **PK `cliente_id`** (String) agrupa todos os itens do carrinho de um cliente, e **SK `produto_id`** (String) distingue cada produto. PK + SK juntas formam a chave única por item. Isso habilita três operações eficientes: `PutItem`/`UpdateItem` grava a linha exata `cliente_id + produto_id`; `Query` por `cliente_id` traz **todos** os itens do carrinho de um cliente numa consulta; `DeleteItem` remove um item específico. Os atributos não-chave (`quantidade`, `nome_produto`, `preco`, `atualizado_em`) **não** são declarados na criação — o DynamoDB é *schemaless* fora das chaves. A capacidade é **On-Demand**: não há RCU/WCU provisionadas cobradas 24h "só por existir", o mais seguro contra custo num lab de tráfego esporádico.

**2) Backend `cart.py` com boto3.** O modo `CART_MODE=dynamodb` passou a usar `boto3` de verdade, mantendo a **interface pública estável** (`add_item`, `get_cart`, `remove_item`, `clear_cart`) — por isso **rotas e templates não mudaram**. O recurso `Table` é criado de forma **preguiçosa (lazy)**, então importar `cart.py` não exige credenciais AWS (o modo relacional e os testes locais continuam funcionando). `add_item` incrementa `quantidade` via `UpdateItem` (`ADD`) e desnormaliza `nome_produto`/`preco`; `get_cart` faz `Query` pela PK e converte `Decimal` → `int`/`float`; `remove_item` usa `DeleteItem`; `clear_cart` usa `batch_writer`.

**3) IAM Role de menor privilégio.** O backend roda **na EC2**. Seguindo o padrão do S3 (Projeto 01), **não** gravamos chaves fixas: acrescentamos à **Role já anexada à EC2** (`ec2-imagens-s3-role`) uma política que libera **somente** `GetItem`, `PutItem`, `UpdateItem`, `DeleteItem` e `Query`, **restritas ao ARN da tabela `carrinho-lab`** — nada de `dynamodb:*` (Req. 11). Credenciais temporárias e rotacionadas.

**4) DNS `dynamodb-lab.inhesta.net`.** Registro A na zona `inhesta.net` apontando para o **mesmo IP/Elastic IP** da EC2 já usado por `rds-lab` e `aurora-lab`. Como só o carrinho mudou de banco, muda apenas o **nome de entrada** — os três nomes coexistem apontando para a mesma aplicação.

## A prova — recuperação por PITR

O teste central (detalhado em `TESTES.md`) cumpre o ciclo de recuperação:

1. **Habilitar o PITR** na `carrinho-lab` (retenção padrão de 35 dias). O PITR é o **backup contínuo e incremental** do DynamoDB: uma vez ligado, permite restaurar para **qualquer segundo** dentro de uma janela de **35 dias**. Diferente do backup sob demanda (uma "foto" que você precisa lembrar de tirar antes do incidente), o PITR é granular por segundo e não exige ação prévia — feito exatamente para "alguém apagou dados e eu preciso voltar para pouco antes".
2. **Criar dados e REGISTRAR O HORÁRIO** (em UTC, o fuso que o PITR usa). Esse é o **horário-alvo** do restore — um instante em que os dados ainda existiam.
3. **Excluir os dados propositalmente** — o incidente simulado.
4. **Restaurar para o momento anterior**, gerando a tabela **nova** `carrinho-lab-restaurada`.

A **regra de ouro** é o coração do aprendizado: o PITR **nunca sobrescreve** a tabela original. O restore **sempre cria uma tabela nova** — seguro por design, pois os dados atuais continuam intactos enquanto você inspeciona a cópia restaurada. A comparação final mostra a original sem os itens e a restaurada com os itens do horário-alvo.

## Observabilidade — o evento nasce do alarme

A observabilidade reaproveita o padrão do Projeto 01 ("Dashboard + alarme + Log Group `/aws/events/...` + regra EventBridge"), mudando só o delta do DynamoDB:

- **Dashboard `db-lab-03-dynamodb`** com `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `SuccessfulRequestLatency`, `ThrottledRequests` e `SystemErrors`.
- **Alarme didático** (ex.: `ThrottledRequests > 0`), com SNS opcional.
- **Log Group `/aws/events/db-lab-03-dynamodb`** e **regra EventBridge** com o fluxo **`CloudWatch Alarm → EventBridge → CloudWatch Logs`**.

Diferença conceitual importante vs. o Projeto 01: lá o fluxo principal partia de **eventos do serviço** RDS (como failover). O DynamoDB **não emite** essa família de eventos de serviço; por isso, aqui o evento nasce do **alarme do CloudWatch** — a regra reage às mudanças de estado (`CloudWatch Alarm State Change`) e grava no Log Group.

## Custo — a decisão sobre o relacional (Req. 9.3 e 12)

> O maior custo da série é manter um banco relacional caro ligado sem necessidade.

O Projeto 03 usa o DynamoDB **só para o carrinho**. Produtos, clientes e pedidos continuam no relacional, mas **qual** relacional é uma **decisão de custo** — o Req. 9.3 diz que o Projeto 03 não deve exigir o Aurora ligado se não for necessário. Como o Aurora não tem classe `micro` (mínimo `db.t3.medium`), mantê-lo ocioso é o item mais caro. Antes dos testes, escolha:

| Cenário | Quando | Ação | Custo |
|---|---|---|---|
| **A) Aurora não é necessário** | Foco só em DynamoDB/PITR | **Parar** (reversível, 7 dias) ou **Excluir com snapshot final** o Aurora | O menor (só snapshot/armazenamento) |
| **B) A loja precisa do relacional** | Quer produtos/clientes/pedidos ao vivo | Manter **um** relacional mínimo: RDS Single-AZ (`db.t3.micro`) **ou** Aurora com 1 instância | Baixo |

Nunca manter Aurora + RDS + réplicas ao mesmo tempo sem necessidade (Req. 12.2). A execução está em `EXCLUSAO.md`.

## Resultados e aprendizados

**Resultados** (evidências em `evidencias/`, resultados observados em `TESTES.md`):

- **Carrinho migrado para o DynamoDB sem recriar a aplicação** — só o backend de `cart.py` e a configuração `CART_MODE=dynamodb`. Produtos, clientes e pedidos seguiram no relacional.
- **Tabela `carrinho-lab`** (PK `cliente_id` + SK `produto_id`, On-Demand) validada por operação real (`PutItem`/`GetItem`/`DeleteItem`) e pelo uso da loja — não só pelo status "Ativa" (Req. 14.4).
- **Acesso via `dynamodb-lab.inhesta.net`** apontando para a mesma EC2 de `rds-lab`/`aurora-lab`.
- **PITR habilitado** e **ciclo de recuperação comprovado**: criar → registrar horário → excluir → **restaurar em nova tabela** → comparar (original `Count = 0`, restaurada `Count = 2`).
- **Observabilidade ativa** com as 5 métricas, alarme e o fluxo `CloudWatch Alarm → EventBridge → CloudWatch Logs` comprovado por evento real.
- **Custo controlado:** On-Demand, tabela de teste excluída, PITR desabilitável e Aurora parado/excluído ou reduzido ao mínimo.

**Aprendizados-chave:**

- **NoSQL vs. relacional é decisão por perfil de dado.** O carrinho (efêmero, alto volume de escrita, acesso por chave) encaixa em DynamoDB; produtos/pedidos, com relacionamentos e integridade transacional, seguem no relacional — **persistência poliglota**.
- **A modelagem da chave define a eficiência.** PK `cliente_id` + SK `produto_id` torna `Query` por cliente e `Put`/`Delete` por item operações de custo mínimo, sem `Scan`.
- **On-Demand controla custo:** sem capacidade provisionada ociosa, paga-se pelo que se usa.
- **PITR e a regra "restore = nova tabela":** backup contínuo, janela de 35 dias, restore granular por segundo; o restore materializa uma tabela nova e preserva a original.
- **Reaproveitamento vale ouro:** interface estável de `cart.py` + backend por configuração = zero mudança em rotas/templates/infra.
- **O fluxo de evento parte do alarme:** como o DynamoDB não emite eventos de serviço de failover, a observabilidade de eventos nasce do alarme do CloudWatch.

## Relação com AWS SAA (SAA-C03)

- **DynamoDB (NoSQL gerenciado):** tabelas, PK/SK, modelo *schemaless*, quando escolher NoSQL vs. relacional.
- **Capacidade On-Demand vs. Provisionada:** trade-offs de custo e resposta a picos.
- **Point-in-Time Recovery:** backup contínuo, janela de 35 dias, restore granular; restore cria **sempre** uma nova tabela; PITR vs. backup sob demanda.
- **Segurança e IAM:** acesso via IAM Role com credenciais temporárias e política de **menor privilégio** restrita ao ARN, em vez de chaves fixas.
- **Observabilidade:** métricas do DynamoDB, alarmes e integração EventBridge/CloudWatch Logs.
- **Custo:** dimensionar o mínimo, não manter recursos caros ociosos, remover artefatos de teste.

## Conclusão

O Projeto 03 mostra que **escolher o banco certo para cada dado** é uma decisão de arquitetura de primeira ordem: mover **apenas o carrinho** para o DynamoDB — mantendo o restante no relacional — resolve o descasamento de perfil sem reescrever a aplicação, apenas trocando o backend de `cart.py` e a configuração. Somando o **PITR**, ganhamos recuperação granular e gerenciada, com a garantia de que o restore nunca destrói a original. Com o carrinho migrado, a recuperação comprovada e a observabilidade ativa, o ambiente fica pronto para o **Projeto 04**, que reaproveita `dynamodb-lab.inhesta.net` e evolui a `carrinho-lab` para **Global Tables** e **AWS Backup**.
