---
title: ISPDB_MODIFIABLE — parâmetros por PDB
description: Consulta, alteração, herança e validação de parâmetros em uma PDB
---

# ISPDB_MODIFIABLE — parâmetros por PDB

## Como funciona

`ISPDB_MODIFIABLE` em `V$PARAMETER` informa se um initialization parameter aceita valor específico no nível da PDB.

- `TRUE`: a PDB pode sobrescrever o valor herdado do root.
- `FALSE`: o parâmetro é controlado em outro escopo.

O valor deve ser conferido na versão real do banco. Não deduza a capacidade apenas pelo tipo do parâmetro.

## Herança em duas camadas

Uma PDB começa herdando valores da instância/CDB root. Quando um parâmetro permite escopo por PDB, `ALTER SYSTEM` dentro dela cria uma sobrescrita local. Alterar depois o valor do root não substitui necessariamente a sobrescrita; as PDBs sem valor próprio continuam herdando.

Esse mecanismo permite políticas diferentes para otimizador, memória e outros recursos sem iniciar outra instância. Ainda existe um único conjunto de processos e recursos físicos. Um valor local não pode ultrapassar limites globais ou transformar um parâmetro estático de instância em dinâmico.

Use três propriedades em conjunto:

| Coluna | Significado |
|---|---|
| `ISPDB_MODIFIABLE` | aceita valor específico por PDB |
| `ISSYS_MODIFIABLE` | pode mudar por `ALTER SYSTEM` e em qual condição |
| `ISSES_MODIFIABLE` | pode ser sobrescrito somente para a sessão |

`TRUE` na primeira coluna não garante `SCOPE=MEMORY` imediato. O parâmetro pode exigir reabertura ou restart conforme sua natureza.

## Test case — inspecionar o parâmetro

```sql
select name,
       value,
       ispdb_modifiable,
       issys_modifiable,
       isdefault
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

Na Oracle AI Database 26ai, o parâmetro é modificável por PDB. Compare root e PDB:

```sql
alter session set container=cdb$root;

select con_id, name, value, ispdb_modifiable
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';

alter session set container=certlab;

select con_id, name, value, ispdb_modifiable
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

Execute as consultas em sessões separadas ou registre o container após cada `ALTER SESSION`. `CON_ID` ajuda, mas uma query em `V$PARAMETER` dentro da PDB normalmente mostra a visão efetiva daquele container. Para comparar toda a CDB, use views `CDB_*`/dinâmicas apropriadas a partir do root e privilégios administrativos.

## Aplicar uma sobrescrita local

Dentro da PDB de laboratório:

```sql
alter system set optimizer_use_sql_plan_baselines = false
  scope=both;

select name, value, isdefault
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

Confirme que outra PDB continua com seu próprio valor:

```sql
alter session set container=dev;

select name, value
  from v$parameter
 where name = 'optimizer_use_sql_plan_baselines';
```

`SCOPE=MEMORY` muda a instância corrente até fechamento/restart. `SCOPE=SPFILE` prepara o próximo ciclo. `SCOPE=BOTH` tenta aplicar agora e persistir. Dentro de PDB, a persistência dos parâmetros suportados é administrada pelo mecanismo Multitenant; a sintaxe continua expressando se o valor é atual, persistente ou ambos.

No exemplo, desligar SQL Plan Baselines faz o otimizador daquela PDB deixar de usar os planos gerenciados, mas não exclui as baselines. Reativar o parâmetro permite uso novamente. Portanto, parâmetro controla comportamento; objetos SPM continuam armazenados.

## Remover a sobrescrita

```sql
alter session set container=certlab;

alter system reset optimizer_use_sql_plan_baselines
  scope=spfile;
```

Alguns parâmetros exigem reabertura da PDB ou restart para o valor persistente entrar em vigor. Confira `ISSYS_MODIFIABLE` e a documentação do parâmetro.

Depois do reset e do ciclo exigido, o valor volta a ser herdado do root. Não valide apenas `ISDEFAULT`: diferencie default de fábrica, valor herdado e valor explicitamente definido. Compare root, PDB e configuração persistente.

## Comparar com um parâmetro não modificável

```sql
select name, value, ispdb_modifiable
  from v$parameter
 where name in (
   'optimizer_use_sql_plan_baselines',
   'db_block_size',
   'control_files'
 )
 order by name;
```

`DB_BLOCK_SIZE` participa da estrutura física do banco e `CONTROL_FILES` aponta arquivos da instância; não fazem sentido como preferências independentes de uma PDB. Essa diferença ajuda a reconhecer por que alguns parâmetros são globais, enquanto opções de otimizador e limites de recursos podem ter valor local.

## Procedimento seguro

1. confirmar container e valor atual;
2. consultar as três propriedades de modificação;
3. registrar o valor do root e de uma PDB de controle;
4. aplicar mudança reversível somente na PDB de laboratório;
5. abrir nova sessão e validar valor efetivo;
6. resetar, executar o ciclo necessário e confirmar a herança.

Evite testar parâmetros de memória sem folga ou opções não documentadas. Uma mudança local ainda pode afetar estabilidade de toda a instância quando aumenta consumo compartilhado.

## Referências oficiais

- [V$PARAMETER](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/V-PARAMETER.html)
- [OPTIMIZER_USE_SQL_PLAN_BASELINES](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/OPTIMIZER_USE_SQL_PLAN_BASELINES.html)
