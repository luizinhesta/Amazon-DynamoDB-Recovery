# Arquitetura — NoSQL Gerenciado e Recuperação no Tempo: Carrinho no DynamoDB

## ANTES vs DEPOIS

```text
ANTES
Aplicação → banco relacional (produtos, clientes, pedidos, carrinho)

DEPOIS
Aplicação
├── banco relacional (produtos, clientes, pedidos)
└── Amazon DynamoDB (carrinho)  → PITR habilitado
```

## Modelo de dados DynamoDB

```text
Tabela: carrinho-lab
PK: cliente_id (String)
SK: produto_id (String)
Atributos: quantidade (N), nome_produto (S), preco (N), atualizado_em (S)
Capacidade: On-Demand
```

## Recuperação (PITR)

```text
Criar dados → registrar horário → excluir dados → Restore (para momento anterior)
Resultado: o restore gera uma NOVA tabela (ex.: carrinho-lab-restaurada)
Comparar: tabela original vs tabela restaurada
```

Regra de ouro (Req. 9.7): o restore do PITR **nunca sobrescreve** a tabela original — ele **sempre cria uma tabela nova**. Detalhe do ciclo em `IMPLANTACAO.md` (Passos 5.1–5.5) e `TESTES.md` (T3D.3/T3D.4).

## Observabilidade (delta em relação ao Projeto 01)

```text
Dashboard db-lab-03-dynamodb (métricas AWS/DynamoDB da carrinho-lab):
  ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits,
  SuccessfulRequestLatency, ThrottledRequests, SystemErrors

Fluxo obrigatório do evento (Req. 9.9):
  CloudWatch Alarm → EventBridge → CloudWatch Logs (/aws/events/db-lab-03-dynamodb)
```

Diferença em relação ao Projeto 01: lá o fluxo principal partia de **eventos do serviço RDS** (`RDS → EventBridge → Logs`). O DynamoDB não emite essa família de eventos de serviço, então o evento nasce do **alarme do CloudWatch** (`CloudWatch Alarm State Change`). Detalhes em `IMPLANTACAO.md` (Passos 6.1–6.5).

## Controle de custo (Requisitos 9.3 e 12)

No Projeto 03, o DynamoDB cobre **apenas o carrinho**; produtos, clientes e pedidos ficam no relacional. Se o Aurora do Projeto 02 **não for necessário** aos testes, **parar ou excluir** o cluster para evitar cobrança do relacional mais caro da série (o Aurora não tem classe `micro`). Se a loja precisar do relacional durante o Projeto 03, manter um **relacional mínimo** (RDS Single-AZ barato a partir de snapshot, ou Aurora com 1 única instância).

A decisão detalhada (cenários, trade-off e passos no Console) está no `README.md` (seção "Custo — a decisão sobre o relacional") e no `IMPLANTACAO.md` (seção "Decisão sobre o banco relacional e o Aurora"), dando sequência à limpeza do Projeto 02. A confirmação do estado final e a lista de recursos removíveis antes do Projeto 04 estão em `EXCLUSAO.md`.

## Resultado alcançado (delta do Projeto 03)

O único bloco de dados que troca de tecnologia é o **carrinho** (relacional → DynamoDB). EC2, S3, banco relacional (produtos/clientes/pedidos), Route 53 e observabilidade são reaproveitados. O que muda:

| Aspecto | ANTES (Projeto 02) | DEPOIS (Projeto 03) |
|---|---|---|
| Carrinho | tabelas relacionais `carrinho`/`itens_carrinho` (com JOIN) | Amazon DynamoDB `carrinho-lab` (PK `cliente_id` + SK `produto_id`, On-Demand) |
| Backend do carrinho | `cart.py` em `CART_MODE=relational` | `cart.py` em `CART_MODE=dynamodb` (boto3) — mesma interface pública |
| Produtos/clientes/pedidos | banco relacional | banco relacional (sem mudança) |
| Recuperação do carrinho | dump/restore manual do relacional | **PITR** (restore granular por segundo, gera nova tabela) |
| Observabilidade | eventos do serviço RDS/Aurora | `CloudWatch Alarm → EventBridge → CloudWatch Logs` (DynamoDB) |
| DNS | `aurora-lab.inhesta.net` | `dynamodb-lab.inhesta.net` (mesma EC2) |

Resultado comprovado: carrinho persistindo no DynamoDB, PITR habilitado, restore para momento anterior gerando `carrinho-lab-restaurada` (original sem os itens × restaurada com os itens do horário-alvo) e o fluxo de observabilidade do DynamoDB funcional. Evidências em `evidencias/`; resultados observados em `TESTES.md`.
