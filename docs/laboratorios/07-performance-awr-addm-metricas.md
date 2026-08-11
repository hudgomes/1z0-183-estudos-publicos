---
title: Performance — AWR, ADDM, ASH e métricas
description: Snapshots, baselines, relatórios, eventos de espera, alertas e diagnóstico histórico
---

# Performance — AWR, ADDM, ASH e métricas

## Fluxo de diagnóstico

```text
Métrica atual -> V$SYSMETRIC
Sessões ativas -> V$ACTIVE_SESSION_HISTORY
Histórico -> DBA_HIST_* / AWR
Análise automática -> ADDM
Período importante -> AWR Baseline
Comparação -> AWR Compare Period Report
```

ADDM analisa os snapshots do AWR e depende de uma estimativa realista da capacidade de I/O. O parâmetro `DBIO_EXPECTED`, configurado na tarefa ADDM, representa o tempo médio de leitura do storage.

## Test case 1 — snapshots e ASH

```sql
select snap_id, begin_interval_time, end_interval_time
  from dba_hist_snapshot
 order by snap_id desc
 fetch first 5 rows only;

select sample_time, session_id, sql_id, event, wait_class
  from v$active_session_history
 order by sample_time desc
 fetch first 20 rows only;
```

`DBA_HIST_ACTIVE_SESS_HISTORY` guarda amostras históricas. Para eventos e SQL agregados, use `DBA_HIST_SYSTEM_EVENT` e `DBA_HIST_SQLSTAT`.

## Test case 2 — escolher o relatório

| Necessidade | Script/relatório |
|---|---|
| AWR de um intervalo | `awrrpt.sql` |
| AWR de um SQL específico | `awrsqrpt.sql` |
| Comparar períodos da mesma base | AWR Compare Period |
| Comparar bases diferentes | `awrddrpt.sql` |

No SQLcl/SQL*Plus como usuário autorizado:

```text
@?/rdbms/admin/awrrpt.sql
@?/rdbms/admin/awrsqrpt.sql
@?/rdbms/admin/awrddrpt.sql
```

## Test case 3 — baseline

```sql
begin
  dbms_workload_repository.create_baseline(
    start_snap_id => <SNAP_INICIAL>,
    end_snap_id   => <SNAP_FINAL>,
    baseline_name => 'LAB_DAYTIME');
end;
/
```

Uma baseline preserva os snapshots associados. A janela da `SYSTEM_MOVING_WINDOW` não pode exceder a retenção atual do AWR.

Para horários futuros ou repetitivos, use `CREATE_BASELINE_TEMPLATE`; templates repetitivos usam, entre outros, `day_of_week` e `hour_in_day`.

## Test case 4 — métricas e alertas

```sql
select metric_name, value, metric_unit
  from v$sysmetric
 where group_id = 2
 order by metric_name;

select reason, suggested_action, creation_time
  from dba_outstanding_alerts
 order by creation_time desc;
```

`DBMS_SERVER_ALERT.SET_THRESHOLD` define warning, critical e período de observação. Para voltar ao padrão, informe `NULL` nos valores de warning e critical.

Adaptive thresholds usam uma baseline móvel e níveis de significância, como percentis, para identificar comportamento anormal.

## Ler eventos sem decorar respostas

- `log file sync`: foreground aguardando confirmação de commit; investigue latência de redo e frequência de commits.
- `direct path read`: leitura direta, comum em full scans e index fast full scans.
- Compare DB Time, CPU, I/O e carga antes de concluir que um único evento é a causa.

## Referências oficiais

- [Automatic Workload Repository](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgdba/awr-report-ui.html)
- [Automatic Database Diagnostic Monitor](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgdba/addm-tab.html)
- [SQL Tuning Advisor e fontes AWR/ASH](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/sql-tuning-advisor.html)
