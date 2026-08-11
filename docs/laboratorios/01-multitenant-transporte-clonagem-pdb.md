---
title: Transporte, plug e clonagem de PDB
description: Compatibilidade, arquivos XML e PDB, COPY, MOVE, OMF, hot clone e snapshot copy
---

# Transporte, plug e clonagem de PDB

## O que você precisa entender

Uma PDB pode ser movida logicamente entre CDBs por `UNPLUG`/`PLUG` ou copiada por clonagem.

| Recurso | O que faz |
|---|---|
| XML | Guarda metadados e caminhos; os datafiles continuam separados |
| Arquivo `.pdb` | Pacote comprimido com XML e datafiles |
| `COPY` | Copia os arquivos; é o comportamento padrão |
| `MOVE` | Move os arquivos e remove a origem após o sucesso |
| `NOCOPY` | Reutiliza os arquivos onde estão |
| Hot clone | Clona uma PDB aberta em `READ WRITE`; exige Local Undo e `ARCHIVELOG` |
| Snapshot copy | Usa arquivos esparsos/snapshot copy-on-write do storage |

Antes do plug, confirme versão, `COMPATIBLE`, character set, opções instaladas e endian format. Um plug direto não converte datafiles entre plataformas com endian diferente.

## Test case 1 — validar e plugar com XML

Execute como `SYS` em um ambiente descartável.

```sql
-- Na CDB de origem
alter pluggable database lab_src close immediate;
alter pluggable database lab_src unplug into '/tmp/lab_src.xml';
```

Copie o XML e os datafiles para o destino. Antes do plug:

```sql
set serveroutput on
declare
  l_ok boolean;
begin
  l_ok := dbms_pdb.check_plug_compatibility(
    pdb_descr_file => '/tmp/lab_src.xml',
    pdb_name       => 'LAB_DST');
  dbms_output.put_line(case when l_ok then 'COMPATIVEL' else 'INCOMPATIVEL' end);
end;
/

select type, message, status
  from pdb_plug_in_violations
 where name = 'LAB_DST'
 order by time;
```

Com OMF ativo, o destino é definido por `DB_CREATE_FILE_DEST`:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.xml'
  copy;

alter pluggable database lab_dst open;
```

Sem OMF e sem `PDB_FILE_NAME_CONVERT`, informe a conversão:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.xml'
  copy
  file_name_convert = ('/origem/lab_src/', '/destino/lab_dst/');
```

## Test case 2 — gerar um único arquivo transportável

```sql
alter pluggable database lab_src close immediate;
alter pluggable database lab_src
  unplug into '/tmp/lab_src.pdb';
```

O `.pdb` contém metadados e datafiles. No destino:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.pdb';
```

## Test case 3 — clone local

Consulte primeiro os pré-requisitos:

```sql
select log_mode from v$database;
select property_value
  from database_properties
 where property_name = 'LOCAL_UNDO_ENABLED';
```

Em `ARCHIVELOG` com Local Undo, a origem pode permanecer aberta:

```sql
create pluggable database lab_clone from certlab;
alter pluggable database lab_clone open;
```

Em `NOARCHIVELOG`, feche a origem e abra-a em `READ ONLY` antes do clone.

## Como validar e limpar

```sql
select name, open_mode, restricted
  from v$pdbs
 where name in ('LAB_SRC','LAB_DST','LAB_CLONE');

alter pluggable database lab_clone close immediate;
drop pluggable database lab_clone including datafiles;
```

## Pontos de prova

- O destino não pode estar em release inferior à origem.
- Endian diferente exige migração/conversão suportada; plug direto não converte blocos.
- `DBMS_PDB.CHECK_PLUG_COMPATIBILITY` valida o manifesto antes do plug.
- Com OMF, o Oracle administra o destino dos arquivos.
- Hot clone: Local Undo + `ARCHIVELOG` + archived logs preservados durante a cópia.
- Snapshot copy depende de suporte a arquivos esparsos ou snapshot no storage.

## Referências oficiais

- [Plugging In an Unplugged PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/plugging-in-a-pdb.html)
- [Cloning a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/cloning-a-pdb.html)
- [Removing a PDB e arquivo `.pdb`](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/removing-a-pdb.html)
