---
title: ISPDB_MODIFIABLE — parâmetros por PDB
description: Consulta, alteração, herança e validação de parâmetros em uma PDB
---

# ISPDB_MODIFIABLE — parâmetros por PDB

## Como funciona

`ISPDB_MODIFIABLE` em `V$PARAMETER` informa se um initialization parameter aceita valor específico no nível da PDB.

- `TRUE`: a PDB pode sobrescrever o valor herdado do root.
- `FALSE`: o parâmetro é controlado em outro escopo.

O valor deve ser conferido na versão real do banco. Não deduza a capacidade apenas pelo tipo do parâmetro.

## Test case — inspecionar o parâmetro

```sql
select name,
       value,
       ispdb_modifiable,
       issys_modifiable,
       isdefault
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

Na Oracle AI Database 26ai, o parâmetro é modificável por PDB. Compare root e PDB:

```sql
alter session set container=cdb$root;

select con_id, name, value, ispdb_modifiable
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';

alter session set container=certlab;

select con_id, name, value, ispdb_modifiable
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

## Aplicar uma sobrescrita local

Dentro da PDB de laboratório:

```sql
alter system set optimizer_use_sql_plan_baselines = false
  scope=both;

select name, value, isdefault
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

Confirme que outra PDB continua com seu próprio valor:

```sql
alter session set container=dev;

select name, value
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

## Remover a sobrescrita

```sql
alter session set container=certlab;

alter system reset optimizer_use_sql_plan_baselines
  scope=spfile;
```

Alguns parâmetros exigem reabertura da PDB ou restart para o valor persistente entrar em vigor. Confira `ISSYS_MODIFIABLE` e a documentação do parâmetro.

## Comparar com um parâmetro não modificável

```sql
select name, value, ispdb_modifiable
  from v$parameter
 where name in (
   'optimizer_use_sql_plan_baselines',
   'db_block_size',
   'control_files'
 )
 order by name;
```

## Referências oficiais

- [V$PARAMETER](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/V-PARAMETER.html)
- [OPTIMIZER_USE_SQL_PLAN_BASELINES](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/OPTIMIZER_USE_SQL_PLAN_BASELINES.html)
