---
title: Data Guard per PDB — source e target CDB
description: Proteção de uma PDB entre CDBs independentes com Data Guard Broker
---

# Data Guard per PDB — source e target CDB

## Arquitetura

Data Guard per PDB protege uma PDB individual entre duas CDBs que continuam operando como databases primárias independentes.

```text
SOURCE CDB (PRIMARY)             TARGET CDB (PRIMARY)
  └─ SALES (source PDB)           └─ DGPDB_SALES (target PDB)
         primary role                    standby role
```

O papel de standby pertence à target PDB. A target CDB não precisa ser uma physical standby da source CDB e pode hospedar outras PDBs.

## Pré-requisitos

- Oracle Database release e RU compatíveis com DGPDB.
- `ARCHIVELOG`, FORCE LOGGING e Data Guard Broker configurados.
- Conectividade e serviços estáticos/dinâmicos testados.
- Source e target CDB registradas em configurações Broker.
- Destinos dos datafiles e credenciais protegidos.
- Backup e plano de rollback antes da configuração.

## Test case guiado — Broker

Crie e valide a configuração da origem:

```text
DGMGRL> CONNECT sys@source_cdb
DGMGRL> CREATE CONFIGURATION SourceConfig
         AS PRIMARY DATABASE IS source_cdb
         CONNECT IDENTIFIER IS source_cdb;
DGMGRL> SHOW CONFIGURATION;
```

Em outra sessão, crie a configuração da target CDB:

```text
DGMGRL> CONNECT sys@target_cdb
DGMGRL> CREATE CONFIGURATION TargetConfig
         AS PRIMARY DATABASE IS target_cdb
         CONNECT IDENTIFIER IS target_cdb;
DGMGRL> SHOW CONFIGURATION;
```

Associe as configurações:

```text
DGMGRL> CONNECT sys@source_cdb
DGMGRL> ADD CONFIGURATION TargetConfig
         CONNECT IDENTIFIER IS target_cdb;
```

Adicione a target PDB, ajustando os paths reais:

```text
DGMGRL> ADD PLUGGABLE DATABASE 'dgpdb_sales' AT 'target_cdb'
         SOURCE IS 'sales' AT 'source_cdb'
         PDBFileNameConvert IS
         '/oradata/source/sales,/oradata/target/dgpdb_sales';
```

Depois da instanciação/cópia dos arquivos:

```text
DGMGRL> ENABLE CONFIGURATION ALL;
```

## Validação

```text
DGMGRL> SHOW DATABASE source_cdb;
DGMGRL> SHOW DATABASE target_cdb;
DGMGRL> SHOW PLUGGABLE DATABASE sales AT source_cdb;
DGMGRL> SHOW PLUGGABLE DATABASE dgpdb_sales AT target_cdb;
```

No SQL, confirme cada contexto:

```sql
select name, database_role, open_mode from v$database;
select name, open_mode from v$pdbs order by con_id;
```

Uma troca planejada é coordenada pelo Broker:

```text
DGMGRL> SWITCHOVER TO PLUGGABLE DATABASE
         dgpdb_sales AT target_cdb;
```

Faça esse passo somente após validar saúde, lag, conectividade, serviços e procedimento de retorno.

## Referências oficiais

- [Data Guard Broker Concepts](https://docs.oracle.com/en/database/oracle/oracle-database/26/dgbkr/oracle-data-guard-broker-concepts.html)
- [Data Guard per PDB with DGMGRL](https://docs.oracle.com/en/database/oracle/oracle-database/26/dgbkr/using-data-guard-broker-to-manage-data-guard-pdb-configurations.html)
