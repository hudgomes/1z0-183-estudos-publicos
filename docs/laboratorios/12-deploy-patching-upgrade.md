---
title: Deploy — patching, Fleet Patching e AutoUpgrade
description: OPatchAuto, datapatch, Gold Images, Local Rolling, Data Guard e análise de upgrade
---

# Deploy — patching, Fleet Patching e AutoUpgrade

## Não confunda as camadas

```text
OPatch      -> aplica binários em um Oracle Home
OPatchAuto  -> orquestra GI + homes gerenciados + recursos
datapatch   -> aplica alterações SQL no dicionário
FPP         -> padroniza e distribui Gold Images em escala
AutoUpgrade -> analisa, corrige e executa upgrade de release
```

Patch atualiza uma instalação dentro de uma linha de release; upgrade move o banco para um release novo e altera compatibilidade/dicionário. Ambos podem usar novo Oracle home, mas seus objetivos e procedimentos não são intercambiáveis.

Também separe **binário** e **SQL**. OPatch altera arquivos do home. `datapatch` conecta ao banco e registra/aplica componentes SQL. Uma instalação pode mostrar o RU correto em `opatch lsinventory` e ainda ter dicionário pendente em alguma PDB.

Out-of-place patching prepara um home novo e troca a execução para ele. Facilita padronização e retorno ao home anterior, embora alterações SQL e mudanças de dados possam exigir procedimento próprio de rollback. In-place altera o home em uso e geralmente oferece menos isolamento.

## Test case 1 — analisar antes de aplicar

```bash
opatchauto apply /stage/RU -analyze
```

`-analyze` verifica aplicabilidade e conflitos sem instalar o patch.

Antes da análise, leia o README específico, confira espaço, versão do OPatch, inventário e patches sobrepostos. Conflito detectado pode exigir merge patch ou rollback de um interim patch; não force aplicação sobre inventário inconsistente.

Durante instalação de um novo Grid home já atualizado:

```bash
./gridSetup.sh -applyRU /stage/patch_36000000
```

`opatchauto` consegue coordenar Grid home e Database homes e automatiza stop/start da pilha quando necessário; `opatch` sozinho não faz essa orquestração.

`gridSetup.sh -applyRU` aplica o RU ao home durante instalação/configuração. Ele não atualiza automaticamente o dicionário de todos os bancos que futuramente usarão esse home. Quando um database home passa a executar a nova imagem, valide inventário binário e depois o estado SQL de cada CDB/PDB.

## Test case 2 — conferir datapatch em CDB

Abra as PDBs que devem receber alterações SQL:

```sql
alter pluggable database all open;
```

No host:

```bash
datapatch -verbose
```

Depois:

```sql
select con_id, patch_id, action, status, action_time
  from cdb_registry_sqlpatch
 order by action_time desc;
```

`datapatch` processa o root e as PDBs abertas. PDB fechada precisa ser aberta e reconciliada depois.

`CDB_REGISTRY_SQLPATCH` mostra o resultado por container. Uma linha `SUCCESS` no root não significa que PDBs fechadas receberam o patch. Depois de abri-las, execute ou deixe o mecanismo suportado reconciliar o SQL conforme o README, e confirme cada `CON_ID`. Objetos inválidos e alert log completam a validação.

Não execute `datapatch` antes de confirmar que a instância está usando o Oracle home atualizado. O utilitário precisa corresponder aos binários em execução.

## Fleet Patching and Provisioning

FPP usa Gold Images para provisionar e atualizar muitos homes de forma padronizada. Pode administrar Grid Infrastructure e Oracle Database. Local Rolling Patching cria um ambiente transitório no mesmo servidor, aplica o patch e realiza a troca com redução de indisponibilidade.

Gold Image representa um home conhecido, com nível de patch e configuração controlados; não é backup de database. FPP distribui a imagem e coordena movimentos em frota, reduzindo variação entre servidores. Local Rolling usa capacidade adicional temporária e serviços para deslocar o workload; sua disponibilidade depende da arquitetura e dos recursos suportados.

## Test case 3 — AutoUpgrade Analyze

Antes de upgrade, reúna estatísticas do dicionário e esvazie o recycle bin quando recomendado:

```sql
exec dbms_stats.gather_dictionary_stats;
purge dba_recyclebin;
```

Execute a análise:

```bash
java -jar autoupgrade.jar \
  -config /stage/lab.cfg \
  -mode analyze
```

Analyze é somente leitura para os objetos da aplicação e gera relatórios de readiness. `fixups` pode alterar a origem; faça backup antes.

AutoUpgrade possui fases como analyze, fixups, deploy e postupgrade. `analyze` pode ser executado antecipadamente e repetido. Os relatórios classificam checks, ações automáticas e tarefas manuais. `fixups` não deve ser confundido com relatório: ele executa correções permitidas e precisa de revisão e backup.

O arquivo de configuração separa propriedades globais e jobs por banco. Caminhos de homes, SID, target version, log directory e estratégia de timezone precisam corresponder ao host. Depois do upgrade, valide registry, timezone, objetos inválidos, parâmetros obsoletos e conectividade das aplicações antes de alterar `COMPATIBLE` de forma irreversível.

## Data Guard e patch

Estratégia típica quando o patch permite:

```text
1. Aplicar binários no standby
2. Aplicar binários no primary
3. Executar datapatch no primary
```

Sempre siga o README específico do RU; a ordem pode variar por tecnologia e patch.

Data Guard permite reduzir indisponibilidade aplicando binários por lado e realizando switchover quando suportado. Ainda assim, redo transport não replica mudança de Oracle home. Primary e standby devem terminar em níveis compatíveis, e `datapatch` normalmente ocorre no primary aberto para que as alterações SQL sejam enviadas por redo.

## HugePages

Para HugePages no Linux, use ASMM e deixe `MEMORY_TARGET` e `MEMORY_MAX_TARGET` em zero. AMM não usa HugePages da forma esperada.

Após trocar kernel, memória ou tamanho de SGA durante um upgrade, recalcule HugePages. Subdimensionamento pode fazer parte da SGA usar páginas comuns; superdimensionamento reserva memória que outras cargas não conseguem usar.

## Referências oficiais

- [AutoUpgrade Processing Modes](https://docs.oracle.com/en/database/oracle/oracle-database/26/upgrd/about-autoupgrade-processing-modes.html)
- [Preparing to Upgrade Oracle Database](https://docs.oracle.com/en/database/oracle/oracle-database/26/upgrd/tasks-prepare-upgrade-oracle-database.html)
- [Grid Infrastructure Installation and Upgrade Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-infrastructure-installation-and-upgrade-guide-linux.pdf)
