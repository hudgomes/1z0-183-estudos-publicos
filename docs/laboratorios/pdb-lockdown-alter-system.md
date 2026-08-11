---
title: PDB Lockdown Profile e ALTER SYSTEM
description: Restrição de comandos administrativos dentro de uma PDB
---

# PDB Lockdown Profile e ALTER SYSTEM

## Como funciona

Lockdown Profiles limitam operações que usuários podem executar dentro de uma PDB, mesmo quando possuem privilégios locais. O profile é criado no `CDB$ROOT` e associado à PDB por `PDB_LOCKDOWN`.

O bloqueio pode atuar sobre statements, clauses, features ou options. O teste abaixo isola `ALTER SYSTEM` em uma PDB e usa outra PDB como controle.

## Test case — bloquear ALTER SYSTEM

No `CDB$ROOT`, como usuário administrativo:

```sql
create lockdown profile lab_lockdown;

alter lockdown profile lab_lockdown
  disable statement = ('ALTER SYSTEM');
```

Aplique somente na PDB de laboratório:

```sql
alter session set container=certlab;
alter system set pdb_lockdown = lab_lockdown;
```

Confira:

```sql
select name, value
  from v$parameter
 where name = 'pdb_lockdown';
```

Conectado como administrador local da `CERTLAB`, tente uma alteração reversível:

```sql
alter system set optimizer_mode = all_rows;
```

O comando deve ser bloqueado pelo profile. Em uma PDB sem o profile, a mesma operação depende apenas dos privilégios e da possibilidade de modificar o parâmetro naquele container.

## Inspecionar as regras

No root:

```sql
select profile_name, rule_type, rule, clause, clause_option, status
  from dba_lockdown_profiles
 where profile_name = 'LAB_LOCKDOWN'
 order by rule_type, rule;
```

## Tornar o teste mais específico

Bloquear todo `ALTER SYSTEM` é amplo. Em cenários reais, restrinja apenas a clause ou option necessária e valide os impactos da aplicação.

Exemplo conceitual:

```sql
alter lockdown profile lab_lockdown
  enable statement = ('ALTER SYSTEM');

alter lockdown profile lab_lockdown
  disable statement = ('ALTER SYSTEM') clause = ('SET');
```

## Limpeza

```sql
alter session set container=certlab;
alter system reset pdb_lockdown;

alter session set container=cdb$root;
drop lockdown profile lab_lockdown;
```

## Referências oficiais

- [Using PDB Lockdown Profiles](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/using-pdb-lockdown-profiles.html)
- [ALTER LOCKDOWN PROFILE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/ALTER-LOCKDOWN-PROFILE.html)
