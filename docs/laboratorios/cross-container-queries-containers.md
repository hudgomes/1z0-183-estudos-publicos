---
title: Consultas cross-container com CONTAINERS()
description: Relatórios agregados entre Application PDBs a partir do Application Root
---

# Consultas cross-container com CONTAINERS()

## Como funciona

`CONTAINERS()` consulta um objeto comum no root e nas PDBs abertas subordinadas. Em um Application Root, isso permite gerar um relatório único sobre todas as Application PDBs.

O objeto deve existir no root e nas PDBs. Um objeto `SHARING=METADATA` é um caso típico: a definição é comum e cada tenant mantém seus próprios dados.

## O que a consulta distribui

Ao executar `CONTAINERS(objeto)` no root apropriado, o Oracle envia a parte elegível da consulta aos containers participantes e reúne o resultado. A pseudoidentificação `CON_ID` permite rastrear a origem de cada linha. Isso evita abrir uma conexão por tenant e montar o agregado na aplicação.

O alcance depende do root atual. No `CDB$ROOT`, a visão pode envolver PDBs da CDB quando o objeto e os privilégios suportam o uso. No Application Root, o cenário típico envolve suas Application PDBs. Um objeto somente local, sem correspondência nos containers, não se torna automaticamente comum por estar dentro de `CONTAINERS()`.

Estado e acessibilidade também importam. PDB fechada não executa a parte local; modo restrito e privilégios podem impedir participação. Portanto, um total cross-container representa os containers elegíveis naquele momento, não necessariamente todos os tenants cadastrados.

## Arquitetura do laboratório

```text
LAB_APP_ROOT
  ├─ LAB_APP_PDB1 -> LAB_SALES.ORDERS
  └─ LAB_APP_PDB2 -> LAB_SALES.ORDERS
```

## Criar o objeto comum

Durante uma operação de instalação no Application Root:

```sql
create user lab_sales identified by "<SENHA_FORTE_EXCLUSIVA>"
  container=all;

create table lab_sales.orders (
  order_id number primary key,
  amount   number(12,2) not null
) sharing=metadata;
```

Sincronize a aplicação nas duas Application PDBs e insira dados diferentes em cada uma.

`SHARING=METADATA` compartilha a definição criada durante o ciclo da aplicação. Cada Application PDB mantém seus próprios segmentos e linhas. Com `SHARING=DATA`, os dados residem no Application Root e são vistos pelos tenants; com `EXTENDED DATA`, dados comuns e locais coexistem. A escolha muda o significado do agregado e deve ser feita ao desenhar o objeto.

## Test case — agregado global

Conectado ao Application Root como o application common user proprietário:

```sql
select sum(amount) total_amount
  from containers(lab_sales.orders);
```

O resultado combina linhas do root e das Application PDBs abertas, exceto containers em modo restrito.

Empurre filtros e agregações para a consulta distribuída. Selecionar todas as linhas para somar depois aumenta transferência e memória. Predicados em colunas reais podem ser avaliados localmente; funções não suportadas, conversões e joins complexos devem ser testados pelo plano.

## Identificar o tenant

Quando a tabela não possui uma coluna `CON_ID`, a consulta cross-container fornece essa identificação:

```sql
select con_id,
       count(*) order_count,
       sum(amount) total_amount
  from containers(lab_sales.orders)
 group by con_id
 order by con_id;
```

Resolva o nome do container:

```sql
select c.name container_name,
       count(*) order_count,
       sum(o.amount) total_amount
  from containers(lab_sales.orders) o
  join v$containers c on c.con_id = o.con_id
 group by c.name
 order by c.name;
```

Nunca trate `CON_ID` como identificador de negócio permanente entre CDBs. Ele identifica o container dentro daquela CDB e pode mudar após unplug/plug. Para relatórios duráveis, associe-o ao nome da PDB ou a uma chave de tenant mantida pela aplicação.

O usuário que consulta precisa de privilégio sobre o objeto e alcance adequado nos containers. Common users podem usar `CONTAINER_DATA` para limitar quais PDBs certas views container-data expõem; esse controle é diferente de um filtro `WHERE` e deve ser considerado no desenho de segurança.

## Testar o efeito do estado da PDB

Feche uma Application PDB, execute novamente a agregação e compare. Apenas PDBs elegíveis e abertas participam.

```sql
alter pluggable database lab_app_pdb2 close immediate;

select con_id, sum(amount)
  from containers(lab_sales.orders)
 group by con_id;

alter pluggable database lab_app_pdb2 open;
```

## CONTAINERS() e Container Map

`CONTAINERS()` distribui a consulta. Container Map acrescenta poda por uma chave de roteamento, evitando acessar PDBs que não podem conter o valor filtrado.

Container Map requer uma tabela de mapa e desenho de Application Container. Um predicado na chave permite ao otimizador escolher apenas PDBs relevantes. Sem mapa, `CONTAINERS()` ainda funciona, porém pode consultar todas as PDBs abertas. A função resolve agregação; o mapa resolve roteamento/poda.

## Validação operacional

Compare três resultados: todas as PDBs abertas, uma PDB fechada e a mesma PDB reaberta. Consulte `V$CONTAINERS` junto do total para registrar participantes. Para volume maior, examine o plano e métricas por serviço; correção do total e eficiência da distribuição são verificações diferentes.

## Referências oficiais

- [SELECT e cláusula CONTAINERS](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/SELECT.html)
- [Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
