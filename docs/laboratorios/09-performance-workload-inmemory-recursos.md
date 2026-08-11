---
title: Performance — workload, In-Memory e Resource Manager
description: SPA, Database Replay, In-Memory, HugePages, paralelismo, compressão e limites de recursos
---

# Performance — workload, In-Memory e Resource Manager

## Testar mudança ou reproduzir sistema?

| Ferramenta | Unidade analisada |
|---|---|
| SQL Performance Analyzer | SQL de um SQL Tuning Set, antes e depois |
| Database Replay | Workload capturado com concorrência e timing |

## Test case 1 — sequência do SPA

```text
1. Capturar o workload em um SQL Tuning Set (STS)
2. Executar Pre-Change SQL Trial
3. Aplicar a mudança no sistema de teste
4. Executar Post-Change SQL Trial
5. Comparar os trials e gerar relatório
```

Para Database Replay, o workload capturado precisa ser preprocessado no sistema de teste. `Workload Scale-up` aumenta a carga para avaliar capacidade futura.

## In-Memory em quatro decisões

```sql
select name, value
  from v$parameter
 where name in ('inmemory_size','inmemory_automatic_level','heat_map');
```

- `INMEMORY_SIZE > 0` habilita o column store.
- `INMEMORY_AUTOMATIC_LEVEL=LOW|MEDIUM|HIGH` automatiza população/evicção conforme a versão/licença.
- Quando o desenho usar Heat Map, confirme `HEAT_MAP=ON`.
- `MEMCOMPRESS FOR CAPACITY HIGH` prioriza economia de espaço, com maior custo de CPU.

FastStart usa uma tablespace dedicada para persistir IMCUs e acelerar a repopulação após restart:

```sql
begin
  dbms_inmemory_admin.faststart_enable('IM_FASTSTART_TS');
end;
/
```

## HugePages e gerenciamento de memória

HugePages não é compatível com Automatic Memory Management. Em Linux, use ASMM:

```text
MEMORY_TARGET=0
MEMORY_MAX_TARGET=0
SGA_TARGET=<valor>
SGA_MAX_SIZE=<valor>
PGA_AGGREGATE_TARGET=<valor>
```

Valide no banco:

```sql
select name, value
  from v$parameter
 where name in ('memory_target','memory_max_target','sga_target','sga_max_size');
```

## Resource Manager e paralelismo

- `CPU_COUNT` limita a visão de CPU da instância.
- `RESOURCE_MANAGER_PLAN` ativa um plano que distribui CPU e controla sessões/consultas.
- Resource Manager pode cancelar ou terminar consultas longas de baixa prioridade.
- Auto DOP normalmente exige que a instrução ultrapasse um limiar mínimo de tempo estimado; uma consulta de poucos segundos pode permanecer serial.

## Compressão

Advanced Row Compression reduz armazenamento e pode reduzir I/O em scans. O ganho deve ser medido porque DML e compressão também consomem CPU.

```sql
select table_name, compression, compress_for
  from user_tables
 where table_name = 'LAB_PEDIDOS';
```

## Referências oficiais

- [In-Memory Initialization Parameters](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/init-parameters-for-im-column-store.html)
- [Managing IM FastStart](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/managing-im-faststart-for-im-column-store.html)
- [Automatic In-Memory](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/configuring-memory-management.html)
