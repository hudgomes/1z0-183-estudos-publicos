---
title: Deploy — DBCA, templates e provisionamento silencioso
description: Pré-requisitos, CDB, OMF, templates, FRA, ARCHIVELOG e character set
---

# Deploy — DBCA, templates e provisionamento silencioso

## Decisões de provisionamento

| Necessidade | Opção |
|---|---|
| Criar CDB em modo silencioso | `-createAsContainerDatabase true` |
| Somente validar pré-requisitos | `-executePrereqs` |
| Criar muitas bases idênticas rapidamente | Template com datafiles (seed template) |
| Gerenciar nomes/localização dos arquivos | OMF / `DB_CREATE_FILE_DEST` |
| Recuperação e archivelogs centralizados | FRA + Enable Archiving |

## Test case 1 — validar sem criar banco

No host Oracle:

```bash
dbca -silent -executePrereqs -databaseConfigType SINGLE
```

Esse modo verifica o ambiente e não cria a base. Se o instalador reclamar que não há listener, configure ou inicie o listener antes de repetir.

## Test case 2 — resposta mínima de uma CDB

Exemplo conceitual; ajuste paths, memória e senhas fora do arquivo:

```bash
dbca -silent -createDatabase \
  -gdbname LABCDB \
  -sid LABCDB \
  -createAsContainerDatabase true \
  -numberOfPDBs 1 \
  -pdbName LABPDB \
  -storageType FS \
  -useOMF true \
  -datafileDestination /u02/app/oracle \
  -recoveryAreaDestination /u03/app/oracle \
  -enableArchive true \
  -characterSet AL32UTF8
```

Nunca coloque a senha real no script versionado; use prompt, wallet ou mecanismo seguro da ferramenta.

## Test case 3 — confirmar o resultado

```sql
select name, cdb, log_mode, open_mode
  from v$database;

select name, open_mode
  from v$pdbs
 order by con_id;

select name, value
  from v$parameter
 where name in ('db_create_file_dest','db_recovery_file_dest');

select value database_charset
  from nls_database_parameters
 where parameter = 'NLS_CHARACTERSET';
```

Com OMF, os arquivos da seed e das novas PDBs ficam sob o destino administrado pelo Oracle. Sem OMF, a criação de PDB precisa de conversão/destino explícito quando não há padrão configurado.

## Templates

- Structure Only: guarda definição, sem copiar datafiles.
- Include Datafiles: seed template; entrega uma cópia pronta e é mais rápido para bases idênticas.

## Character set

Escolha o character set na criação. Alterar o character set do CDB root depois é uma operação restrita e complexa; `AL32UTF8` é a escolha usual para workloads multilíngues e recursos modernos.

## Referências oficiais

- [Creating a CDB with DBCA](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-and-configuring-an-oracle-database.html)
- [DBCA Command Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/dbca-command.html)
- [PDB Storage e OMF](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/overview-of-pdb-creation.html)
