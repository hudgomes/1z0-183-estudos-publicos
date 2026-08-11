---
title: Oracle AI Database — recursos para desenvolvedores
description: Privilégios por schema, DB_DEVELOPER_ROLE, SQL Domains, annotations, duality views e concorrência
---

# Oracle AI Database — recursos para desenvolvedores

## Privilégios no nível do schema

Um grant por schema cobre objetos existentes e futuros sem conceder acesso a todos os schemas:

```sql
grant select any table on schema app_schema to reporter_role;
grant read any table on schema sales to reporting_role;
```

Confira:

```sql
select grantee, privilege, schema
  from dba_schema_privs
 where grantee in ('REPORTER_ROLE','REPORTING_ROLE');
```

`DB_DEVELOPER_ROLE` reúne privilégios comuns ao desenvolvimento, evitando grants administrativos excessivos. Ainda aplique menor privilégio e quota de tablespace.

## Test case 1 — SQL Domain

```sql
create domain email_domain as varchar2(200)
  constraint email_domain_ck
  check (regexp_like(email_domain, '^[^@]+@[^@]+\.[^@]+$'))
  annotations (Display 'E-mail');

create table lab_contato (
  id    number primary key,
  email email_domain
);
```

O domain centraliza tipo, validação, display, ordenação e annotations reutilizáveis.

## Test case 2 — annotations

```sql
create table lab_cliente_ai (
  id   number primary key annotations (Identity),
  nome varchar2(100) annotations (Display 'Nome do cliente')
) annotations (Display 'Clientes');

select object_name, column_name, annotation_name, annotation_value
  from user_annotations_usage
 order by object_name, column_name;
```

Annotations são metadados declarativos; não substituem constraints.

## JSON-relational duality

Duality views expõem documentos JSON atualizáveis enquanto os dados permanecem normalizados em tabelas relacionais. O `ETAG` detecta atualizações concorrentes sem manter um lock durante toda a interação.

```sql
create json relational duality view lab_cliente_dv as
  select json {'_id' : c.id, 'nome' : c.nome}
    from lab_cliente_ai c
    with update insert delete;
```

## Lock-Free Reservations

Use coluna numérica marcada como `RESERVABLE` e alterações delta:

```sql
create table lab_estoque (
  produto_id number primary key,
  quantidade number reservable
);

update lab_estoque
   set quantidade = quantidade - 1
 where produto_id = 10;
```

Reservas permitem atualizações concorrentes suportadas sem bloquear a linha da maneira tradicional. A coluna precisa ser numérica e a atualização deve ser delta.

## Priority Transactions

Priority Transactions resolve bloqueio entre transações; não é a mesma coisa que Lock-Free Reservations.

```sql
alter session set txn_priority = high;
alter system set priority_txns_high_wait_target = 15;
```

Quando habilitado, um blocker de prioridade inferior pode sofrer rollback automático; a sessão permanece viva e deve reconhecer o rollback.

## SQL Transpiler

O SQL Transpiler transforma funções PL/SQL elegíveis em expressões SQL durante a compilação/otimização, reduzindo o custo de troca entre os engines SQL e PL/SQL. Valide pelo plano e tempo de execução; nem toda função é elegível.

## Referências oficiais

- [GRANT e privilégios por schema](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/GRANT.html)
- [CREATE DOMAIN](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/create-domain.html)
- [Domains e annotations](https://docs.oracle.com/en/database/oracle/oracle-database/26/tdddg/creating-managing-schema-objects.html)
- [JSON-relational duality e concorrência](https://docs.oracle.com/en/database/oracle/oracle-database/26/jsnvu/using-optimistic-concurrency-control-duality-views.html)
