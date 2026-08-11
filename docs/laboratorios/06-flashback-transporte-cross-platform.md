---
title: Flashback, restore points e transporte cross-platform
description: Flashback Table, Flashback Drop, guaranteed restore points e conversão entre plataformas
---

# Flashback, restore points e transporte cross-platform

## Qual tecnologia resolve cada erro?

| Situação | Recurso |
|---|---|
| `DROP TABLE` acidental | Flashback Drop / recycle bin |
| DML incorreto sem mudança estrutural | Flashback Table |
| Reverter toda a base a um ponto conhecido | Flashback Database / restore point |
| Falha de instância | Instance recovery automática com roll forward e rollback |
| Transportar datafiles entre endian diferentes | RMAN `CONVERT` |

## Test case 1 — Flashback Table

Na PDB de laboratório, conceda ao usuário que executará o teste:

```sql
grant execute on sys.dbms_flashback to studylab;
```

Conectado como `STUDYLAB`:

```sql
create table lab_saldo (
  id    number primary key,
  saldo number not null
);

insert into lab_saldo values (1, 1000);
commit;

alter table lab_saldo enable row movement;

column scn_now new_value SCN_ANTES_DO_ERRO noprint
select dbms_flashback.get_system_change_number scn_now from dual;
```

Anote o SCN, altere o dado e recupere:

```sql
update lab_saldo set saldo = 10 where id = 1;
commit;

flashback table lab_saldo to scn &SCN_ANTES_DO_ERRO;
select * from lab_saldo;
```

Flashback Table depende de undo suficiente e não atravessa DDL estrutural, como `DROP COLUMN`, `TRUNCATE` ou mudança de partição.

## Test case 2 — Flashback Drop

```sql
drop table lab_saldo;

select original_name, object_name, type
  from recyclebin
 where original_name = 'LAB_SALDO';

flashback table lab_saldo to before drop;
```

A tabela, constraints, triggers e a maioria dos índices retornam. Índices recuperados podem manter nomes `BIN$...` gerados pelo recycle bin.

## Test case 3 — guaranteed restore point

Antes de uma manutenção de laboratório:

```sql
create restore point grp_before_lab guarantee flashback database;

select name, guarantee_flashback_database, storage_size
  from v$restore_point;
```

Ao terminar:

```sql
drop restore point grp_before_lab;
```

Um guaranteed restore point não expira automaticamente. Se a FRA não conseguir preservar os flashback logs necessários, a instância pode parar para não quebrar a garantia.

## Instance recovery em uma frase

Após falha de energia, o Oracle reaplica redo (**roll forward**) e depois desfaz transações não confirmadas (**rolling back**). O banco pode abrir antes de todo rollback terminar.

## Test case 4 — transporte cross-platform

Confira o endian:

```sql
select platform_id, platform_name, endian_format
  from v$transportable_platform
 order by platform_name;
```

Quando houver conversão suportada:

```text
CONVERT DATAFILE '/stage/users01.dbf'
  FROM PLATFORM 'Solaris[tm] OE (64-bit)'
  FORMAT '+DATA';
```

Em transporte incremental, compare `CHECKPOINT_CHANGE#` nos headers de origem e destino para saber se a cópia recebeu todos os incrementais.

## Limpeza

```sql
drop table lab_saldo purge;
```

## Referências oficiais

- [Using Flashback Database and Restore Points](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/using-flasback-database-restore-points.html)
- [Performing Flashback and DBPITR](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/rman-performing-flashback-dbpitr.html)
- [FLASHBACK TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/FLASHBACK-TABLE.html)
