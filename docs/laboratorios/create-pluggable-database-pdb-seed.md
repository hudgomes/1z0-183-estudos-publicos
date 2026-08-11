---
title: CREATE PLUGGABLE DATABASE a partir de PDB$SEED
description: Criação, estado inicial, administrador local e primeira abertura de uma PDB
---

# CREATE PLUGGABLE DATABASE a partir de PDB$SEED

## Como funciona

Sem uma cláusula `FROM`, `CREATE PLUGGABLE DATABASE` usa `PDB$SEED` como origem. A nova PDB nasce em `MOUNTED`, com status `NEW`, e precisa ser aberta em `READ WRITE` para concluir a integração.

O usuário informado em `ADMIN USER` é local à nova PDB e recebe o papel `PDB_DBA`. Ele não recebe automaticamente privilégios administrativos como `SYSDBA`.

## O papel da PDB$SEED

`PDB$SEED` é o modelo mantido pelo Oracle dentro de cada CDB. Ela permanece normalmente em `READ ONLY` e contém o dicionário e os objetos necessários para iniciar uma nova PDB. Criar sem `FROM` equivale a copiar a seed, mas o Oracle executa também a integração, cria o administrador local e registra a nova PDB no control file.

A seed não é um template de aplicação. Objetos específicos de negócio, usuários locais e dados de uma PDB comum não devem ser adicionados diretamente a ela. Para padronização adicional, use uma PDB modelo clonável, Application Container ou automação posterior à criação.

O estado `NEW` indica que a PDB ainda não concluiu sua primeira abertura `READ WRITE`. Nessa abertura o Oracle termina tarefas internas e muda o status para `NORMAL`. Apenas criar e deixar `MOUNTED` não conclui esse ciclo.

## Pré-requisitos

Confira o root, a seed e a estratégia de arquivos:

```sql
select sys_context('USERENV','CON_NAME') container_name from dual;

select name, open_mode
  from v$pdbs
 order by con_id;

select name, value
  from v$parameter
 where name in ('db_create_file_dest','pdb_file_name_convert');
```

Com OMF ativo, o Oracle escolhe os nomes e destinos. Sem OMF e sem `PDB_FILE_NAME_CONVERT`, use `FILE_NAME_CONVERT`.

Também confirme espaço, versão da seed e modo de undo:

```sql
select property_name, property_value
  from database_properties
 where property_name = 'LOCAL_UNDO_ENABLED';

select con_id, file_name, bytes
  from cdb_data_files
 where con_id = 2
 order by file_id;
```

Os arquivos da nova PDB são cópias dos arquivos da seed. Com OMF, `DB_CREATE_FILE_DEST` define a área e o Oracle gera nomes únicos. Em ASM, o destino costuma ser um diskgroup; em filesystem, um diretório administrado. OMF não decide sozinho quota, tamanho de tablespaces ou política de backup.

## Test case — criar e abrir uma PDB

No `CDB$ROOT`, em ambiente descartável:

```sql
create pluggable database lab_seed_pdb
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";
```

Antes da primeira abertura:

```sql
select p.name, p.open_mode, d.status
  from v$pdbs p
  join cdb_pdbs d on d.pdb_id = p.con_id
 where p.name = 'LAB_SEED_PDB';
```

Abra e persista o estado:

```sql
alter pluggable database lab_seed_pdb open read write;
alter pluggable database lab_seed_pdb save state;
```

`OPEN READ WRITE` torna a PDB utilizável e completa o estado inicial. `SAVE STATE` grava como ela deve abrir após restart da CDB. Sem save state, a instância pode reiniciar com a PDB `MOUNTED`, mesmo que ela estivesse aberta antes da parada.

Valide o administrador local:

```sql
alter session set container=lab_seed_pdb;

select username, common, account_status
  from dba_users
 where username = 'LAB_PDB_ADMIN';

select grantee, granted_role
  from dba_role_privs
 where grantee = 'LAB_PDB_ADMIN';
```

`PDB_DBA` fornece administração local conforme os privilégios contidos no role, mas não transforma o usuário em administrador da CDB. Grants comuns, acesso a outras PDBs e privilégios administrativos protegidos por password file continuam separados. Em produção, crie roles funcionais e evite usar o administrador local pela aplicação.

## Serviços e contexto de conexão

Cada PDB possui serviço padrão com o nome da PDB, além de serviços criados para workloads. Confirme o registro:

```sql
select name, network_name, pdb
  from cdb_services
 where pdb = 'LAB_SEED_PDB';
```

Teste uma conexão pelo serviço da PDB e consulte `SYS_CONTEXT('USERENV','CON_NAME')`. Alterar o container em uma sessão administrativa comprova acesso interno, mas não valida listener, resolução de nome nem connect string da aplicação.

## Sem OMF

```sql
create pluggable database lab_seed_pdb
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>"
  file_name_convert = (
    '/u02/oradata/CDBORCL/pdbseed/',
    '/u02/oradata/CDBORCL/lab_seed_pdb/'
  );
```

Use os caminhos reais obtidos nas views; não copie paths de outro ambiente.

`FILE_NAME_CONVERT` substitui partes dos caminhos da seed ao gerar os nomes do destino. A ordem origem/destino importa, e uma expressão ampla pode apontar para diretório incorreto. Verifique se o diretório existe, permissões do owner e ausência de arquivos com o mesmo nome antes de criar.

## Validação final

Além de `V$PDBS`, confirme `CDB_PDBS.STATUS`, datafiles, tablespaces, objetos inválidos e alert log. Faça backup depois de criar e configurar a PDB; a seed permite recriar uma PDB vazia, mas não recupera dados e configurações adicionados posteriormente.

## Limpeza

```sql
alter session set container=cdb$root;
alter pluggable database lab_seed_pdb close immediate;
drop pluggable database lab_seed_pdb including datafiles;
```

## Referências oficiais

- [Creating a PDB from the Seed](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-a-pdb-from-scratch.html)
- [CREATE PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/CREATE-PLUGGABLE-DATABASE.html)
