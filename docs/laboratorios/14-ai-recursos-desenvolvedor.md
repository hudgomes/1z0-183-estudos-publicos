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

Privilégio no nível do schema fica entre grants objeto a objeto e privilégios globais `ANY`. Ele cobre objetos existentes e futuros do owner indicado, facilitando papéis de leitura ou desenvolvimento sem abrir outros schemas. O texto `ON SCHEMA` faz parte do grant; não cria um novo schema nem transfere ownership.

`SELECT ANY TABLE ON SCHEMA` permite consultas e respeita semântica de `SELECT`. `READ ANY TABLE ON SCHEMA` é mais restrito para alguns cenários porque não concede capacidades associadas a locking explícito. Revise a documentação do privilégio escolhido e evite concedê-lo diretamente a muitos usuários; um role facilita auditoria e revogação.

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

SQL Domain é um objeto de schema aplicado a colunas ou expressões. Ele fornece semântica reutilizável: tipo, constraints, collation, display e metadados podem ser definidos uma vez. Alterar um domain exige avaliar todos os consumidores; ele não é apenas um alias textual de datatype.

Uma constraint do domain valida novos dados e alterações conforme sua aplicação. Para dados existentes, confira o estado de validação ao associar ou evoluir o domain. A expressão deve ser simples e determinística o suficiente para o uso definido.

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

Annotations podem existir em objetos e colunas e ser consultadas pelo dicionário. Ferramentas podem usá-las para rótulos, classificação e geração de interfaces sem depender de convenções no nome. Como não executam regra de integridade, uma annotation `Sensitive` ou `Identity` só produz efeito se a ferramenta consumidora implementar esse significado.

## JSON-relational duality

Duality views expõem documentos JSON atualizáveis enquanto os dados permanecem normalizados em tabelas relacionais. O `ETAG` detecta atualizações concorrentes sem manter um lock durante toda a interação.

```sql
create json relational duality view lab_cliente_dv as
  select json {'_id' : c.id, 'nome' : c.nome}
    from lab_cliente_ai c
    with update insert delete;
```

A duality view define como tabelas e relações formam um documento. Leitura retorna JSON; insert, update e delete do documento podem ser decompostos em DML relacional conforme as permissões declaradas. As tabelas continuam sendo a fonte única, portanto SQL relacional e acesso por documento enxergam os mesmos dados.

O ETAG é calculado a partir dos campos participantes. O cliente lê documento e ETAG e devolve o valor ao atualizar; se outra transação modificou conteúdo relevante, o banco detecta conflito otimista. Isso evita manter lock durante o tempo de interação, mas exige que o cliente trate conflito e refaça ou reconcilie a alteração.

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

A reserva registra intenção sobre um valor, como reduzir estoque, e verifica constraints no commit. Outras transações podem criar deltas compatíveis sem esperar pelo lock tradicional da linha. A atualização precisa manter a forma relativa `coluna = coluna + expressão` ou `- expressão`; atribuição absoluta perde a semântica de reserva.

O recurso é adequado a contadores e quantidades com regras simples. Não substitui transação para alterações de múltiplas tabelas nem resolve sozinho reserva física, pagamento ou idempotência da aplicação.

## Priority Transactions

Priority Transactions resolve bloqueio entre transações; não é a mesma coisa que Lock-Free Reservations.

```sql
alter session set txn_priority = high;
alter system set priority_txns_high_wait_target = 15;
```

Quando habilitado, um blocker de prioridade inferior pode sofrer rollback automático; a sessão permanece viva e deve reconhecer o rollback.

Prioridade atua quando transações entram em conflito. O target define quanto uma transação de maior prioridade espera antes de o banco agir sobre blockers inferiores. Rollback da transação não desconecta necessariamente a sessão; o código precisa capturar o erro, limpar estado e decidir se repete a unidade de trabalho.

## SQL Transpiler

O SQL Transpiler transforma funções PL/SQL elegíveis em expressões SQL durante a compilação/otimização, reduzindo o custo de troca entre os engines SQL e PL/SQL. Valide pelo plano e tempo de execução; nem toda função é elegível.

Funções com efeitos colaterais, construções não suportadas ou dependências incompatíveis podem permanecer como chamadas PL/SQL convencionais. A transformação preserva semântica dentro das regras suportadas; ela não autoriza remover testes de resultado. Compare plano, chamadas e tempo com dados representativos.

## Validação integrada

No laboratório, consulte `DBA_SCHEMA_PRIVS` para o grant, `USER_DOMAINS` e views relacionadas para o domain, `USER_ANNOTATIONS_USAGE` para metadados e o DDL da duality view. Para concorrência, use duas sessões e registre commit, conflito e estado final; uma execução em sessão única não demonstra o benefício.

## Referências oficiais

- [GRANT e privilégios por schema](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/GRANT.html)
- [CREATE DOMAIN](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/create-domain.html)
- [Domains e annotations](https://docs.oracle.com/en/database/oracle/oracle-database/26/tdddg/creating-managing-schema-objects.html)
- [JSON-relational duality e concorrência](https://docs.oracle.com/en/database/oracle/oracle-database/26/jsnvu/using-optimistic-concurrency-control-duality-views.html)
