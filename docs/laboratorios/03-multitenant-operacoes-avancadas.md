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

Os três recursos atuam em camadas diferentes. Container Map decide **quais Application PDBs participam de uma consulta**. Proxy PDB decide **onde os dados realmente residem**. O modo de upgrade decide **como o dicionário da PDB será ajustado à versão da CDB**. Um não substitui o outro.

## Test case 1 — reconhecer Container Map

Uma Container Map associa valores de uma coluna a Application PDBs. O otimizador usa o filtro para acessar apenas os containers relevantes.

Ela faz sentido em um Application Container com objetos compartilhados e dados distribuídos por uma chave, como região ou tenant. A tabela de mapa relaciona valores dessa chave aos containers. Quando a consulta no Application Root contém um predicado compatível, o Oracle pode podar PDBs que não possuem aqueles valores. Sem predicado utilizável, a consulta pode alcançar todas as PDBs abertas.

`CONTAINERS(tabela)` apenas projeta no root as linhas do objeto correspondente existente nos containers e acrescenta `CON_ID` para identificar a origem. Container Map é uma otimização e um desenho de roteamento, não apenas uma forma diferente de escrever `CONTAINERS()`.

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

A Proxy PDB possui uma representação local, mas encaminha as operações para a PDB referenciada. Isso permite incluir um banco remoto na hierarquia Multitenant local sem transferir datafiles. Em contrapartida, disponibilidade, latência e segurança passam a depender da rede, do database link e da PDB remota.

Antes de adotá-la, valide compatibilidade entre CDBs, credenciais do link, serviço remoto e comportamento esperado para DDL/DML. Se o objetivo for eliminar a dependência do servidor original, o mecanismo adequado é clone remoto ou relocation, não proxy.

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

Uma PDB plugada em CDB mais nova pode apresentar violações de compatibilidade até que seu dicionário e componentes sejam sincronizados. Abrir normalmente não corrige automaticamente todo cenário. `OPEN UPGRADE` restringe o uso para executar o processo de atualização; após o procedimento, reabra em modo normal e confirme o registry.

Replay Upgrade reduz indisponibilidade preparando parte do trabalho antes do cutover, enquanto AutoUpgrade coordena análise, fixups e execução. A escolha depende da versão de origem, destino e método suportado. Em qualquer opção, preserve backup, manifesto de unplug e caminho de retorno até concluir a validação funcional.

## Checklist de diagnóstico

```sql
select name, open_mode, restricted
  from v$pdbs
 order by con_id;

select property_name, property_value
  from database_properties
 where property_name in ('UPGRADE_PDB_ON_OPEN','PDB_UPGRADE_SYNC');
```

Complete o diagnóstico com `DBA_REGISTRY`, objetos inválidos, alert log e serviço da PDB. Uma PDB `READ WRITE` ainda pode ter componentes inválidos ou avisos não resolvidos; modo de abertura sozinho não comprova que a migração terminou corretamente.

## Referências oficiais

- [Application Containers e Container Map](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
- [Administering PDBs e Replay Upgrade](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/administering-pdbs-with-sql-plus.html)
- [ALTER PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/ALTER-PLUGGABLE-DATABASE.html)
