# Exclusão / Limpeza — NoSQL Gerenciado e Recuperação no Tempo: Carrinho no DynamoDB

> O que **remover, reduzir ou desabilitar** ao encerrar o Projeto 03, antes de iniciar o Projeto 04 (Global Tables + AWS Backup), **sem quebrar a loja/carrinho** nem perder o que o Projeto 04 reaproveita. Regra de ouro: **manter o que o Projeto 04 usa** e **remover o que foi criado só para os testes**.
>
> Substitua `<REGIAO>` (ex.: `us-east-1`) pelos valores do seu ambiente.

## Princípios de custo

- **Req. 12.2:** nunca manter simultaneamente, sem necessidade, vários bancos relacionais (Aurora + RDS + réplicas). No Projeto 03 o alvo é **no máximo um relacional ativo** (o mínimo) **ou nenhum**, além do DynamoDB para o carrinho.
- **Req. 12.3:** ao terminar cada teste, informe o que pode ser excluído.
- O DynamoDB On-Demand não tem custo fixo de "instância ligada" — o custo vem de requisições/armazenamento (mínimo no lab) e do PITR (backup contínuo, enquanto ligado).

---

## A) Remover / reduzir antes do Projeto 04

### 1. Tabela de teste `carrinho-lab-restaurada` — REMOVER (normalmente já excluída na Tarefa 39)
- Tabela nova gerada pelo restore do PITR para comprovar a recuperação. É **temporária** e não segue para o Projeto 04.
- Confirmar que não existe: DynamoDB → **Tabelas** → a lista deve ter `carrinho-lab`, **sem** `carrinho-lab-restaurada`.
- Se ainda existir: DynamoDB → `carrinho-lab-restaurada` → **Ações → Excluir tabela** (descartável, sem snapshot).

### 2. Itens de teste/carga na `carrinho-lab` — REMOVER
- Itens sintéticos: `cliente-teste-36` (validação), `cliente-pitr-39` (PITR) e, se provocou throttling, `carga-40` (`p-1`..`p-200`).
- Exemplo (apagar a carga do teste de alarme, na EC2):
  ```bash
  for i in $(seq 1 200); do \
    aws dynamodb delete-item --region <REGIAO> --table-name carrinho-lab \
      --key "{\"cliente_id\":{\"S\":\"carga-40\"},\"produto_id\":{\"S\":\"p-$i\"}}" >/dev/null 2>&1; \
  done
  aws dynamodb scan --region <REGIAO> --table-name carrinho-lab \
    --filter-expression "cliente_id = :c" --expression-attribute-values '{":c":{"S":"carga-40"}}'   # espera Count: 0
  ```
  Repita para `cliente-teste-36`/`cliente-pitr-39` se ainda existirem.
- **Se mudou a tabela para Provisionado no teste de throttling:** reverter para **On-Demand** (DynamoDB → `carrinho-lab` → Configurações adicionais/Capacidade → Sob demanda), para não manter RCU/WCU cobrando.

### 3. PITR da `carrinho-lab` — OPCIONAL desabilitar
- O Projeto 04 usa AWS Backup + Global Tables e **não depende do PITR**. Após os testes de recuperação, pode desligar para encerrar o custo do backup contínuo (mínimo no lab).
- DynamoDB → `carrinho-lab` → aba **Backups** → **Recuperação para um ponto no tempo** → **Desativar/Editar**.
- Se preferir manter a proteção ligada, o custo permanece desprezível — a decisão é sua.

### 4. Estado do banco relacional / Aurora — CONFIRMAR o cenário adotado
- **Cenário A:** confirmar que o Aurora está **Parado** ou **Excluído** (com snapshot final). É o estado mais econômico; produtos/clientes/pedidos ficam indisponíveis, mas o carrinho no DynamoDB funciona.
- **Cenário B:** confirmar **apenas um** relacional ativo e mínimo — RDS Single-AZ (`db.t3.micro`) **ou** Aurora 1 instância — nunca os dois. Esvaziar `DB_READER_ENDPOINT` se ficou só com o writer.
- **Req. 12.2:** nunca Aurora + RDS + réplicas ao mesmo tempo sem necessidade.

---

## B) Manter (reaproveitado no Projeto 04)

| Recurso | Por quê |
|---|---|
| **Tabela `carrinho-lab`** (On-Demand, limpa) | O Projeto 04 adiciona réplica em `us-east-1`, virando **Global Table**. Não excluir. |
| **EC2 + aplicação** (`CART_MODE=dynamodb`) | Mesma app/EC2; o Projeto 04 não cria nova aplicação. |
| **IAM Role da EC2** (S3 + `carrinho-lab`) | Continua acessando a tabela por credenciais temporárias. |
| **Amazon S3** (imagens) | A loja segue servindo do S3. Custo ínfimo. |
| **Route 53** (`dynamodb-lab.inhesta.net`) | O Projeto 04 reaproveita este DNS (Req. 10.1). |
| **Observabilidade** (dashboard, alarme, Log Group, regra EventBridge) | Documenta a série; custo baixíssimo. Limpeza final no Projeto 04. |

> **Elastic IP:** se for **parar** a EC2 entre projetos, libere o EIP ocioso; se ficar ligada, mantenha para não reeditar o DNS.

---

## C) Sequência recomendada (antes do Projeto 04)

1. Confirmar que `carrinho-lab-restaurada` foi excluída (A.1).
2. Limpar itens de teste/carga da `carrinho-lab` e garantir **On-Demand** (A.2).
3. (Opcional) Desabilitar o PITR (A.3) — o Projeto 04 não depende dele.
4. Confirmar o estado do relacional conforme o cenário A/B (A.4): no máximo **um** relacional ativo, ou nenhum.
5. Verificar o que MANTER (seção B): `carrinho-lab` limpa e ativa, EC2 + app, IAM Role, S3, `dynamodb-lab.inhesta.net`, observabilidade.
6. Validar: abrir `http://dynamodb-lab.inhesta.net:5000/`, adicionar/remover um item e conferir na `carrinho-lab`.

**Estado-alvo:** carrinho em produção na `carrinho-lab` (On-Demand, limpa), acessível por `dynamodb-lab.inhesta.net`, observabilidade ativa; tabelas/itens de teste removidos; PITR desabilitado (opcional); no máximo um relacional ativo (ou nenhum). Ambiente enxuto e pronto para o Projeto 04 evoluir a `carrinho-lab` para **Global Tables** + **AWS Backup**.
