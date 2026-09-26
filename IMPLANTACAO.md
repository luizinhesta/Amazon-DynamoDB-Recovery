# Implantação — NoSQL Gerenciado e Recuperação no Tempo: Carrinho no DynamoDB

> Guia enxuto de **criação** dos recursos pela console + comandos na EC2. O Projeto 03 muda **apenas o carrinho**: relacional → **Amazon DynamoDB** (`carrinho-lab`). Produtos, clientes e pedidos continuam no relacional; EC2, S3, código e observabilidade são reaproveitados.
>
> **Testes e validações** (carrinho no DynamoDB, restore por PITR, dashboard, alarme) ficam no **`TESTES.md`** — este guia foca em **subir os recursos**. Ao terminar cada fase, rode os testes correspondentes do `TESTES.md`.

## Conceitos-chave do Projeto 03 (leia antes de começar)

- **Por que DynamoDB só para o carrinho:** o carrinho é **efêmero, de alto volume de escrita e acesso por chave** (por cliente) — perfil ideal para NoSQL. Produtos/clientes/pedidos, com relacionamentos e integridade, seguem no relacional. Usar cada banco para o que faz melhor é **persistência poliglota**.
- **Chave composta (PK + SK):** `cliente_id` (partição) agrupa o carrinho de um cliente; `produto_id` (ordenação) distingue cada produto. Juntas formam a chave única por item — habilitam `Query` por cliente e `Put`/`Delete` por item sem `Scan`.
- **On-Demand:** sem capacidade (RCU/WCU) provisionada cobrada 24h; paga-se pelo uso. É o mais seguro contra custo em tráfego esporádico de lab.
- **IAM Role, não chaves fixas:** a EC2 acessa o DynamoDB com credenciais temporárias da Role já anexada, com política de **menor privilégio** restrita ao ARN da tabela.
- **PITR (Point-in-Time Recovery):** backup contínuo que permite restaurar para **qualquer segundo** dos últimos **35 dias**. O restore **sempre cria uma tabela nova** — nunca sobrescreve a original.

## Pré-requisito: projetos anteriores concluídos

- **EC2** rodando a aplicação Flask (systemd) em `<APP_DIR>` (ex.: `/opt/aws-database-lab`), com IAM Role já anexada (a do S3).
- **Amazon S3** servindo as imagens (`STORAGE_MODE=s3`) e **Route 53** com a zona hospedada ativa.
- **Observabilidade** dos projetos anteriores preservada (será só estendida).
![Descrição da imagem](<imagens/imagem%20(15).png>)
![Descrição da imagem](<imagens/imagem%20(17).png>)

![Descrição da imagem](<imagens/imagem%20(3).png>)
![Descrição da imagem](<imagens/imagem%20(2).png>)


## Convenções (variáveis)

| Variável | O que é |
|---|---|
| `<REGIAO>` | Região da tabela e da EC2 (ex.: `us-east-1`) |
| `<ACCOUNT_ID>` | ID da conta AWS (12 dígitos) |
| `<APP_DIR>` | Diretório da aplicação na EC2 |
| `<IAM_ROLE_EC2>` | Role já anexada à EC2 (ex.: `ec2-imagens-s3-role`) |
| `<DOMINIO_DYNAMO>` | Subdomínio do Projeto 03 (ex.: `dynamodb-lab.<sua-zona>`) |

---

## Fase 0 — Decisão de custo do banco relacional (ANTES de começar)

> O DynamoDB só cobre o carrinho. O relacional (Aurora/RDS) é o item mais caro da série — decida o que fazer antes de subir o resto. O Aurora não tem classe `micro`, então mantê-lo ocioso é o maior custo.

- **Cenário A (mais econômico) — não precisa da loja relacional durante os testes:** **parar** ou **excluir** o Aurora (RDS → cluster → Ações → Parar temporariamente, ou excluir com snapshot final). O carrinho no DynamoDB funciona sozinho; produtos/clientes/pedidos ficam indisponíveis.
- **Cenário B — quer a loja inteira funcional:** manter **um único** relacional mínimo — RDS MySQL Single-AZ (`db.t3.micro`) **ou** Aurora com 1 instância (sem Readers; esvaziar `DB_READER_ENDPOINT`).
- **Regra (Req. 12.2):** nunca manter Aurora + RDS + réplicas ligados ao mesmo tempo sem necessidade. Detalhes em `EXCLUSAO.md`.

---

## Fase 1 — Criar a tabela DynamoDB `carrinho-lab` (Tarefa 36)

### 1.1 Criar a tabela
1. Console → confirmar a região **`<REGIAO>`** (canto superior direito).
2. Serviços → **DynamoDB** → **Tabelas** → **Criar tabela**.
3. Campos:
   - Nome da tabela: **`carrinho-lab`**
   - Chave de partição: **`cliente_id`** — tipo **String**
   - Chave de classificação (sort key): **`produto_id`** — tipo **String**
   - Configurações: **Personalizar configurações**
   - Modo de capacidade: **Sob demanda** (On-Demand)
4. Manter padrões: classe **Standard**, criptografia da AWS, sem índices secundários, PITR desligado (habilitado na Fase 4).
5. **Criar tabela**. Aguardar status **Ativa**.

> Os atributos não-chave (`quantidade`, `nome_produto`, `preco`, `atualizado_em`) **não** são declarados aqui — o DynamoDB é *schemaless* fora das chaves; a aplicação os grava.

![Descrição da imagem](<imagens/imagem%20(16).png>)

### 1.2 Dar permissão IAM à EC2 (menor privilégio)

> A EC2 já tem a Role do S3. Adicione a ela uma política inline só para a `carrinho-lab` — sem chaves fixas, sem `dynamodb:*`, restrita ao ARN da tabela. Uma instância só tem **uma** Role, então acrescentamos a política à existente.

1. Serviços → **IAM** → **Funções** → abrir **`<IAM_ROLE_EC2>`**.
2. **Adicionar permissões → Criar política em linha** → aba **JSON**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "CarrinhoLabAcessoMinimo",
         "Effect": "Allow",
         "Action": [
           "dynamodb:GetItem",
           "dynamodb:PutItem",
           "dynamodb:UpdateItem",
           "dynamodb:DeleteItem",
           "dynamodb:Query"
         ],
         "Resource": "arn:aws:dynamodb:<REGIAO>:<ACCOUNT_ID>:table/carrinho-lab"
       }
     ]
   }
   ```
3. Nome: `politica-carrinho-dynamodb-lab` → **Criar política**.

> **Validação:** confirme a tabela e a permissão com uma operação real (`put`/`get`/`delete`) — ver `TESTES.md`, **T3D.1** e Bloco A. "Ativa" no Console não basta (Req. 14.4).

![Descrição da imagem](<imagens/imagem%20(5).png>)
![Descrição da imagem](<imagens/imagem%20(8).png>)

---

## Fase 2 — Apontar o carrinho para o DynamoDB (Tarefa 37)

> O backend DynamoDB já está no código-fonte (`cart.py`, escolhido por `CART_MODE`). Aqui é **só configuração** — rotas e templates não mudam. A migração é reversível trocando `CART_MODE` de volta para `relational`.

### 2.1 Ajustar o `.env` e reiniciar (na EC2)
```bash
cd <APP_DIR>
sudo -u ec2-user .venv/bin/pip install -r requirements.txt   # garante o boto3
sudo cp .env .env.p02.bak
sudo nano .env
```
Ajustar:
```dotenv
CART_MODE=dynamodb
DYNAMODB_TABLE=carrinho-lab
AWS_REGION=<REGIAO>
LAB_PROJECT=Projeto 03 — AWS DynamoDB Recovery Lab
LAB_ARCH_VERSION=Projeto 03 — Carrinho no DynamoDB (carrinho-lab)
```
Manter `DB_WRITER_ENDPOINT`/`DB_READER_ENDPOINT` e `STORAGE_MODE=s3` como estavam.
```bash
sudo systemctl restart aws-database-lab
```

> **Validação:** carrinho persistindo no DynamoDB pela loja — ver `TESTES.md`, **T3D.1**.

---

## Fase 3 — DNS do Projeto 03 (Tarefa 38)

> Só o carrinho mudou de banco; a EC2 é a mesma. O novo nome aponta para o **mesmo IP** de `rds-lab`/`aurora-lab` — muda apenas o ponto de entrada oficial.

1. Confirmar o **IP público / Elastic IP** da EC2 (EC2 → Instâncias). Sem Elastic IP, aloque um (o IP muda a cada stop/start e quebraria o DNS).
2. Route 53 → Zonas hospedadas → sua zona → **Criar registro**:
   - Nome: subdomínio do Projeto 03 (resulta em `<DOMINIO_DYNAMO>`)
   - Tipo: **A** · Alias: **Off**
   - Valor: **IP da EC2** (só o IP, sem `http://` nem porta)
   - TTL: `300` · Roteamento: **Simples**
3. **Criar registros**. *(Os registros `rds-lab`/`aurora-lab` podem coexistir — mesmo IP.)*

> **Validação:** `nslookup` + abrir a loja pelo domínio (porta 5000, que o DNS não altera) — ver `TESTES.md`, **T3D.0**.
![Descrição da imagem](<imagens/imagem%20(19).png>)

---

## Fase 4 — Habilitar o PITR (Tarefa 39)

> O PITR é o backup contínuo (janela de 35 dias) e **só protege eventos depois de ativado** — por isso é habilitado **antes** de qualquer teste de exclusão/restore.

DynamoDB → `carrinho-lab` → aba **Backups** → **Recuperação para um ponto no tempo** → **Ativar** (retenção padrão 35 dias). O status passa a **Ativado** e o Console mostra o intervalo restaurável.

> **Teste de recuperação** (criar dados → registrar horário → excluir → restaurar em nova tabela → comparar): ver `TESTES.md`, **T3D.2/T3D.3/T3D.4 (Bloco A)**. É o coração do Projeto 03.

---

## Fase 5 — Observabilidade do DynamoDB (Tarefa 40)

> Reaproveita o padrão dos projetos anteriores. Aqui o fluxo obrigatório parte do **alarme**: `CloudWatch Alarm → EventBridge → CloudWatch Logs` — porque o DynamoDB não emite eventos de serviço de failover. Tudo na região `<REGIAO>`.

### 5.1 Dashboard `db-lab-03-dynamodb`
CloudWatch → **Painéis** → **Criar painel** → `db-lab-03-dynamodb`. Widgets (Linha) com métricas do namespace **DynamoDB**, `TableName = carrinho-lab`:
- `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits` (capacidade)
- `SuccessfulRequestLatency` (por operação; Average) (latência)
- `ThrottledRequests` (Sum), `SystemErrors` (Sum) (saúde)

![Descrição da imagem](<imagens/imagem%20(13).png>)
![Descrição da imagem](<imagens/imagem%20(14).png>)

### 5.2 Alarme didático
CloudWatch → **Alarmes** → **Criar alarme** → DynamoDB → `carrinho-lab` → **ThrottledRequests**:
- Estatística **Sum**, período **1 min**, condição **Maior que 0**
![Descrição da imagem](<imagens/imagem%20(12).png>)

- Nome: `db-lab-03-dynamodb-throttling`
- Tratar dados ausentes: **Tratar como não violado (bom)**
- SNS: opcional (e-mail)

![Descrição da imagem](<imagens/imagem%20(10).png>)
![Descrição da imagem](<imagens/imagem%20(11).png>)

### 5.3 Log Group
CloudWatch → **Logs** → **Grupos de logs** → **Criar** → nome **`/aws/events/db-lab-03-dynamodb`**, retenção **30 dias**.

### 5.4 Regra EventBridge
EventBridge → **Regras** → barramento `default` → **Criar regra** `db-lab-03-dynamodb-alarme-eventos`:
- Tipo: **Regra com padrão de evento** → **Padrão personalizado (JSON)**:
  ```json
  {
    "source": ["aws.cloudwatch"],
    "detail-type": ["CloudWatch Alarm State Change"],
    "resources": ["arn:aws:cloudwatch:<REGIAO>:<ACCOUNT_ID>:alarm:db-lab-03-dynamodb-throttling"]
  }
  ```
- Alvo: **Grupo de logs do CloudWatch** → `/aws/events/db-lab-03-dynamodb`.
- Criar regra.

> **Validação:** forçar o disparo do alarme e localizar o evento no Log Group — ver `TESTES.md`, **T3D.5/T3D.6 (Bloco B)**.

---

## Encerramento — limpeza de custo

Para reduzir custo antes do Projeto 04 (excluir tabela de teste, limpar itens de teste, decisão do relacional, PITR), siga o **`EXCLUSAO.md`**.

**Estado-alvo:** EC2 + `carrinho-lab` (On-Demand, limpa) + S3 + `<DOMINIO_DYNAMO>`, observabilidade ativa, tabela/itens de teste removidos, no máximo um relacional ativo (ou nenhum). A `carrinho-lab` segue para o Projeto 04 (vira Global Table).
