---
title: RMAN — estratégia, retenção e desempenho
description: Retention policy, FRA, block change tracking, image copies, canais e multisection
---

# RMAN — estratégia, retenção e desempenho

## Conceitos essenciais

| Objetivo | Recurso |
|---|---|
| Manter capacidade de recuperar os últimos N dias | `RECOVERY WINDOW OF N DAYS` |
| Acelerar incrementais | Block Change Tracking |
| Reduzir RTO | Incrementally Updated Image Copies |
| Paralelizar um datafile grande | `SECTION SIZE` |
| Usar vários dispositivos/canais | `PARALLELISM` |
| Controlar arquivos abertos por backup set | `MAXOPENFILES` |
| Liberar archivelogs com segurança | Archive deletion policy |

## Test case 1 — observar antes de configurar

No RMAN:

```text
SHOW ALL;
REPORT OBSOLETE;
LIST BACKUP SUMMARY;
```

No SQL:

```sql
select log_mode from v$database;
select name, space_limit, space_used, space_reclaimable
  from v$recovery_file_dest;
select status, filename, bytes
  from v$block_change_tracking;
```

## Test case 2 — retenção e manutenção

```text
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 14 DAYS;
CONFIGURE BACKUP OPTIMIZATION ON;

CROSSCHECK BACKUP;
DELETE EXPIRED BACKUP;
DELETE NOPROMPT OBSOLETE;
```

`CROSSCHECK` não apaga o backup físico. Ele compara repositório e mídia e marca como `EXPIRED` o que não foi encontrado. `DELETE EXPIRED` remove esses registros do repositório.

Para desfazer uma configuração persistente:

```text
CONFIGURE BACKUP OPTIMIZATION CLEAR;
```

## Test case 3 — incremental e image copy

```text
CONFIGURE DEVICE TYPE DISK PARALLELISM 4;

BACKUP INCREMENTAL LEVEL 0 DATABASE TAG 'LAB_L0';
BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE TAG 'LAB_L1C';
```

- Diferencial: blocos alterados desde o level 0 ou level 1 mais recente.
- Cumulativo: blocos alterados desde o level 0; restauração mais simples, backup maior.

Padrão de image copy atualizada:

```text
RUN {
  RECOVER COPY OF DATABASE WITH TAG 'LAB_IU' UNTIL TIME 'SYSDATE-1';
  BACKUP INCREMENTAL LEVEL 1 FOR RECOVER OF COPY
    WITH TAG 'LAB_IU' DATABASE;
}
```

Em uma falha, `SWITCH DATAFILE ... TO COPY` reduz o tempo até a abertura.

## Test case 4 — paralelismo e multisection

```text
CONFIGURE DEVICE TYPE DISK PARALLELISM 4;
BACKUP SECTION SIZE 2G DATABASE;
```

`SECTION SIZE` permite que canais diferentes trabalhem no mesmo datafile grande. Para tape, `BACKUP_TAPE_IO_SLAVES=TRUE` usa memória da Large Pool quando disponível.

## FRA e archivelogs

```text
CONFIGURE ARCHIVELOG DELETION POLICY
  TO BACKED UP 2 TIMES TO DEVICE TYPE SBT;
```

Guaranteed Restore Points, backups e archivelogs não elegíveis à exclusão podem encher a FRA. Monitore antes que alcance 100%.

## Referências oficiais

- [Backup and Recovery User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/toc.htm)
- [Validating Database Files and Backups](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/validating-database-files-backups.html)
- [Backup and Recovery Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/rcmrf/oracle-ai-database-backup-and-recovery-reference.pdf)
