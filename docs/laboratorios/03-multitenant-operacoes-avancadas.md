---
title: Operações Multitenant avançadas
description: Container Map, Proxy PDB, upgrade de PDB e diagnóstico de compatibilidade
---

# Operações Multitenant avançadas

## Três mecanismos que não devem ser confundidos

| Necessidade | Recurso correto |
|---|---|
| Podar PDBs por uma chave de partição em consulta do Application Root | Container Map |
| Expor dados de uma PDB remota sem mover seus datafiles | Proxy PDB |
| Executar atualização clássica de uma PDB conectada a release superior | Abrir em `UPGRADE`/`MIGRATE` |

## Test case 1 — reconhecer Container Map

Uma Container Map associa valores de uma coluna a Application PDBs. O otimizador usa o filtro para acessar apenas os containers relevantes.

No Application Root, inspecione os objetos compartilhados:

```sql
select owner, table_name, sharing
  from dba_tables
 where sharing in ('METADATA LINK','DATA LINK','EXTENDED DATA LINK')
 order by owner, table_name;

select name, open_mode, application_pdb
  from v$containers
 order by con_id;
```

Regra de prova: `CONTAINERS()` agrega linhas entre PDBs; **Container Map** adiciona poda por chave e evita consultar PDBs irrelevantes.

## Test case 2 — anatomia de uma Proxy PDB

Uma Proxy PDB guarda metadados locais e referencia uma PDB remota por database link:

```sql
create database link remote_cdb_link
  connect to remote_clone_user identified by "<SENHA_FORTE_EXCLUSIVA>"
  using 'REMOTE_PDB_SERVICE';

create pluggable database proxy_sales
  as proxy from remote_sales@remote_cdb_link;
```

Valide no destino:

```sql
select pdb_name, status
  from cdb_pdbs
 where pdb_name = 'PROXY_SALES';
```

Não confunda Proxy PDB com relocação: proxy mantém os dados remotos; relocação move a PDB.

## Test case 3 — PDB conectada após mudança de release

Após o plug, consulte violações antes de abrir para usuários:

```sql
select name, type, status, message
  from pdb_plug_in_violations
 where name = 'LAB_UPGRADE'
 order by time;
```

Quando o upgrade clássico for necessário:

```sql
alter pluggable database lab_upgrade open upgrade;
```

`OPEN UPGRADE` também aparece como modo `MIGRATE`. Em releases atuais, Replay Upgrade ou AutoUpgrade pode automatizar o fluxo; a pergunta deve deixar claro quando exige o utilitário clássico.

## Checklist de diagnóstico

```sql
select name, open_mode, restricted
  from v$pdbs
 order by con_id;

select property_name, property_value
  from database_properties
 where property_name in ('UPGRADE_PDB_ON_OPEN','PDB_UPGRADE_SYNC');
```

## Referências oficiais

- [Application Containers e Container Map](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
- [Administering PDBs e Replay Upgrade](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/administering-pdbs-with-sql-plus.html)
- [ALTER PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/ALTER-PLUGGABLE-DATABASE.html)
