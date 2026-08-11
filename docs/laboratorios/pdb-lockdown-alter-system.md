---
title: PDB Lockdown Profile e ALTER SYSTEM
description: Restrição de comandos administrativos dentro de uma PDB
---

# PDB Lockdown Profile e ALTER SYSTEM

## Como funciona

Lockdown Profiles limitam operações que usuários podem executar dentro de uma PDB, mesmo quando possuem privilégios locais. O profile é criado no `CDB$ROOT` e associado à PDB por `PDB_LOCKDOWN`.

O bloqueio pode atuar sobre statements, clauses, features ou options. O teste abaixo isola `ALTER SYSTEM` em uma PDB e usa outra PDB como controle.

## Defesa adicional ao privilégio

Privilégios dizem o que um usuário está autorizado a fazer. Lockdown Profile impõe uma fronteira adicional ao container, inclusive para usuários locais muito privilegiados. Isso reduz a capacidade de uma PDB interferir em recursos compartilhados ou usar funcionalidades que o provedor não deseja expor.

O profile não concede nada. Uma operação só funciona quando o usuário possui os privilégios necessários **e** nenhuma regra de lockdown a proíbe. Revogar privilégio continua sendo a primeira camada; lockdown é útil para reforçar isolamento e padronizar limites entre tenants.

Profiles são criados no `CDB$ROOT` porque fazem parte da administração da CDB. A associação por `PDB_LOCKDOWN` define qual conjunto de regras vale na PDB. Outra PDB sem essa associação permanece com seu próprio comportamento.

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

O erro demonstra que a regra foi aplicada, mas registre também o usuário, container e comando. Um erro de privilégio ou parâmetro não modificável pode parecer sucesso do lockdown. A PDB de controle ajuda a provar que a diferença é realmente o profile.

## Inspecionar as regras

No root:

```sql
select profile_name, rule_type, rule, clause, clause_option, status
  from dba_lockdown_profiles
 where profile_name = 'LAB_LOCKDOWN'
 order by rule_type, rule;
```

As regras podem desabilitar um statement inteiro ou apenas clauses/options. Features abrangem capacidades como acesso a rede, sistema operacional ou determinados recursos, conforme a sintaxe suportada. Uma regra ampla é simples, porém pode bloquear administração legítima; uma regra específica reduz efeito colateral, mas exige inventário do workload.

Ordem e combinação de regras importam quando um statement geral é habilitado e uma clause específica é desabilitada. Consulte o dicionário depois de cada alteração em vez de confiar apenas no script pretendido.

## Tornar o teste mais específico

Bloquear todo `ALTER SYSTEM` é amplo. Em cenários reais, restrinja apenas a clause ou option necessária e valide os impactos da aplicação.

Exemplo conceitual:

```sql
alter lockdown profile lab_lockdown
  enable statement = ('ALTER SYSTEM');

alter lockdown profile lab_lockdown
  disable statement = ('ALTER SYSTEM') clause = ('SET');
```

Construa a política começando pelo requisito. Para impedir alteração de parâmetros, uma clause específica é melhor que bloquear operações não relacionadas do mesmo statement. Para impedir uma feature, use a categoria própria em vez de tentar listar todos os comandos que poderiam alcançá-la.

Teste com:

- usuário local comum sem privilégio;
- administrador local que teria o privilégio;
- PDB protegida;
- PDB de controle;
- operação permitida e operação bloqueada.

Essa matriz separa efeito de grant, efeito do profile e erro de sintaxe.

## Limpeza

```sql
alter session set container=certlab;
alter system reset pdb_lockdown;

alter session set container=cdb$root;
drop lockdown profile lab_lockdown;
```

Remova primeiro a associação da PDB e só depois exclua o profile. Em ambiente real, confirme se outras PDBs usam o mesmo nome antes do drop. Mudanças em lockdown devem ser tratadas como segurança de plataforma, com auditoria e teste de regressão dos jobs administrativos.

## Referências oficiais

- [PDB_LOCKDOWN](https://docs.oracle.com/en/database/oracle/oracle-database/26/refrn/PDB_LOCKDOWN.html)
- [ALTER LOCKDOWN PROFILE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/ALTER-LOCKDOWN-PROFILE.html)
