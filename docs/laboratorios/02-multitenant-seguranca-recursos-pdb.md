---
title: Segurança, privilégios e recursos por PDB
description: Usuários comuns e locais, CONTAINER, auditoria, memória, estado restrito e keystore isolado
---

# Segurança, privilégios e recursos por PDB

## Modelo mental

| Ação | Escopo |
|---|---|
| Grant no root sem cláusula | `CONTAINER=CURRENT`: somente o root |
| Grant no root com `CONTAINER=ALL` | Root e PDBs atuais/futuras |
| Grant dentro de uma PDB | Privilégio local naquela PDB |
| Usuário criado no Application Root com `CONTAINER=ALL` | Application Common User |
| `ALTER SESSION SET CONTAINER` | Exige `SET CONTAINER` ou privilégio administrativo compatível |

Um usuário comum pode receber um privilégio local. Por exemplo, conectar em `SALES_PDB` e executar `GRANT DBA TO C##ADMIN` concede o papel apenas naquele container.

## Identidade, origem e alcance

Segurança Multitenant possui três perguntas independentes: **onde o usuário foi criado**, **onde o privilégio foi concedido** e **em quais containers ele pode ser usado**. Um usuário comum existe no root e nas PDBs, mas não recebe automaticamente os mesmos poderes em todas elas. Um usuário local pertence a uma única PDB e não se conecta ao root.

Em uma CDB, nomes de usuários comuns normalmente usam o prefixo definido em `COMMON_USER_PREFIX`, geralmente `C##`. Em um Application Container também existem application common users, cujo alcance fica limitado ao Application Root e às Application PDBs associadas.

`CONTAINER=ALL` cria ou concede algo em escopo comum. `CONTAINER=CURRENT` limita a operação ao container atual e costuma ser o padrão quando a cláusula é omitida. A coluna `COMMON` mostra a origem comum do grant; `INHERITED` ajuda a identificar se o registro foi herdado do root. Essas colunas são mais confiáveis que deduzir o alcance apenas pelo nome do usuário.

## Test case 1 — comparar grant comum e local

No `CDB$ROOT`, como `SYS`:

```sql
create user c##cert_reader identified by "<SENHA_FORTE_EXCLUSIVA>"
  container=all;
grant create session to c##cert_reader container=all;
grant create table to c##cert_reader container=current;
```

Confirme o escopo:

```sql
select grantee, privilege, common, inherited
  from cdb_sys_privs
 where grantee = 'C##CERT_READER'
 order by con_id, privilege;
```

Agora conceda um privilégio apenas na PDB de laboratório:

```sql
alter session set container=certlab;
grant create table to c##cert_reader;

select grantee, privilege, common, inherited
  from dba_sys_privs
 where grantee = 'C##CERT_READER';
```

## Test case 2 — estado restrito persistente

No root:

```sql
alter pluggable database certlab close immediate;
alter pluggable database certlab open read write restricted;
alter pluggable database certlab save state;

select name, open_mode, restricted
  from v$pdbs
 where name = 'CERTLAB';
```

Somente sessões com `RESTRICTED SESSION` conseguem conectar. Para voltar:

```sql
alter pluggable database certlab close immediate;
alter pluggable database certlab open read write;
alter pluggable database certlab save state;
```

`RESTRICTED` é um modo de abertura, não um estado de proteção contra escrita. A PDB ainda pode estar `READ WRITE`, mas novas conexões ficam limitadas a usuários autorizados. É útil em manutenção, validação pós-patch e testes antes de liberar o serviço. `SAVE STATE` persiste modo de abertura e condição restrita para o próximo restart; sem ele, a PDB pode voltar fechada ou em outro estado.

## Test case 3 — limites de memória por PDB

Com Local Undo, entre em `CERTLAB` e verifique se os parâmetros são modificáveis:

```sql
alter session set container=certlab;

select name, value, ispdb_modifiable
  from v$parameter
 where name in ('sga_target','pga_aggregate_limit');
```

Em ambiente com memória disponível:

```sql
alter system set sga_target = 512m scope=both;
alter system set pga_aggregate_limit = 1g scope=both;
```

O CDB root continua impondo o limite físico global. Não dimensione PDBs sem comparar a soma das metas com a memória do host.

Nem todo parâmetro pode ser alterado dentro de uma PDB. `ISPDB_MODIFIABLE='TRUE'` indica que existe valor específico por PDB. O valor local limita ou direciona o consumo daquele container, enquanto a instância continua compartilhando SGA, PGA e CPU. Metas incompatíveis não criam memória: sob pressão, as PDBs competem dentro do limite global e das regras do Resource Manager.

Para CPU, sessões, paralelismo e tempo de execução, use um plano do Database Resource Manager. Parâmetros de memória e Resource Manager se complementam: um controla orçamento de memória; o outro distribui recursos e impõe limites ao workload.

## Auditoria e TDE

- Política local criada no root audita somente o root.
- Política comum habilitada no root pode valer em todos os PDBs.
- Um keystore em modo `ISOLATED` dá à PDB uma chave mestra independente.

Unified Auditing separa a **definição** da política de sua **habilitação**. Uma política pode existir e ainda não estar ativa para nenhum usuário. Políticas comuns são administradas no root com alcance comum; políticas locais registram atividades do container em que foram habilitadas. Na análise, consulte o `CON_ID` para não atribuir ao root uma ação ocorrida em uma PDB.

TDE também possui duas topologias. Em modo `UNITED`, as PDBs dependem do keystore e da administração do root. Em modo `ISOLATED`, uma PDB pode manter chave mestra própria, o que melhora separação de responsabilidades, mas adiciona procedimentos de abertura, backup e rotação de chaves. Criptografia não substitui o backup do keystore: perder as chaves torna os datafiles inutilizáveis.

Inspeção segura:

```sql
select con_id, wrl_type, status, wallet_type, keystore_mode
  from v$encryption_wallet
 order by con_id;
```

## Limpeza

```sql
alter session set container=cdb$root;
drop user c##cert_reader cascade;
```

## Referências oficiais

- [Configuring Privilege and Role Authorization](https://docs.oracle.com/en/database/oracle/oracle-database/26/dbseg/configuring-privilege-and-role-authorization.html)
- [Administering PDBs](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/administering-pdbs-with-sql-plus.html)
- [AUDIT — Unified Auditing](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/AUDIT-Unified-Auditing.html)
