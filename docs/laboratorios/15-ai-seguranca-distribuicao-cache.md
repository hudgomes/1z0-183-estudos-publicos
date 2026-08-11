---
title: Oracle AI Database — segurança, disponibilidade e distribuição
description: True Cache, SQL Firewall, TLS, Raft e Application Continuity
---

# Oracle AI Database — segurança, disponibilidade e distribuição

## True Cache

True Cache é uma instância sem datafiles próprios para a aplicação: mantém blocos em memória, recebe mudanças do primary e atende consultas read-only. A consistência é controlada por SCNs.

O padrão recomendado usa serviços separados:

```text
SALES     -> serviço do primary, aceita DML
SALES_TC  -> serviço do True Cache, somente leitura
```

Com cliente/UCP compatível, a aplicação usa o serviço para encaminhar leituras ao cache e DML ao primary sem codificar database links em cada consulta.

Validação conceitual:

```sql
select name, network_name, goal, failover_type
  from dba_services
 order by name;
```

## SQL Firewall

Fluxo correto:

```text
Enable -> Capture workload válido -> Review -> Generate allow-list -> Enforce
```

```sql
exec dbms_sql_firewall.enable;
exec dbms_sql_firewall.create_capture('APP_USER', top_level_only => false);
```

Depois de executar o workload esperado:

```sql
exec dbms_sql_firewall.stop_capture('APP_USER');
exec dbms_sql_firewall.generate_allow_list('APP_USER');

begin
  dbms_sql_firewall.enable_allow_list(
    username => 'APP_USER',
    enforce  => dbms_sql_firewall.enforce_sql,
    block    => true);
end;
/
```

SQL Firewall usa allow-list e pode apenas registrar ou também bloquear SQL/contextos não autorizados. Ele complementa binds, menor privilégio e revisão de código; não torna concatenação de entrada segura.

## TLS

TLS 1.3 protege tráfego de rede suportado. Verifique versão do servidor, cliente, cipher suites, wallet e configuração do listener; habilitar TLS não substitui autenticação nem autorização.

## Raft replication

Em Oracle Globally Distributed Database, Raft replica unidades menores e decide commits por maioria.

```text
Leader + 2 Followers = 3 réplicas
Leader + 1 Follower disponível = maioria de 2 -> commit possível
```

O mecanismo oferece failover automático e zero data loss dentro do shard group quando há quorum. Ele não fornece HA para o shard catalog; proteja o catálogo separadamente, por exemplo com Data Guard.

## Application Continuity

Para manutenção planejada, o serviço precisa estar configurado antes das novas conexões:

```text
FAILOVER_TYPE=AUTO
COMMIT_OUTCOME=TRUE
```

`FAILOVER_TYPE=AUTO` habilita Transparent Application Continuity quando suportado. `COMMIT_OUTCOME=TRUE` ajuda a resolver se o commit ocorreu após uma falha de comunicação.

## Referências oficiais

- [True Cache Configuration](https://docs.oracle.com/en/database/oracle/oracle-database/26/odbtc/overview-true-cache-configuration.html)
- [Getting Started with SQL Firewall](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlfw/getting-started-sql-firewall.html)
- [Raft Replication](https://docs.oracle.com/en/database/oracle/oracle-database/26/shard/raft-replication.html)
- [Ensuring Application Continuity](https://docs.oracle.com/en/database/oracle/oracle-database/26/racad/ensuring-application-continuity.html)
