---
title: Application Containers — ciclo de vida e compartilhamento
description: Application Root, Application PDB, INSTALL, SYNC, PATCH, UPGRADE e SHARING
---

# Application Containers — ciclo de vida e compartilhamento

## Arquitetura

Um Application Container agrupa uma aplicação comum e várias Application PDBs:

```text
CDB$ROOT
  └─ APP_ROOT                 Application Root
       ├─ APP_PDB1            tenant 1
       └─ APP_PDB2            tenant 2
```

O Application Root guarda a definição mestre versionada. Cada Application PDB sincroniza essa definição e mantém seus dados específicos.

## Tipos de compartilhamento

| `SHARING` | Metadados | Dados |
|---|---|---|
| `METADATA` | Compartilhados | Separados por Application PDB |
| `DATA` | Compartilhados | Centralizados no Application Root |
| `EXTENDED DATA` | Compartilhados | Dados comuns mais dados locais |
| `NONE` | Não compartilhados | Não compartilhados |

## Test case — criar o container

No `CDB$ROOT`, com OMF ativo:

```sql
create pluggable database lab_app_root
  as application container
  admin user lab_app_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

alter pluggable database lab_app_root open;
alter session set container=lab_app_root;
```

Crie duas Application PDBs:

```sql
create pluggable database lab_app_pdb1
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

create pluggable database lab_app_pdb2
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

alter pluggable database all open;
```

## Instalar a versão inicial

No Application Root:

```sql
alter pluggable database application lab_sales_app
  begin install '1.0';

create table lab_produtos (
  id    number primary key,
  nome  varchar2(100),
  preco number(10,2)
) sharing=metadata;

alter pluggable database application lab_sales_app
  end install '1.0';
```

Sincronize cada Application PDB:

```sql
alter session set container=lab_app_pdb1;
alter pluggable database application lab_sales_app sync;

alter session set container=lab_app_pdb2;
alter pluggable database application lab_sales_app sync;
```

## Aplicar patch

No Application Root:

```sql
alter session set container=lab_app_root;

alter pluggable database application lab_sales_app
  begin patch 1 minimum version '1.0';

alter table lab_produtos add categoria varchar2(50);

alter pluggable database application lab_sales_app
  end patch 1;
```

Sincronize novamente as PDBs. O patch existe no root antes de produzir efeito nos tenants.

## Validação

```sql
select app_name, app_version, app_status
  from dba_applications
 order by app_name;

select object_name, object_type, sharing, status
  from dba_objects
 where object_name = 'LAB_PRODUTOS';
```

## Limpeza

Feche e remova primeiro as Application PDBs; depois remova o Application Root, sempre com nomes previamente validados.

## Referências oficiais

- [About Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
- [Creating Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-application-containers1.html)
