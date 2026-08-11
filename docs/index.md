---
title: Estudos públicos — Oracle 1Z0-183
---

# Estudos públicos — Oracle 1Z0-183

Materiais autorais com explicação teórica e técnica, comparações entre recursos, laboratórios controlados, critérios de validação e referências oficiais do Oracle Database.

## Materiais disponíveis

### Transporte, plug e clonagem de PDB

Explica manifesto XML, arquivo `.pdb`, `COPY`, `MOVE`, `NOCOPY`, OMF, compatibilidade, hot clone e snapshot copy, com critérios para escolher cada método.

[Abrir guia](./laboratorios/01-multitenant-transporte-clonagem-pdb.html)

### CREATE PLUGGABLE DATABASE a partir de PDB$SEED

Mostra o papel da seed, o estado inicial da nova PDB, o administrador local, destino dos datafiles, primeira abertura, `SAVE STATE` e validação do serviço.

[Abrir guia](./laboratorios/create-pluggable-database-pdb-seed.html)

### Relocação online de PDB com indisponibilidade mínima

Detalha cópia com a origem em `READ WRITE`, database link, redo, acompanhamento, cutover, serviços e diferenças entre relocation, clone, unplug/plug e Data Guard.

[Abrir guia](./laboratorios/pdb-online-relocation-minimal-downtime.html)

### DROP PLUGGABLE DATABASE e tratamento dos datafiles

Compara `INCLUDING DATAFILES`, `KEEP DATAFILES` e `UNPLUG`, explicando o que acontece com os metadados, arquivos físicos, backups e possibilidades de transporte.

[Abrir guia](./laboratorios/drop-pdb-including-datafiles.html)

### Segurança, privilégios e recursos por PDB

Aborda usuários comuns e locais, escopo `CONTAINER`, grants herdados, modo restrito, parâmetros de memória, Unified Auditing e keystores TDE united e isolated.

[Abrir guia](./laboratorios/02-multitenant-seguranca-recursos-pdb.html)

### PDB Lockdown Profile e ALTER SYSTEM

Explica Lockdown Profile como camada adicional aos privilégios e demonstra regras por statement, clause e feature, usando uma PDB de controle para validar o bloqueio.

[Abrir guia](./laboratorios/pdb-lockdown-alter-system.html)

### ISPDB_MODIFIABLE — parâmetros por PDB

Mostra herança do root, sobrescrita local, `SCOPE`, propriedades de modificação, reset e comparação entre parâmetros locais e parâmetros estruturais da instância.

[Abrir guia](./laboratorios/ispdb-modifiable-pdb-parameters.html)

### Application Containers — ciclo de vida e compartilhamento

Explica Application Root, Application PDBs, tipos de `SHARING`, operações `INSTALL`, `SYNC`, `PATCH` e `UPGRADE`, versionamento e validação por tenant.

[Abrir guia](./laboratorios/oracle-application-containers-install-sync-patch.html)

### Consultas cross-container com CONTAINERS()

Apresenta consultas distribuídas a partir do root, identificação por `CON_ID`, objetos compartilhados, efeito do estado das PDBs e diferença entre `CONTAINERS()` e Container Map.

[Abrir guia](./laboratorios/cross-container-queries-containers.html)

### Operações Multitenant avançadas

Compara Container Map, Proxy PDB e upgrade de PDB, explicando roteamento, residência dos dados, compatibilidade, modos `UPGRADE`/`MIGRATE` e diagnóstico pós-migração.

[Abrir guia](./laboratorios/03-multitenant-operacoes-avancadas.html)

### Data Guard per PDB — source e target CDB

Explica proteção de uma PDB entre CDBs independentes, configurações do Broker, instanciação, transporte e aplicação, switchover, serviços e validação dos papéis no nível da PDB.

[Abrir guia](./laboratorios/data-guard-per-pdb-target-cdb.html)

### RMAN — estratégia, retenção e desempenho

Detalha retention policy, estados `OBSOLETE` e `EXPIRED`, incrementais, Block Change Tracking, image copies atualizadas, canais, multisection, FRA e exclusão de archivelogs.

[Abrir guia](./laboratorios/04-rman-estrategia-configuracao.html)

### RMAN — validação, corrupção e recuperação

Compara `VALIDATE`, `BACKUP VALIDATE` e `RESTORE VALIDATE`, além de Data Recovery Advisor, block recovery, restore/recover de datafile e PDB, control file e incarnations.

[Abrir guia](./laboratorios/05-rman-validacao-recuperacao.html)

### Flashback, restore points e transporte cross-platform

Explica Flashback Table, Flashback Drop, Flashback Database, restore points, instance recovery e transporte de tablespaces entre plataformas com conversão de endian.

[Abrir guia](./laboratorios/06-flashback-transporte-cross-platform.html)

### Performance — AWR, ADDM, ASH e métricas

Apresenta diagnóstico orientado por DB Time, snapshots, amostras ASH, relatórios AWR, análise ADDM, baselines, thresholds, alertas e interpretação de eventos de espera.

[Abrir guia](./laboratorios/07-performance-awr-addm-metricas.html)

### Performance — otimizador e SQL Tuning

Explica cardinalidade, Adaptive Plans, Statistics Feedback, estatísticas pendentes, function-based indexes, SQL Profiles, SQL Patches, SQL Plan Baselines, advisors e tracing seletivo.

[Abrir guia](./laboratorios/08-performance-otimizador-sql.html)

### Performance — workload, In-Memory e recursos

Compara SQL Performance Analyzer e Database Replay e detalha Database In-Memory, FastStart, HugePages, Resource Manager, paralelismo e compressão.

[Abrir guia](./laboratorios/09-performance-workload-inmemory-recursos.html)

### Deploy — DBCA e provisionamento

Explica as camadas do provisionamento, validação de pré-requisitos, criação silenciosa de CDB/PDB, OMF, FRA, `ARCHIVELOG`, templates, serviços e character set.

[Abrir guia](./laboratorios/10-deploy-dbca-provisionamento.html)

### Deploy — Grid Infrastructure e SRVCTL

Aborda instalação image-based, grupos do sistema operacional, CLUVFY, dry-run, troca out-of-place do Grid home, Oracle Restart, Clusterware, recursos e serviços.

[Abrir guia](./laboratorios/11-deploy-grid-restart-srvctl.html)

### Deploy — patching e AutoUpgrade

Separa patch binário, `datapatch` e upgrade, detalhando OPatchAuto, Gold Images, Fleet Patching, Data Guard, fases do AutoUpgrade e validação por container.

[Abrir guia](./laboratorios/12-deploy-patching-upgrade.html)

### Oracle AI Vector Search

Explica embeddings, tipo `VECTOR`, métricas de distância, busca exata e aproximada, HNSW, IVF, Vector Pool, target accuracy, ONNX, quantização e hybrid search.

[Abrir guia](./laboratorios/13-ai-vector-search.html)

### Oracle AI Database — recursos para desenvolvedores

Apresenta privilégios por schema, `DB_DEVELOPER_ROLE`, SQL Domains, annotations, JSON-relational duality, ETAG, Lock-Free Reservations, Priority Transactions e SQL Transpiler.

[Abrir guia](./laboratorios/14-ai-recursos-desenvolvedor.html)

### Oracle AI Database — segurança, disponibilidade e distribuição

Separa os papéis de True Cache, SQL Firewall, TLS, Raft replication e Application Continuity, incluindo consistência, quorum, allow-list, serviços e failover.

[Abrir guia](./laboratorios/15-ai-seguranca-distribuicao-cache.html)
