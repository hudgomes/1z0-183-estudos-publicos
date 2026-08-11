---
title: DROP PLUGGABLE DATABASE e tratamento dos datafiles
description: INCLUDING DATAFILES, KEEP DATAFILES, UNPLUG e validação física
---

# DROP PLUGGABLE DATABASE e tratamento dos datafiles

## Diferença entre as operações

| Operação | Metadados da CDB | Datafiles |
|---|---|---|
| `DROP ... INCLUDING DATAFILES` | Remove | Exclui fisicamente |
| `DROP ... KEEP DATAFILES` | Remove | Mantém no storage |
| `UNPLUG INTO` | Prepara transporte | Mantém ou empacota |

`INCLUDING DATAFILES` é destrutivo. Use somente com uma PDB criada especificamente para o laboratório.

## Test case — criar, localizar e remover

Com OMF no `CDB$ROOT`:

```sql
create pluggable database lab_drop_pdb
  admin user lab_drop_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

alter pluggable database lab_drop_pdb open;
```

Registre os arquivos antes da remoção:

```sql
select c.name container_name, d.file_id, d.file_name
  from v$containers c
  join cdb_data_files d on d.con_id = c.con_id
 where c.name = 'LAB_DROP_PDB'
 order by d.file_id;
```

Feche e remova:

```sql
alter pluggable database lab_drop_pdb close immediate;
drop pluggable database lab_drop_pdb including datafiles;
```

Confirme que os metadados desapareceram:

```sql
select name
  from v$pdbs
 where name = 'LAB_DROP_PDB';
```

No host, valide somente os paths anotados antes do drop. Nunca use uma variável ou glob amplo para excluir arquivos manualmente.

## Comparação controlada com KEEP DATAFILES

Crie outra PDB descartável, anote os arquivos e execute:

```sql
alter pluggable database lab_keep_pdb close immediate;
drop pluggable database lab_keep_pdb keep datafiles;
```

A PDB sai do control file, mas os datafiles permanecem. Esse fluxo é útil quando outra operação tratará os arquivos depois.

## Relação com UNPLUG

```sql
alter pluggable database lab_move_pdb close immediate;
alter pluggable database lab_move_pdb
  unplug into '/tmp/lab_move_pdb.pdb';
```

O arquivo `.pdb` é um artefato comprimido com manifesto e datafiles. `UNPLUG` prepara transporte; não equivale a uma exclusão definitiva.

## Referências oficiais

- [Removing a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/removing-a-pdb.html)
- [DROP PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/DROP-PLUGGABLE-DATABASE.html)
