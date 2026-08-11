---
title: Oracle AI Database — segurança, disponibilidade e distribuição
description: True Cache, SQL Firewall, TLS, Raft e Application Continuity
---

# Oracle AI Database — segurança, disponibilidade e distribuição

## True Cache

True Cache é uma instância sem datafiles próprios para a aplicação: mantém blocos em memória, recebe mudanças do primary e atende consultas read-only. A consistência é controlada por SCNs.

Ele não é um banco independente nem um standby de recuperação. O primary continua sendo a fonte de dados e o destino de DML. O cache aquece conforme as consultas, mantém apenas o conjunto útil e pode ser reconstruído. Separar serviços permite que o driver escolha o destino sem espalhar regras de roteamento pelo SQL.

O padrão recomendado usa serviços separados:

```text
SALES     -> serviço do primary, aceita DML
SALES_TC  -> serviço do True Cache, somente leitura
```

Com cliente/UCP compatível, a aplicação usa o serviço para encaminhar leituras ao cache e DML ao primary sem codificar database links em cada consulta.

A aplicação precisa definir quão recente deve ser a leitura. O mecanismo usa SCNs para garantir consistência de sessão ou nível requerido e pode esperar o cache alcançar o primary. Quanto mais rígida a consistência, menor a chance de leitura antiga e maior a possibilidade de espera ou encaminhamento ao primary.

True Cache escala leituras; não aumenta capacidade de escrita. Consultas que dependem de estado não confirmado, locks ou DML devem permanecer no serviço do primary.

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

Capture um ciclo representativo: login, rotinas normais, jobs, manutenção autorizada e variações legítimas de SQL. Uma allow-list criada com workload incompleto causa bloqueio de operações válidas. Revise as views de captura e violações antes de ativar enforcement.

O firewall avalia SQL e contextos permitidos para usuários protegidos. Modos de observação ajudam a calibrar sem interromper produção; modo de bloqueio deve entrar depois de testes e procedimento de exceção. Contas administrativas e automações precisam de escopo deliberado, não exclusão automática sem análise.

## TLS

TLS 1.3 protege tráfego de rede suportado. Verifique versão do servidor, cliente, cipher suites, wallet e configuração do listener; habilitar TLS não substitui autenticação nem autorização.

TLS oferece confidencialidade e integridade em trânsito. Autenticação unilateral valida o servidor; mutual TLS também valida certificado do cliente. O listener precisa de endpoint TCPS e wallet, e o cliente precisa confiar na cadeia emissora e conferir identidade do servidor. Manter TCP sem criptografia em paralelo pode anular a política se aplicações continuarem usando o endpoint antigo.

Valide pelo connect descriptor real da aplicação, versão negociada e logs, não apenas pela existência de certificados. Rotação, expiração e proteção da chave privada fazem parte da operação.

## Raft replication

Em Oracle Globally Distributed Database, Raft replica unidades menores e decide commits por maioria.

```text
Leader + 2 Followers = 3 réplicas
Leader + 1 Follower disponível = maioria de 2 -> commit possível
```

O mecanismo oferece failover automático e zero data loss dentro do shard group quando há quorum. Ele não fornece HA para o shard catalog; proteja o catálogo separadamente, por exemplo com Data Guard.

Em um grupo com três réplicas, duas formam maioria. Perder uma réplica mantém commits; perder quorum interrompe progresso de escrita para evitar duas histórias divergentes. Isso é consistência por consenso, não simples replicação assíncrona.

Raft protege a unidade replicada no shard group. Roteamento global, shard directors e catálogo possuem papéis diferentes e precisam de desenho próprio de disponibilidade. Quorum também não substitui backup contra exclusão lógica propagada ou retenção histórica.

## Application Continuity

Para manutenção planejada, o serviço precisa estar configurado antes das novas conexões:

```text
FAILOVER_TYPE=AUTO
COMMIT_OUTCOME=TRUE
```

`FAILOVER_TYPE=AUTO` habilita Transparent Application Continuity quando suportado. `COMMIT_OUTCOME=TRUE` ajuda a resolver se o commit ocorreu após uma falha de comunicação.

Application Continuity registra trabalho recuperável da sessão e pode reproduzi-lo em outra instância após falha. Transaction Guard fornece outcome do commit para evitar repetir transação que talvez já tenha sido confirmada. TAC automatiza cobertura para aplicações compatíveis, mas estado de sessão não recuperável e chamadas com efeitos externos precisam de análise.

O serviço também deve definir drain timeout, stop option, failover restore e objetivos de balanceamento conforme a arquitetura. Em manutenção planejada, drene conexões antes de parar a instância e teste com o mesmo pool/driver usado em produção.

## Como separar os cinco recursos

| Necessidade | Recurso |
|---|---|
| Escalar consultas mantendo primary para DML | True Cache |
| Permitir somente padrões SQL/contextos conhecidos | SQL Firewall |
| Proteger tráfego de rede | TLS |
| Manter consenso entre réplicas de um shard | Raft |
| Preservar experiência da sessão durante failover | Application Continuity |

Eles podem coexistir. Cada um resolve uma camada diferente e nenhum, isoladamente, entrega segurança e disponibilidade completas.

## Referências oficiais

- [True Cache Configuration](https://docs.oracle.com/en/database/oracle/oracle-database/26/odbtc/overview-true-cache-configuration.html)
- [Getting Started with SQL Firewall](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlfw/getting-started-sql-firewall.html)
- [Raft Replication](https://docs.oracle.com/en/database/oracle/oracle-database/26/shard/raft-replication.html)
- [Ensuring Application Continuity](https://docs.oracle.com/en/database/oracle/oracle-database/26/racad/ensuring-application-continuity.html)
