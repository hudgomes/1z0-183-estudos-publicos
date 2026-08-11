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

## Comece pelo tempo, não pelo sintoma

DB Time é a soma do tempo em CPU e do tempo de espera das sessões foreground. Ele pode ser maior que o tempo de relógio porque várias sessões trabalham simultaneamente. Uma investigação consistente delimita o intervalo, mede a carga total e só então desce para SQL, sessões, objetos e arquivos.

AWR armazena estatísticas acumuladas em snapshots. Para interpretar um relatório, compare deltas entre início e fim; o valor absoluto de um contador desde a abertura da instância raramente explica sozinho o período. ASH amostra sessões ativas e ajuda a preservar dimensão temporal, mas uma amostra não representa cada chamada individual.

ADDM consome pares de snapshots e prioriza achados pelo impacto em DB Time. Um achado é evidência para investigação, não autorização automática para aplicar a recomendação. Capacidade de I/O incorreta, janela atípica ou workload incompleto podem distorcer o benefício estimado.

O uso de AWR, ASH e ADDM deve respeitar o licenciamento aplicável ao ambiente. Views dinâmicas e métricas em tempo real não eliminam essa obrigação.

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

ASH inclui `SESSION_STATE`. Quando o estado é `ON CPU`, o campo de evento não deve ser interpretado como espera atual. Para sessões esperando, use `WAIT_CLASS`, `EVENT` e os parâmetros do evento. Agrupar amostras por `SQL_ID`, módulo, serviço e objeto costuma revelar onde o tempo foi concentrado.

## Test case 2 — escolher o relatório

| Necessidade | Script/relatório |
|---|---|
| AWR de um intervalo | `awrrpt.sql` |
| AWR de um SQL específico | `awrsqrpt.sql` |
| Comparar períodos da mesma base | AWR Compare Period |
| Comparar bases diferentes | `awrddrpt.sql` |

Relatório de SQL específico detalha estatísticas e planos de um `SQL_ID`; ele não substitui o AWR do período, que mostra impacto sobre todo o banco. Compare Period normaliza e contrasta dois intervalos, útil para identificar regressão após mudança. Ao comparar bancos diferentes, confirme que duração, carga e configuração são realmente comparáveis.

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

Baseline fixa preserva um intervalo já conhecido, como fechamento mensal. Baseline template agenda intervalos futuros ou repetitivos. A moving window acompanha a retenção e alimenta mecanismos adaptativos. Baseline não é backup de banco nem captura SQL text de forma ilimitada; ela preserva snapshots AWR para comparação e thresholds.

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

Threshold estático compara uma métrica a números definidos. Threshold adaptativo compara o comportamento atual ao histórico equivalente, reduzindo alertas falsos em workloads com sazonalidade. `DBA_THRESHOLDS`, `DBA_OUTSTANDING_ALERTS` e o histórico de alertas ajudam a diferenciar configuração, evento atual e evento já resolvido.

## Ler eventos sem decorar respostas

- `log file sync`: foreground aguardando confirmação de commit; investigue latência de redo e frequência de commits.
- `direct path read`: leitura direta, comum em full scans e index fast full scans.
- Compare DB Time, CPU, I/O e carga antes de concluir que um único evento é a causa.

Eventos indicam onde a sessão passou tempo, mas o mesmo evento pode ter causas diferentes. `log file sync` pode vir de storage lento ou commits excessivamente frequentes. `db file sequential read` costuma refletir leitura de bloco único, mas pode ser normal em acesso seletivo. Sempre correlacione volume, latência, SQL responsável e mudança recente.

## Referências oficiais

- [Automatic Workload Repository](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgdba/awr-report-ui.html)
- [Automatic Database Diagnostic Monitor](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgdba/addm-tab.html)
- [SQL Tuning Advisor e fontes AWR/ASH](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/sql-tuning-advisor.html)
