---
title: Relocação online de PDB com indisponibilidade mínima
description: Cópia online, database link, cutover e validação da PDB no destino
---

# Relocação online de PDB com indisponibilidade mínima

## Como funciona

PDB Relocation move uma PDB para outra CDB por meio de um database link. A origem pode permanecer em `READ WRITE` durante a cópia. O cutover ocorre quando a PDB de destino é aberta.

O fluxo copia data blocks, undo e redo, aplica as mudanças restantes e integra a PDB ao destino. Ele é diferente de cold clone e de unplug/plug, que exigem uma janela maior com a origem fechada.

## Pré-requisitos

- CDBs compatíveis e com Local Undo.
- Conectividade Oracle Net entre origem e destino.
- Usuário comum de clone com privilégios apropriados.
- Nome de serviço e database link validados.
- Espaço e destino de arquivos definidos; OMF simplifica a criação.
- Serviços da PDB planejados para o cutover.

## Test case — preparar a origem

No root da origem:

```sql
create user c##lab_clone identified by "<SENHA_FORTE_EXCLUSIVA>"
  container=all;

grant create session, create pluggable database
  to c##lab_clone container=all;

select name, open_mode
  from v$pdbs
 where name = 'LAB_SOURCE_PDB';
```

A origem deve permanecer `READ WRITE` durante a fase de cópia.

## Preparar o destino

No root da CDB de destino:

```sql
create database link source_cdb_link
  connect to c##lab_clone identified by "<SENHA_FORTE_EXCLUSIVA>"
  using 'SOURCE_CDB_SERVICE';

select * from dual@source_cdb_link;
```

Só continue depois que a consulta pelo link funcionar.

## Executar a relocação

```sql
create pluggable database lab_target_pdb
  from lab_source_pdb@source_cdb_link
  relocate availability normal;
```

Durante a cópia, acompanhe:

```sql
select opname, sofar, totalwork, units, elapsed_seconds
  from v$session_longops
 where totalwork > 0
   and sofar < totalwork
 order by start_time;
```

Conclua o cutover:

```sql
alter pluggable database lab_target_pdb open read write;
alter pluggable database lab_target_pdb save state;
```

## Validação

No destino:

```sql
select name, open_mode, restricted
  from v$pdbs
 where name = 'LAB_TARGET_PDB';

select pdb_name, status
  from cdb_pdbs
 where pdb_name = 'LAB_TARGET_PDB';
```

Valide também serviços, conexões da aplicação, objetos inválidos, alert log e backup da nova PDB.

## AVAILABILITY NORMAL e MAX

`NORMAL` é usado quando os listeners pertencem a uma rede comum. `MAX` atende topologias com redes de listener isoladas e exige configuração compatível. Escolha pelo desenho de rede, não apenas pelo nome da opção.

## Referências oficiais

- [Relocating a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/relocating-a-pdb.html)
- [CREATE PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/CREATE-PLUGGABLE-DATABASE.html)
