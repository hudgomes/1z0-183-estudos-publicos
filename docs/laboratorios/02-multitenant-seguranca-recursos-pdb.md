---
title: Segurança, privilégios e recursos por PDB
description: Usuários comuns e locais, CONTAINER, auditoria, memória, estado restrito e keystore isolado
---

# Segurança, privilégios e recursos por PDB

## Modelo mental

| Ação | Escopo |
|---|---|
| Grant no root sem cláusula | `CONTAINER=CURRENT`: somente o root |
| Grant no root com `CONTAINER=ALL` | Root e PDBs atuais/futuras |
| Grant dentro de uma PDB | Privilégio local naquela PDB |
| Usuário criado no Application Root com `CONTAINER=ALL` | Application Common User |
| `ALTER SESSION SET CONTAINER` | Exige `SET CONTAINER` ou privilégio administrativo compatível |

Um usuário comum pode receber um privilégio local. Por exemplo, conectar em `SALES_PDB` e executar `GRANT DBA TO C##ADMIN` concede o papel apenas naquele container.

## Test case 1 — comparar grant comum e local

No `CDB$ROOT`, como `SYS`:

```sql
create user c##cert_reader identified by "<SENHA_FORTE_EXCLUSIVA>"
  container=all;
grant create session to c##cert_reader container=all;
grant create table to c##cert_reader container=current;
```

Confirme o escopo:

```sql
select grantee, privilege, common, inherited
  from cdb_sys_privs
 where grantee = 'C##CERT_READER'
 order by con_id, privilege;
```

Agora conceda um privilégio apenas na PDB de laboratório:

```sql
alter session set container=certlab;
grant create table to c##cert_reader;

select grantee, privilege, common, inherited
  from dba_sys_privs
 where grantee = 'C##CERT_READER';
```

## Test case 2 — estado restrito persistente

No root:

```sql
alter pluggable database certlab close immediate;
alter pluggable database certlab open read write restricted;
alter pluggable database certlab save state;

select name, open_mode, restricted
  from v$pdbs
 where name = 'CERTLAB';
```

Somente sessões com `RESTRICTED SESSION` conseguem conectar. Para voltar:

```sql
alter pluggable database certlab close immediate;
alter pluggable database certlab open read write;
alter pluggable database certlab save state;
```

## Test case 3 — limites de memória por PDB

Com Local Undo, entre em `CERTLAB` e verifique se os parâmetros são modificáveis:

```sql
alter session set container=certlab;

select name, value, ispdb_modifiable
  from v$parameter
 where name in ('sga_target','pga_aggregate_limit');
```

Em ambiente com memória disponível:

```sql
alter system set sga_target = 512m scope=both;
alter system set pga_aggregate_limit = 1g scope=both;
```

O CDB root continua impondo o limite físico global. Não dimensione PDBs sem comparar a soma das metas com a memória do host.

## Auditoria e TDE

- Política local criada no root audita somente o root.
- Política comum habilitada no root pode valer em todos os PDBs.
- Um keystore em modo `ISOLATED` dá à PDB uma chave mestra independente.

Inspeção segura:

```sql
select con_id, wrl_type, status, wallet_type, keystore_mode
  from v$encryption_wallet
 order by con_id;
```

## Limpeza

```sql
alter session set container=cdb$root;
drop user c##cert_reader cascade;
```

## Referências oficiais

- [Configuring Privilege and Role Authorization](https://docs.oracle.com/en/database/oracle/oracle-database/26/dbseg/configuring-privilege-and-role-authorization.html)
- [Administering PDBs](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/administering-pdbs-with-sql-plus.html)
- [AUDIT — Unified Auditing](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/AUDIT-Unified-Auditing.html)
