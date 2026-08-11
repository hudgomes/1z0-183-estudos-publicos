---
title: Consultas cross-container com CONTAINERS()
description: Relatórios agregados entre Application PDBs a partir do Application Root
---

# Consultas cross-container com CONTAINERS()

## Como funciona

`CONTAINERS()` consulta um objeto comum no root e nas PDBs abertas subordinadas. Em um Application Root, isso permite gerar um relatório único sobre todas as Application PDBs.

O objeto deve existir no root e nas PDBs. Um objeto `SHARING=METADATA` é um caso típico: a definição é comum e cada tenant mantém seus próprios dados.

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

## Test case — agregado global

Conectado ao Application Root como o application common user proprietário:

```sql
select sum(amount) total_amount
  from containers(lab_sales.orders);
```

O resultado combina linhas do root e das Application PDBs abertas, exceto containers em modo restrito.

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

## Referências oficiais

- [SELECT e cláusula CONTAINERS](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/SELECT.html)
- [Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
