---
title: Data Guard per PDB — source e target CDB
description: Proteção de uma PDB entre CDBs independentes com Data Guard Broker
---

# Data Guard per PDB — source e target CDB

## Arquitetura

Data Guard per PDB protege uma PDB individual entre duas CDBs que continuam operando como databases primárias independentes.

```text
SOURCE CDB (PRIMARY)             TARGET CDB (PRIMARY)
  └─ SALES (source PDB)           └─ DGPDB_SALES (target PDB)
         primary role                    standby role
```

O papel de standby pertence à target PDB. A target CDB não precisa ser uma physical standby da source CDB e pode hospedar outras PDBs.

## Diferença para Data Guard tradicional

No Data Guard tradicional, uma physical standby representa o database inteiro: control file, redo e datafiles acompanham o primary. Em Data Guard per PDB, a unidade protegida é uma PDB, e as CDBs dos dois lados continuam independentes. A source CDB pode ser primary de suas PDBs e, ao mesmo tempo, hospedar outra PDB que atua como standby de uma origem diferente, conforme topologia suportada.

O Broker coordena duas configurações e conhece o pareamento source/target. O redo referente à PDB protegida é transportado e aplicado no destino. Operações de role transition trocam o papel da PDB, não transformam toda a target CDB em standby.

Essa granularidade facilita proteção seletiva, mas aumenta a importância de serviços e roteamento. A aplicação precisa conectar ao serviço que acompanha a PDB atualmente primária; conectar ao serviço genérico da CDB pode direcionar ao lugar errado após switchover.

## Pré-requisitos

- Oracle Database release e RU compatíveis com DGPDB.
- `ARCHIVELOG`, FORCE LOGGING e Data Guard Broker configurados.
- Conectividade e serviços estáticos/dinâmicos testados.
- Source e target CDB registradas em configurações Broker.
- Destinos dos datafiles e credenciais protegidos.
- Backup e plano de rollback antes da configuração.

Source e target precisam usar Local Undo e atender requisitos específicos de versão, patches, wallet/credenciais e propriedades do Broker. Não assuma que duas CDBs em `ARCHIVELOG` já estão prontas. Valide também timezone, character set, opções instaladas, espaço e conversão de nomes de arquivos.

Force Logging reduz risco de operações nologging deixarem o destino irrecuperável. Ele não corrige blocos já criados sem redo antes da ativação. Faça backup base e confirme ausência de operações não registradas conforme o procedimento oficial.

## Test case guiado — Broker

Crie e valide a configuração da origem:

```text
DGMGRL> CONNECT sys@source_cdb
DGMGRL> CREATE CONFIGURATION SourceConfig
         AS PRIMARY DATABASE IS source_cdb
         CONNECT IDENTIFIER IS source_cdb;
DGMGRL> SHOW CONFIGURATION;
```

Em outra sessão, crie a configuração da target CDB:

```text
DGMGRL> CONNECT sys@target_cdb
DGMGRL> CREATE CONFIGURATION TargetConfig
         AS PRIMARY DATABASE IS target_cdb
         CONNECT IDENTIFIER IS target_cdb;
DGMGRL> SHOW CONFIGURATION;
```

Associe as configurações:

```text
DGMGRL> CONNECT sys@source_cdb
DGMGRL> ADD CONFIGURATION TargetConfig
         CONNECT IDENTIFIER IS target_cdb;
```

Cada `CONNECT IDENTIFIER` precisa funcionar dos hosts envolvidos e resolver para um serviço adequado à administração do Broker. Registro estático pode ser necessário para operações quando a instância ainda não está aberta. Teste resolução e autenticação antes de iniciar cópia de datafiles.

Adicione a target PDB, ajustando os paths reais:

```text
DGMGRL> ADD PLUGGABLE DATABASE 'dgpdb_sales' AT 'target_cdb'
         SOURCE IS 'sales' AT 'source_cdb'
         PDBFileNameConvert IS
         '/oradata/source/sales,/oradata/target/dgpdb_sales';
```

Depois da instanciação/cópia dos arquivos:

```text
DGMGRL> ENABLE CONFIGURATION ALL;
```

`PDBFileNameConvert` mapeia diretórios quando OMF não resolve o destino. Em ASM/OMF, use a forma recomendada para a plataforma. A fase de instanciação fornece a cópia inicial; redo recebido depois mantém a target PDB alinhada. A proteção não está pronta apenas porque o objeto aparece no Broker: aplicação, lag e status precisam estar saudáveis.

## Validação

```text
DGMGRL> SHOW DATABASE source_cdb;
DGMGRL> SHOW DATABASE target_cdb;
DGMGRL> SHOW PLUGGABLE DATABASE sales AT source_cdb;
DGMGRL> SHOW PLUGGABLE DATABASE dgpdb_sales AT target_cdb;
```

No SQL, confirme cada contexto:

```sql
select name, database_role, open_mode from v$database;
select name, open_mode from v$pdbs order by con_id;
```

Uma troca planejada é coordenada pelo Broker:

```text
DGMGRL> SWITCHOVER TO PLUGGABLE DATABASE
         dgpdb_sales AT target_cdb;
```

Faça esse passo somente após validar saúde, lag, conectividade, serviços e procedimento de retorno.

Durante o switchover, o Broker coordena fechamento, aplicação final e troca dos papéis. A antiga target torna-se a nova source primária da PDB, e a antiga source passa ao papel de standby correspondente. Outras PDBs das duas CDBs não trocam de papel.

Failover é usado quando a origem não pode ser alcançada e pode exigir reintegração posterior. A garantia de perda zero depende de modo de proteção, transporte e aplicação estarem em condições adequadas antes da falha.

## Checklist depois da transição

- confirmar roles e status pelo DGMGRL;
- validar lag de transporte/aplicação;
- abrir o serviço de escrita somente no novo primary da PDB;
- testar conexão e uma transação controlada;
- confirmar que o lado anterior recebe e aplica redo;
- revisar alert logs e mensagens do Broker.

Não use somente `V$DATABASE.DATABASE_ROLE` para concluir o papel da PDB: a CDB pode continuar `PRIMARY` nos dois servidores. O estado específico deve ser verificado pelos comandos DGPDB do Broker.

## Referências oficiais

- [Data Guard Broker Concepts](https://docs.oracle.com/en/database/oracle/oracle-database/26/dgbkr/oracle-data-guard-broker-concepts.html)
- [Managing Members of a DG PDB Broker Configuration](https://docs.oracle.com/en/database/oracle/oracle-database/26/dgbkr/managing-oracle-data-guard-broker-dgpdb-configuration-members.html)
