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

## Retenção não é agendamento

A retention policy descreve **por quanto tempo o banco deve continuar recuperável**; ela não executa backups nem remove arquivos sozinha. `RECOVERY WINDOW OF 14 DAYS` exige que o conjunto retido permita voltar a qualquer ponto dentro da janela. `REDUNDANCY 2` preserva uma quantidade de backups completos ou level 0, independentemente de dias.

O RMAN classifica como `OBSOLETE` o backup que deixou de ser necessário para a política atual. `EXPIRED` possui outro significado: o repositório conhece a peça, mas `CROSSCHECK` não a encontrou no disco ou na mídia. Um backup pode estar velho e ainda `AVAILABLE`, ou recente e `EXPIRED` se foi removido fora do RMAN.

O repositório fica no control file e, opcionalmente, em um Recovery Catalog. `CONTROL_FILE_RECORD_KEEP_TIME` influencia por quanto tempo registros reutilizáveis permanecem no control file, mas não substitui a retention policy nem protege fisicamente uma peça de backup.

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

`DELETE OBSOLETE` usa a política para remover peças que já não sustentam a recuperabilidade definida. Antes de automatizá-lo, gere `REPORT OBSOLETE`, valide cópias externas e confirme se há exigências de retenção que o RMAN não conhece.

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

Um backup `FULL` e um `INCREMENTAL LEVEL 0` leem todos os blocos usados, mas somente o level 0 participa como base da estratégia incremental. Block Change Tracking mantém um pequeno arquivo com a localização dos blocos modificados; ele evita varrer todos os datafiles em cada level 1, porém não elimina a necessidade de ler os blocos que realmente serão copiados.

Padrão de image copy atualizada:

```text
RUN {
  RECOVER COPY OF DATABASE WITH TAG 'LAB_IU' UNTIL TIME 'SYSDATE-1';
  BACKUP INCREMENTAL LEVEL 1 FOR RECOVER OF COPY
    WITH TAG 'LAB_IU' DATABASE;
}
```

Em uma falha, `SWITCH DATAFILE ... TO COPY` reduz o tempo até a abertura.

No ciclo de incrementally updated image copies, o level 1 é aplicado à image copy anterior. A cópia avança no tempo e fica próxima do estado atual, diminuindo o trabalho de restore. O deslocamento `SYSDATE-1` preserva uma folga para que exista incremental aplicável em cada execução; a ordem `RECOVER COPY` antes de `BACKUP ... FOR RECOVER OF COPY` é intencional.

## Test case 4 — paralelismo e multisection

```text
CONFIGURE DEVICE TYPE DISK PARALLELISM 4;
BACKUP SECTION SIZE 2G DATABASE;
```

`SECTION SIZE` permite que canais diferentes trabalhem no mesmo datafile grande. Para tape, `BACKUP_TAPE_IO_SLAVES=TRUE` usa memória da Large Pool quando disponível.

Paralelismo só ajuda quando CPU, discos e destino de backup suportam a vazão. Cada canal é uma sessão de servidor com buffers e I/O próprios. Muitos canais sobre o mesmo conjunto de discos podem aumentar contenção. `FILESPERSET`, `MAXOPENFILES`, tamanho de seção e multiplexação controlam como arquivos são distribuídos entre backup sets e devem ser medidos, não maximizados por padrão.

## FRA e archivelogs

```text
CONFIGURE ARCHIVELOG DELETION POLICY
  TO BACKED UP 2 TIMES TO DEVICE TYPE SBT;
```

Guaranteed Restore Points, backups e archivelogs não elegíveis à exclusão podem encher a FRA. Monitore antes que alcance 100%.

A FRA cheia não é resolvida apenas aumentando `DB_RECOVERY_FILE_DEST_SIZE`. Primeiro identifique o consumidor, confirme a política de exclusão, transporte/aplicação no standby e existência de backup dos logs. Arquivos necessários a restore points garantidos ou à recuperação não serão descartados apenas por pressão de espaço.

## Referências oficiais

- [Backup and Recovery User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/toc.htm)
- [Validating Database Files and Backups](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/validating-database-files-backups.html)
- [Backup and Recovery Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/rcmrf/oracle-ai-database-backup-and-recovery-reference.pdf)
