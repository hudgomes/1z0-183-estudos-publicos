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

## Como o plano é formado

O otimizador estima cardinalidade e custo usando estatísticas de tabelas, colunas, índices, sistema e expressões. Um plano ruim pode nascer de estatística ausente, distribuição não uniforme, correlação entre colunas, predicado não sargable, bind sensitivity ou mudança de ambiente. Trocar o plano sem corrigir a causa pode apenas esconder o problema.

Adaptive Plan contém caminhos alternativos no plano compilado e pode escolher durante a execução com base nas linhas observadas. Statistics Feedback ocorre depois de uma execução quando estimativas e realidade divergem; as informações podem influenciar um novo parse. SQL Plan Directive e estatísticas estendidas tratam outros aspectos de estimativa. São mecanismos relacionados, mas não equivalentes.

Antes de intervir, capture `SQL_ID`, child cursor, plano real e estatísticas de execução. `EXPLAIN PLAN` mostra uma previsão e pode diferir do cursor executado. Para o plano real, prefira `DBMS_XPLAN.DISPLAY_CURSOR` com informações de execução quando `STATISTICS_LEVEL` ou hint apropriada coletou as linhas.

## Test case 1 — estatísticas pendentes

```sql
exec dbms_stats.set_table_prefs(user, 'LAB_PEDIDOS', 'PUBLISH', 'FALSE');
exec dbms_stats.gather_table_stats(user, 'LAB_PEDIDOS');

alter session set optimizer_use_pending_statistics = true;
explain plan for select * from lab_pedidos where status = 'ABERTO';
select * from table(dbms_xplan.display);
```

Valide antes de publicar. Assim uma coleta ruim não altera todas as sessões imediatamente.

Finalize explicitamente o ciclo:

```sql
exec dbms_stats.publish_pending_stats(user, 'LAB_PEDIDOS');
-- ou, se o teste for ruim:
exec dbms_stats.delete_pending_stats(user, 'LAB_PEDIDOS');
```

`PUBLISH=FALSE` controla visibilidade da nova coleta. `NO_INVALIDATE` controla quando cursores que usam estatísticas antigas serão invalidados; não cria estatísticas pendentes. Locks de estatísticas impedem coleta convencional, mas também não fornecem automaticamente uma área de teste.

## Test case 2 — function-based index

```sql
create index lab_cliente_nome_upper_ix
    on lab_cliente(upper(nome));

select *
  from lab_cliente
 where upper(nome) = 'ANA';
```

O predicado precisa corresponder à expressão indexada e as estatísticas devem estar atualizadas.

Uma function-based index armazena o resultado da expressão. Diferenças de função, conversão implícita ou tipo podem impedir seu uso. Para expressões condicionais, valores `NULL` não entram em índice B-tree de coluna única, comportamento que pode ser usado para indexar apenas subconjuntos, desde que a lógica seja documentada.

## Test case 3 — SQL Plan Management

```sql
select sql_handle, plan_name, enabled, accepted, fixed
  from dba_sql_plan_baselines
 order by created desc;
```

Um plano novo capturado pode entrar como `ACCEPTED=NO`. Ele só passa a ser candidato aceito após evolução/verificação. Um plano `FIXED=YES` tem precedência sobre planos não fixos.

SQL Plan Baseline guarda planos permitidos e busca estabilidade. SQL Profile fornece fatores auxiliares para melhorar estimativas, mas não fixa diretamente um plano. SQL Patch injeta hints sem alterar o texto da aplicação. A escolha depende do objetivo: corrigir estimativa, limitar planos aceitos ou contornar SQL que não pode ser editado.

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

Uma recomendação do Tuning Advisor inclui benefício estimado e justificativa. SQL Profile pode ser aceito pelo DBA; índices e reescritas exigem avaliação de custo de DML, armazenamento e manutenção. O advisor opera sobre uma instrução ou SQL Tuning Set, enquanto Access Advisor avalia estruturas para um workload mais amplo.

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

Tracing por serviço, módulo, ação ou client identifier é preferível a habilitar trace global. Instrumentação correta permite separar workloads que usam o mesmo schema e localizar a etapa lenta sem coletar volume desnecessário. Remova o trace depois do intervalo e formate o arquivo com ferramenta apropriada antes de concluir.

## Outros pontos importantes

- PL/SQL Function Result Cache reutiliza o resultado de funções determinísticas no servidor.
- Serviços permitem agregar estatísticas e waits por workload.
- Automatic Indexing em `REPORT ONLY` analisa e relata sem tornar os índices automaticamente visíveis ao workload.

Result Cache deve ser reservado a funções cujos resultados dependem de dados rastreáveis e parâmetros estáveis. `CURSOR_SHARING=FORCE` reduz SQLs distintos por literal, mas pode afetar seletividade e não corrige aplicações vulneráveis; bind variables continuam sendo o desenho preferido.

## Referências oficiais

- [Overview of SQL Plan Management](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/overview-of-sql-plan-management.html)
- [Analyzing SQL with SQL Tuning Advisor](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/sql-tuning-advisor.html)
- [Monitoring and Tracing SQL](https://docs.oracle.com/en/database/oracle/oracle-database/26/tgsql/monitoring-and-tracing-sql.html)
