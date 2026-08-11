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

## O que realmente é removido

Uma PDB possui registro no control file e datafiles no storage. `DROP PLUGGABLE DATABASE` remove o registro lógico da CDB. A cláusula escolhe se os datafiles associados também serão apagados. Objetos, usuários e dados dentro da PDB deixam de ser acessíveis assim que o registro é removido, mesmo quando os arquivos são preservados.

`INCLUDING DATAFILES` solicita ao Oracle a exclusão dos arquivos conhecidos. Com OMF/ASM, essa é a forma correta de deixar o Oracle administrar os nomes. `KEEP DATAFILES` mantém os arquivos para transporte ou tratamento posterior, mas eles deixam de ser gerenciados pela CDB. Preservar arquivo não equivale a ter uma PDB recuperável: também são necessários manifesto, compatibilidade e procedimento de plug.

Backup e archived logs não são apagados automaticamente pela remoção da PDB. A política de retenção e o RMAN tratam esses artefatos separadamente.

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

A PDB deve estar fechada/mounted ou já unplugged conforme o estado permitido para a operação. Antes do drop, confirme o container atual e o nome exato em `V$PDBS`. Executar no root errado ou confundir nomes parecidos é mais perigoso que a própria sintaxe.

Confirme que os metadados desapareceram:

```sql
select name
  from v$pdbs
 where name = 'LAB_DROP_PDB';
```

No host, valide somente os paths anotados antes do drop. Nunca use uma variável ou glob amplo para excluir arquivos manualmente.

Consulte também `CDB_PDBS` e o alert log. Em storage remoto, ASM ou cloud, a ausência no filesystem local não prova que o arquivo foi removido. O Oracle registra cada ação e eventual falha ao excluir um arquivo.

## Comparação controlada com KEEP DATAFILES

Crie outra PDB descartável, anote os arquivos e execute:

```sql
alter pluggable database lab_keep_pdb close immediate;
drop pluggable database lab_keep_pdb keep datafiles;
```

A PDB sai do control file, mas os datafiles permanecem. Esse fluxo é útil quando outra operação tratará os arquivos depois.

Depois de `KEEP DATAFILES`, não mova ou exclua arquivos até confirmar o próximo passo. Para plugar em outra CDB, gere e preserve o manifesto antes do drop ou use o artefato de unplug apropriado. Se não houver intenção de reutilizar, arquivos órfãos apenas consomem storage e devem ser removidos por procedimento controlado.

## Relação com UNPLUG

```sql
alter pluggable database lab_move_pdb close immediate;
alter pluggable database lab_move_pdb
  unplug into '/tmp/lab_move_pdb.pdb';
```

O arquivo `.pdb` é um artefato comprimido com manifesto e datafiles. `UNPLUG` prepara transporte; não equivale a uma exclusão definitiva.

Unplug em XML gera somente o manifesto e mantém datafiles separados. Unplug em `.pdb` empacota manifesto e datafiles em um único archive. Depois do unplug, a PDB não volta a abrir na mesma CDB sem novo plug; o registro pode ser removido com `DROP ... KEEP DATAFILES` quando os arquivos serão reutilizados.

## Como escolher

| Objetivo | Ação segura |
|---|---|
| Descartar completamente uma PDB de laboratório | Fechar e usar `INCLUDING DATAFILES` |
| Transportar mantendo arquivos separados | Unplug XML e preservar datafiles |
| Transportar como pacote único | Unplug em `.pdb` |
| Retirar registro, mas entregar arquivos a outro processo | `KEEP DATAFILES` após preparar metadados |

Sempre registre datafiles e backup antes de uma operação destrutiva. Não reutilize esse roteiro em uma PDB de produção sem change, janela, backup validado e revisão independente do alvo.

## Referências oficiais

- [Removing a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/removing-a-pdb.html)
- [DROP PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/DROP-PLUGGABLE-DATABASE.html)
