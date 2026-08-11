---
title: Performance — otimizador, SQL Tuning e estabilidade de planos
description: Adaptive plans, feedback, estatísticas pendentes, SQL Patch, SQL Plan Baselines e monitoramento
---

# Performance — otimizador, SQL Tuning e estabilidade de planos

## Qual recurso escolher?

| Problema | Recurso provável |
|---|---|
| Estimativa melhora após primeira execução | Statistics Feedback |
| Plano decide entre caminhos durante a execução | Adaptive Plan |
| Não pode alterar SQL de terceiro | SQL Patch |
| Precisa impedir regressão de plano | SQL Plan Baseline |
| Precisa corrigir estimativas | SQL Profile |
| Expressão impede índice normal | Function-based index |
| Literais causam excesso de hard parse | `CURSOR_SHARING=FORCE` como mitigação |

## Test case 1 — estatísticas pendentes

```sql
exec dbms_stats.set_table_prefs(user, 'LAB_PEDIDOS', 'PUBLISH', 'FALSE');
exec dbms_stats.gather_table_stats(user, 'LAB_PEDIDOS');

alter session set optimizer_use_pending_statistics = true;
explain plan for select * from lab_pedidos where status = 'ABERTO';
select * from table(dbms_xplan.display);
```

Valide antes de publicar. Assim uma coleta ruim não altera todas as sessões imediatamente.

## Test case 2 — function-based index

```sql
create index lab_cliente_nome_upper_ix
    on lab_cliente(upper(nome));

select *
  from lab_cliente
 where upper(nome) = 'ANA';
```

O predicado precisa corresponder à expressão indexada e as estatísticas devem estar atualizadas.

## Test case 3 — SQL Plan Management

```sql
select sql_handle, plan_name, enabled, accepted, fixed
  from dba_sql_plan_baselines
 order by created desc;
```

Um plano novo capturado pode entrar como `ACCEPTED=NO`. Ele só passa a ser candidato aceito após evolução/verificação. Um plano `FIXED=YES` tem precedência sobre planos não fixos.

```sql
select dbms_spm.evolve_sql_plan_baseline(
         sql_handle => '<SQL_HANDLE>') as report
  from dual;
```

## Test case 4 — SQL Tuning Advisor

O advisor pode recomendar estatísticas, reescrita, índice, SQL Profile ou baseline. Ele não altera automaticamente o SQL-fonte da aplicação.

```sql
select client_name, status
  from dba_autotask_client
 where client_name = 'sql tuning advisor';
```

SQL Access Advisor é voltado a estruturas de acesso, especialmente índices B-tree e materialized views.

## Monitoramento e instrumentação

Aplicações devem informar módulo e ação:

```sql
begin
  dbms_application_info.set_module('CERT_LAB', 'CARGA_PEDIDOS');
  dbms_application_info.set_action('AGREGAR');
end;
/
```

Para tracing seletivo:

- `DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE`
- `DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE`

Real-Time SQL Monitoring usa principalmente `V$SQL_MONITOR` e `V$SQL_PLAN_MONITOR`. Uma operação composta pode ser delimitada por `DBMS_SQL_MONITOR.BEGIN_OPERATION`.

## Outros pontos importantes

- PL/SQL Function Result Cache reutiliza o resultado de funções determinísticas no servidor.
- Serviços permitem agregar estatísticas e waits por workload.
- Automatic Indexing em `REPORT ONLY` analisa e relata sem tornar os índices automaticamente visíveis ao workload.

## Referências oficiais

- [Overview of SQL Plan Management](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/overview-of-sql-plan-management.html)
- [Analyzing SQL with SQL Tuning Advisor](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/sql-tuning-advisor.html)
- [Monitoring and Tracing SQL](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/monitoring-and-tracing-sql.html)
