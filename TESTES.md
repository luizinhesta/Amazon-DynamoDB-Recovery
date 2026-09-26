# Testes — NoSQL Gerenciado e Recuperação no Tempo: Carrinho no DynamoDB

Matriz de testes do delta do Projeto 03: carrinho no DynamoDB, PITR, restore e observabilidade. Padrão de cada linha: **Teste / Objetivo / Procedimento / Resultado esperado / Resultado observado / Evidência** (Req. 13.5), com testes positivos e de falha/recuperação.

**Convenções**
- Substitua `<REGIAO>` (ex.: `us-east-1`) e o domínio `dynamodb-lab.inhesta.net` pelos valores do seu ambiente.
- Os comandos `aws ...` rodam **na EC2**, usando a IAM Role da instância (sem chaves fixas).
- "Resultado observado" fica `( a preencher )` até a execução; salve os prints em `evidencias/` no padrão `NN-descricao.png`.
- Passo a passo de criação dos recursos: `IMPLANTACAO.md` (ou `IMPLANTACAO-RESUMO.md`). Aqui o foco é **validar/testar**.

**Cobertura (positivos e de falha)**
- **Positivos:** T3D.0 (DNS), T3D.1 (carrinho no DynamoDB), T3D.2 (PITR habilitado), T3D.4 (comparação), T3D.5 (dashboard).
- **De falha / recuperação:** **T3D.3** (exclusão de dados + restore por PITR); **T3D.6** (alarme dispara e evento é registrado via `CloudWatch Alarm → EventBridge → CloudWatch Logs`).

---

## Matriz de testes

| Teste | Objetivo | Procedimento | Resultado esperado | Resultado observado | Evidência |
|---|---|---|---|---|---|
| T3D.0 Acesso via DNS | Acesso pelo domínio (Req. 9.1) | `nslookup dynamodb-lab.inhesta.net`; abrir `http://dynamodb-lab.inhesta.net:5000/` e `/database-lab` | DNS resolve para o IP da EC2 (o mesmo de `rds-lab`/`aurora-lab`); loja abre; `/database-lab` mostra o carrinho no DynamoDB (`carrinho-lab`) e o relacional (se ativo) | _a preencher_ | `39-... .png` (nslookup + navegador) |
| T3D.1 Carrinho no DynamoDB | Migração do carrinho (Req. 9.1/9.2) | Com `CART_MODE=dynamodb`: adicionar item ao carrinho na loja; conferir na tabela (DynamoDB → Explorar itens); alterar quantidade; remover/esvaziar | Cada ação reflete na `carrinho-lab`: `add` cria/atualiza o item (`quantidade`, `nome_produto`, `preco`, `atualizado_em`); `remover`/`esvaziar` exclui. Produtos/clientes/pedidos seguem no relacional | _a preencher_ | `37-carrinho-dynamodb.png` |
| T3D.2 PITR habilitado | Habilitar o backup contínuo (Req. 9.5) | DynamoDB → `carrinho-lab` → aba **Backups** → **Recuperação para um ponto no tempo** → **Ativar** (retenção 35 dias) | Status **Ativado**; Console mostra o intervalo restaurável (mais antigo/mais recente). PITR só protege eventos **após** a ativação | _a preencher_ | `39-pitr-ativado.png` |
| T3D.3 Exclusão + restore por PITR (falha/recuperação) | Recuperar dados excluídos restaurando para instante anterior (Req. 9.6/9.7) | Ver o **roteiro detalhado (bloco A)** abaixo: criar 2 itens → registrar horário → excluir → restaurar em nova tabela | Restore cria a **NOVA tabela** `carrinho-lab-restaurada` (Ativa) com o estado do horário-alvo (2 itens); a **original permanece intacta e sem os itens** — o PITR **nunca sobrescreve** a original | _a preencher_ | `39-restore-nova-tabela.png` |
| T3D.4 Comparação original × restaurada | Comprovar a recuperação (Req. 9.7) | Ver o **bloco A, passo 5** abaixo: `scan` filtrando `cliente-pitr-39` nas duas tabelas | Original: `Count = 0`; Restaurada: `Count = 2` (Notebook qtd 2, Mouse qtd 1). Restore comprovado como **sempre nova tabela** | _a preencher_ | `39-comparacao-original-restaurada.png` |
| T3D.5 Dashboard de métricas | Observabilidade da tabela (Req. 9.8) | CloudWatch → Painéis → `db-lab-03-dynamodb`. Gerar tráfego no carrinho e aguardar alguns minutos | Painel exibe as 5 métricas (`ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `SuccessfulRequestLatency`, `ThrottledRequests`, `SystemErrors`); capacidade/latência com dados; throttling/erros em zero (lab saudável) | _a preencher_ | `40-dashboard-dynamodb.png` |
| T3D.6 Alarme → evento no Log Group (falha) | Comprovar `CloudWatch Alarm → EventBridge → CloudWatch Logs` (Req. 9.9) | Ver o **roteiro detalhado (bloco B)** abaixo: confirmar alarme/regra, forçar o disparo e localizar o evento | Alarme transiciona **OK → ALARM**; surge no Log Group `/aws/events/db-lab-03-dynamodb` um evento `CloudWatch Alarm State Change` (`detail.state.value = "ALARM"`); o Logs Insights lista a transição com nome e motivo | _a preencher_ | `40-evento-no-log-group.png`, `40-logs-insights-alarme.png` |

---

## Roteiro detalhado

### Bloco A — Exclusão de dados e recuperação por PITR (T3D.3 e T3D.4)

> Comprova a **recuperação para um ponto no tempo**. A ideia: criar dados, anotar um horário em que eles existiam, excluí-los (simular o incidente) e restaurar **para aquele horário** numa tabela nova. Pré-requisito: PITR já habilitado (T3D.2).

**1) Criar 2 itens de teste** (na EC2):
```bash
aws dynamodb put-item --region <REGIAO> --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-pitr-39"},"produto_id":{"S":"produto-1"},"quantidade":{"N":"2"},"nome_produto":{"S":"Notebook"},"preco":{"N":"3500"},"atualizado_em":{"S":"2024-01-01T12:00:00Z"}}'
aws dynamodb put-item --region <REGIAO> --table-name carrinho-lab \
  --item '{"cliente_id":{"S":"cliente-pitr-39"},"produto_id":{"S":"produto-2"},"quantidade":{"N":"1"},"nome_produto":{"S":"Mouse"},"preco":{"N":"120"},"atualizado_em":{"S":"2024-01-01T12:00:00Z"}}'
```

**2) Conferir que existem** (esperado `Count: 2`) e **REGISTRAR O HORÁRIO** (o PITR usa UTC):
```bash
aws dynamodb scan --region <REGIAO> --table-name carrinho-lab \
  --filter-expression "cliente_id = :c" --expression-attribute-values '{":c":{"S":"cliente-pitr-39"}}'
date -u "+%Y-%m-%dT%H:%M:%SZ"     # anote este horário-alvo
```
> Aguarde ~1-2 min antes de excluir, para o horário-alvo entrar na janela restaurável (o "mais recente restaurável" fica alguns minutos atrás do relógio real).

**3) Excluir os itens** (simular o incidente; esperado depois `Count: 0`):
```bash
aws dynamodb delete-item --region <REGIAO> --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-pitr-39"},"produto_id":{"S":"produto-1"}}'
aws dynamodb delete-item --region <REGIAO> --table-name carrinho-lab \
  --key '{"cliente_id":{"S":"cliente-pitr-39"},"produto_id":{"S":"produto-2"}}'
aws dynamodb scan --region <REGIAO> --table-name carrinho-lab \
  --filter-expression "cliente_id = :c" --expression-attribute-values '{":c":{"S":"cliente-pitr-39"}}'
```

**4) Restaurar para o horário-alvo** (Console): DynamoDB → `carrinho-lab` → aba **Backups** → **Restaurar**:
- Nome da nova tabela: **`carrinho-lab-restaurada`**
- Data e hora: o **horário-alvo do passo 2** (em UTC), dentro do intervalo restaurável
- **Não** use "Horário mais recente restaurável" (traria o estado atual, já sem os dados)
- Aguardar a nova tabela ficar **Ativa**. A original continua intacta (sem os itens).

**5) Comparar** original × restaurada:
```bash
# original (esperado Count: 0)
aws dynamodb scan --region <REGIAO> --table-name carrinho-lab \
  --filter-expression "cliente_id = :c" --expression-attribute-values '{":c":{"S":"cliente-pitr-39"}}'
# restaurada (esperado Count: 2)
aws dynamodb scan --region <REGIAO> --table-name carrinho-lab-restaurada \
  --filter-expression "cliente_id = :c" --expression-attribute-values '{":c":{"S":"cliente-pitr-39"}}'
```
> Conclusão a documentar: o restore do PITR **sempre gera uma tabela nova** e **jamais sobrescreve** a original. Após comparar, exclua a `carrinho-lab-restaurada` (limpeza de custo — ver `EXCLUSAO.md`); **não** exclua a `carrinho-lab`.

### Bloco B — Alarme dispara e evento no Log Group (T3D.6)

> Comprova o fluxo obrigatório **`CloudWatch Alarm → EventBridge → CloudWatch Logs`**. Pré-requisitos: alarme `db-lab-03-dynamodb-throttling` (estado OK) e regra `db-lab-03-dynamodb-alarme-eventos` habilitada.

**1) Forçar o disparo do alarme** (na EC2) — mais simples que provocar throttling real:
```bash
aws cloudwatch set-alarm-state --region <REGIAO> \
  --alarm-name db-lab-03-dynamodb-throttling \
  --state-value ALARM --state-reason "Teste do fluxo Alarm -> EventBridge -> Logs"
```
> Alternativa (throttling real): mudar a tabela para Provisionado com 1 RCU/1 WCU, gerar rajada de `put-item` e reverter para On-Demand depois. O `set-alarm-state` é o caminho didático.

**2) Localizar o evento** — CloudWatch → Logs → `/aws/events/db-lab-03-dynamodb` → abrir o fluxo mais recente, ou usar o **Logs Insights**:
```text
fields @timestamp, `detail-type` as tipo, detail.alarmName as alarme, detail.state.value as estado, detail.state.reason as motivo
| filter `detail-type` = "CloudWatch Alarm State Change"
| sort @timestamp desc | limit 50
```
Esperado: um evento com `detail-type = "CloudWatch Alarm State Change"`, o alarme do DynamoDB e `detail.state.value = "ALARM"`. O alarme volta sozinho a **OK** na próxima avaliação (gera um segundo evento `OK`).
