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

## Roteiro compacto em 15 blocos

### 1. Transporte, plug e clonagem de PDB

Compatibilidade, XML e `.pdb`, `COPY`, `MOVE`, OMF, hot clone e snapshot copy.

[Abrir guia](./laboratorios/01-multitenant-transporte-clonagem-pdb.html)

### 2. Segurança, privilégios e recursos por PDB

Usuários comuns e locais, `CONTAINER`, auditoria, memória, estado restrito e keystore isolado.

[Abrir guia](./laboratorios/02-multitenant-seguranca-recursos-pdb.html)

### 3. Operações Multitenant avançadas

Container Map, Proxy PDB, upgrade e diagnóstico de compatibilidade.

[Abrir guia](./laboratorios/03-multitenant-operacoes-avancadas.html)

### 4. RMAN — estratégia, retenção e desempenho

Retention policy, FRA, block change tracking, image copies, canais e multisection.

[Abrir guia](./laboratorios/04-rman-estrategia-configuracao.html)

### 5. RMAN — validação, corrupção e recuperação

`VALIDATE`, Data Recovery Advisor, block recovery, restore, control file, PDB e incarnations.

[Abrir guia](./laboratorios/05-rman-validacao-recuperacao.html)

### 6. Flashback e transporte cross-platform

Flashback Table, Flashback Drop, restore points e conversão entre plataformas.

[Abrir guia](./laboratorios/06-flashback-transporte-cross-platform.html)

### 7. Performance — AWR, ADDM, ASH e métricas

Snapshots, baselines, relatórios, eventos de espera, alertas e diagnóstico histórico.

[Abrir guia](./laboratorios/07-performance-awr-addm-metricas.html)

### 8. Performance — otimizador e SQL Tuning

Adaptive plans, feedback, estatísticas pendentes, SQL Patch, baselines e monitoramento.

[Abrir guia](./laboratorios/08-performance-otimizador-sql.html)

### 9. Performance — workload, In-Memory e recursos

SPA, Database Replay, In-Memory, HugePages, paralelismo e Resource Manager.

[Abrir guia](./laboratorios/09-performance-workload-inmemory-recursos.html)

### 10. Deploy — DBCA e provisionamento

Pré-requisitos, CDB, OMF, templates, FRA, `ARCHIVELOG` e character set.

[Abrir guia](./laboratorios/10-deploy-dbca-provisionamento.html)

### 11. Deploy — Grid Infrastructure e SRVCTL

Instalação image-based, CLUVFY, dry-run, `switchGridHome`, Oracle Restart e serviços.

[Abrir guia](./laboratorios/11-deploy-grid-restart-srvctl.html)

### 12. Deploy — patching e AutoUpgrade

OPatchAuto, datapatch, Gold Images, Local Rolling, Data Guard e análise de upgrade.

[Abrir guia](./laboratorios/12-deploy-patching-upgrade.html)

### 13. Oracle AI Vector Search

`VECTOR`, métricas, busca exata, HNSW, quantização, hybrid search e embeddings ONNX.

[Abrir guia](./laboratorios/13-ai-vector-search.html)

### 14. Recursos 26ai para desenvolvedores

Privilégios por schema, SQL Domains, annotations, duality views e concorrência.

[Abrir guia](./laboratorios/14-ai-recursos-desenvolvedor.html)

### 15. Segurança, disponibilidade e distribuição

True Cache, SQL Firewall, TLS, Raft e Application Continuity.

[Abrir guia](./laboratorios/15-ai-seguranca-distribuicao-cache.html)
