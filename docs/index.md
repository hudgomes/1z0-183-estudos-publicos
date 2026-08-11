---
title: Estudos públicos — Oracle 1Z0-183
---

# Estudos públicos — Oracle 1Z0-183

Laboratórios práticos e materiais autorais para reforçar conceitos de administração do Oracle Database.

## Laboratórios disponíveis

### Application Containers — ciclo de vida e compartilhamento

Criação de Application Root e Application PDBs, instalação e versionamento, sincronização, patch, upgrade, tipos de `SHARING`, diagnóstico e limpeza do ambiente.

[Abrir laboratório: INSTALL, SYNC e PATCH](./laboratorios/oracle-application-containers-install-sync-patch.html)

### CREATE PLUGGABLE DATABASE a partir de PDB$SEED

Teste prático do usuário administrativo local, role `PDB_DBA`, estado inicial `MOUNTED`, status `NEW` e primeira abertura em `READ WRITE`.

[Abrir laboratório: criar PDB a partir da seed](./laboratorios/create-pluggable-database-pdb-seed.html)

### PDB Lockdown Profile e ALTER SYSTEM

Criação e aplicação de um Lockdown Profile para bloquear `ALTER SYSTEM` em uma PDB específica, mantendo uma segunda PDB como controle.

[Abrir laboratório: PDB Lockdown Profile](./laboratorios/pdb-lockdown-alter-system.html)

### DROP PLUGGABLE DATABASE INCLUDING DATAFILES

Teste destrutivo controlado para validar a remoção da PDB no control file e a exclusão física dos datafiles, comparando também `KEEP DATAFILES` e `UNPLUG`.

[Abrir laboratório: remover PDB e datafiles](./laboratorios/drop-pdb-including-datafiles.html)

### Data Guard per PDB — target CDB

Arquitetura DGPDB entre duas CDBs primárias independentes, configuração pelo Broker e validação dos papéis source e target no nível da PDB.

[Abrir laboratório: Data Guard per PDB](./laboratorios/data-guard-per-pdb-target-cdb.html)

### Relocação online de PDB com indisponibilidade mínima

Migração de uma PDB entre CDBs com a origem em `READ WRITE` durante a cópia, database link, `RELOCATE`, cutover e comparação entre `AVAILABILITY NORMAL` e `MAX`.

[Abrir laboratório: relocação online de PDB](./laboratorios/pdb-online-relocation-minimal-downtime.html)

### ISPDB_MODIFIABLE — validação de parâmetros por PDB

Consulta a `V$PARAMETER`, alteração controlada de `OPTIMIZER_USE_SQL_PLAN_BASELINES`, herança do CDB root e comparação com parâmetros não modificáveis por PDB.

[Abrir laboratório: parâmetros modificáveis por PDB](./laboratorios/ispdb-modifiable-pdb-parameters.html)

### Consultas cross-container com CONTAINERS()

Relatórios agregados entre Application PDBs com uma única consulta executada no Application Root, identificação por `CON_ID` e testes com PDBs abertas e fechadas.

[Abrir laboratório: consultas cross-container](./laboratorios/cross-container-queries-containers.html)
