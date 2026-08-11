---
title: RMAN — validação, corrupção e recuperação
description: VALIDATE, Data Recovery Advisor, block recovery, restore, control file, PDB e incarnations
---

# RMAN — validação, corrupção e recuperação

## Escolha o comando pelo objetivo

| Objetivo | Comando |
|---|---|
| Ler datafiles e detectar corrupção | `VALIDATE DATABASE` ou `BACKUP VALIDATE` |
| Confirmar que backups podem ser restaurados | `RESTORE DATABASE VALIDATE` |
| Diagnosticar falha conhecida | `LIST FAILURE`, `ADVISE FAILURE` |
| Ver o que o reparo fará | `REPAIR FAILURE PREVIEW` |
| Corrigir blocos isolados | `RECOVER ... BLOCK` |
| Recuperar uma PDB | Fechar PDB, `RESTORE` e `RECOVER PLUGGABLE DATABASE` |

## Test case 1 — validação sem gerar backup

No RMAN:

```text
VALIDATE DATABASE CHECK LOGICAL;
RESTORE DATABASE VALIDATE;
```

Depois, no SQL:

```sql
select file#, block#, blocks, corruption_type
  from v$database_block_corruption
 order by file#, block#;
```

A view é atualizada por leituras que detectam corrupção, como RMAN validation/backup e alguns full scans.

Para validar apenas os arquivos do tablespace:

```text
VALIDATE TABLESPACE USERS CHECK LOGICAL;
```

## Test case 2 — fluxo do Data Recovery Advisor

```text
LIST FAILURE;
ADVISE FAILURE;
REPAIR FAILURE PREVIEW;
REPAIR FAILURE;
```

O DRA é apropriado para falhas físicas diagnosticáveis. Corrupções lógicas que exigem Flashback Database não são seu caso típico de reparo automático.

## Test case 3 — recuperação de bloco

```text
RECOVER DATAFILE 7 BLOCK 12345;
```

Durante block media recovery o datafile pode permanecer online; apenas os blocos afetados ficam indisponíveis. É necessário backup utilizável e redo suficiente, normalmente com a base em `ARCHIVELOG`.

## Test case 4 — datafile e PDB

Para um datafile não crítico:

```text
SQL 'ALTER DATABASE DATAFILE 7 OFFLINE';
RESTORE DATAFILE 7;
RECOVER DATAFILE 7;
SQL 'ALTER DATABASE DATAFILE 7 ONLINE';
```

Para uma PDB:

```text
SQL 'ALTER PLUGGABLE DATABASE CERTLAB CLOSE IMMEDIATE';
RESTORE PLUGGABLE DATABASE CERTLAB;
RECOVER PLUGGABLE DATABASE CERTLAB;
SQL 'ALTER PLUGGABLE DATABASE CERTLAB OPEN';
```

## Control file, incarnation e Active Duplicate

- Após restaurar todos os control files, monte, recupere e abra com `RESETLOGS` quando o cenário exigir.
- Para PITR anterior ao último `RESETLOGS`, selecione a incarnation ancestral correta com `RESET DATABASE TO INCARNATION`.
- Active Duplicate exige conexão de rede com a auxiliar; o serviço da auxiliar deve estar disponível mesmo quando ela está `NOMOUNT`, normalmente por registro estático no listener.
- `DB_FILE_NAME_CONVERT` ajuda quando a hierarquia de diretórios difere no destino.

## Segurança antes de recuperar

```text
LIST BACKUP OF DATABASE SUMMARY;
LIST INCARNATION OF DATABASE;
REPORT SCHEMA;
RESTORE DATABASE PREVIEW;
```

Nunca use arquivos reais de `SYSTEM`/`SYSAUX` para provocar falha em um laboratório. Use uma PDB e tablespace descartáveis.

## Referências oficiais

- [Validating Database Files and Backups](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/validating-database-files-backups.html)
- [Backup and Recovery User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/toc.htm)
- [VALIDATE — RMAN Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/rcmrf/VALIDATE.html)
