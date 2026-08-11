---
title: CREATE PLUGGABLE DATABASE a partir de PDB$SEED
description: Criação, estado inicial, administrador local e primeira abertura de uma PDB
---

# CREATE PLUGGABLE DATABASE a partir de PDB$SEED

## Como funciona

Sem uma cláusula `FROM`, `CREATE PLUGGABLE DATABASE` usa `PDB$SEED` como origem. A nova PDB nasce em `MOUNTED`, com status `NEW`, e precisa ser aberta em `READ WRITE` para concluir a integração.

O usuário informado em `ADMIN USER` é local à nova PDB e recebe o papel `PDB_DBA`. Ele não recebe automaticamente privilégios administrativos como `SYSDBA`.

## Pré-requisitos

Confira o root, a seed e a estratégia de arquivos:

```sql
select sys_context('USERENV','CON_NAME') container_name from dual;

select name, open_mode
  from v$pdbs
 order by con_id;

select name, value
  from v$parameter
 where name in ('db_create_file_dest','pdb_file_name_convert');
```

Com OMF ativo, o Oracle escolhe os nomes e destinos. Sem OMF e sem `PDB_FILE_NAME_CONVERT`, use `FILE_NAME_CONVERT`.

## Test case — criar e abrir uma PDB

No `CDB$ROOT`, em ambiente descartável:

```sql
create pluggable database lab_seed_pdb
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";
```

Antes da primeira abertura:

```sql
select p.name, p.open_mode, d.status
  from v$pdbs p
  join cdb_pdbs d on d.pdb_id = p.con_id
 where p.name = 'LAB_SEED_PDB';
```

Abra e persista o estado:

```sql
alter pluggable database lab_seed_pdb open read write;
alter pluggable database lab_seed_pdb save state;
```

Valide o administrador local:

```sql
alter session set container=lab_seed_pdb;

select username, common, account_status
  from dba_users
 where username = 'LAB_PDB_ADMIN';

select grantee, granted_role
  from dba_role_privs
 where grantee = 'LAB_PDB_ADMIN';
```

## Sem OMF

```sql
create pluggable database lab_seed_pdb
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>"
  file_name_convert = (
    '/u02/oradata/CDBORCL/pdbseed/',
    '/u02/oradata/CDBORCL/lab_seed_pdb/'
  );
```

Use os caminhos reais obtidos nas views; não copie paths de outro ambiente.

## Limpeza

```sql
alter session set container=cdb$root;
alter pluggable database lab_seed_pdb close immediate;
drop pluggable database lab_seed_pdb including datafiles;
```

## Referências oficiais

- [Creating a PDB from the Seed](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-a-pdb-from-scratch.html)
- [CREATE PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/CREATE-PLUGGABLE-DATABASE.html)
