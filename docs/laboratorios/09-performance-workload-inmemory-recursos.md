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

SPA mede como cada SQL se comporta antes e depois de uma mudança: tempo, CPU, buffer gets, I/O e plano. Database Replay reproduz chamadas capturadas preservando características de concorrência e timing. O primeiro isola regressões de SQL; o segundo mostra efeitos sistêmicos, como locks, filas e saturação.

Nenhuma ferramenta deve ser executada primeiro em produção com a mudança ainda não validada. O fluxo usa captura representativa, sistema de teste comparável e critérios objetivos. Diferenças de dados, estatísticas, cache e hardware precisam aparecer no relatório para que a comparação seja honesta.

## Test case 1 — sequência do SPA

```text
1. Capturar o workload em um SQL Tuning Set (STS)
2. Executar Pre-Change SQL Trial
3. Aplicar a mudança no sistema de teste
4. Executar Post-Change SQL Trial
5. Comparar os trials e gerar relatório
```

Para Database Replay, o workload capturado precisa ser preprocessado no sistema de teste. `Workload Scale-up` aumenta a carga para avaliar capacidade futura.

No SPA, um SQL Tuning Set preserva SQL text, binds, planos e estatísticas do workload escolhido. Os trials executam o conjunto em condições diferentes e a comparação classifica melhoria, regressão e erro. Uma mudança pode melhorar o total e ainda prejudicar um SQL crítico; examine impacto agregado e individual.

No Database Replay, filtros de captura excluem sessões administrativas ou cargas que não devem ser repetidas. O preprocessamento transforma arquivos capturados para replay. Sincronização, think time e connection scale alteram fidelidade e carga; ajuste consciente, especialmente em scale-up.

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

O Database In-Memory mantém representação colunar na SGA ao lado do formato de linhas nos datafiles e buffer cache. DML continua transacional no row store e as mudanças são refletidas no column store. Marcar uma tabela como `INMEMORY` não garante população imediata: prioridade, espaço, acesso e política automática influenciam quando os IMCUs entram em memória.

Consultas analíticas se beneficiam de varredura colunar, pruning e vector processing; acessos pontuais e DML continuam adequados ao formato de linhas. A escolha de `MEMCOMPRESS` equilibra espaço, velocidade de população e CPU. Verifique população real em `V$IM_SEGMENTS`, não apenas a cláusula do objeto.

FastStart usa uma tablespace dedicada para persistir IMCUs e acelerar a repopulação após restart:

```sql
begin
  dbms_inmemory_admin.faststart_enable('IM_FASTSTART_TS');
end;
/
```

FastStart não transforma In-Memory em armazenamento primário nem elimina datafiles. Ele persiste uma representação para reduzir o tempo de repopulação após restart. A tablespace deve ser dimensionada, monitorada e protegida como parte do desenho operacional.

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

HugePages reduz overhead de page tables e evita swap da SGA quando o sistema está configurado corretamente. Reserve páginas suficientes para a SGA, mantenha memória convencional para PGA e sistema operacional e confirme no alert log se a instância utilizou as páginas. Apenas definir `SGA_TARGET` não configura o kernel.

## Resource Manager e paralelismo

- `CPU_COUNT` limita a visão de CPU da instância.
- `RESOURCE_MANAGER_PLAN` ativa um plano que distribui CPU e controla sessões/consultas.
- Resource Manager pode cancelar ou terminar consultas longas de baixa prioridade.
- Auto DOP normalmente exige que a instrução ultrapasse um limiar mínimo de tempo estimado; uma consulta de poucos segundos pode permanecer serial.

Resource Manager organiza consumer groups e directives. Ele pode distribuir CPU por shares, limitar paralelismo, controlar tempo estimado, desfazer ou cancelar SQL e trocar sessões de grupo. O plano precisa estar ativo e as sessões precisam ser mapeadas corretamente; criar objetos sem ativar `RESOURCE_MANAGER_PLAN` não muda o workload.

`CPU_COUNT` informa capacidade disponível à instância e influencia diversos cálculos. Auto DOP considera custo, política e `PARALLEL_MIN_TIME_THRESHOLD`. Um objeto marcado `PARALLEL` não garante execução paralela se o otimizador e o Resource Manager decidirem o contrário.

## Compressão

Advanced Row Compression reduz armazenamento e pode reduzir I/O em scans. O ganho deve ser medido porque DML e compressão também consomem CPU.

Não confunda compressão de linhas com In-Memory compression. A primeira altera a organização dos blocos persistidos; a segunda controla o formato colunar na memória. HCC possui requisitos de plataforma/storage e perfis próprios para query ou archive.

```sql
select table_name, compression, compress_for
  from user_tables
 where table_name = 'LAB_PEDIDOS';
```

## Referências oficiais

- [In-Memory Initialization Parameters](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/init-parameters-for-im-column-store.html)
- [Managing IM FastStart](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/managing-im-faststart-for-im-column-store.html)
- [Automatic In-Memory](https://docs.oracle.com/en/database/oracle/oracle-database/26/inmem/configuring-memory-management.html)
