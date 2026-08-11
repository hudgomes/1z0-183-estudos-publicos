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

## Test case 1 — analisar antes de aplicar

```bash
opatchauto apply /stage/RU -analyze
```

`-analyze` verifica aplicabilidade e conflitos sem instalar o patch.

Durante instalação de um novo Grid home já atualizado:

```bash
./gridSetup.sh -applyRU /stage/patch_36000000
```

`opatchauto` consegue coordenar Grid home e Database homes e automatiza stop/start da pilha quando necessário; `opatch` sozinho não faz essa orquestração.

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

## Fleet Patching and Provisioning

FPP usa Gold Images para provisionar e atualizar muitos homes de forma padronizada. Pode administrar Grid Infrastructure e Oracle Database. Local Rolling Patching cria um ambiente transitório no mesmo servidor, aplica o patch e realiza a troca com redução de indisponibilidade.

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

## Data Guard e patch

Estratégia típica quando o patch permite:

```text
1. Aplicar binários no standby
2. Aplicar binários no primary
3. Executar datapatch no primary
```

Sempre siga o README específico do RU; a ordem pode variar por tecnologia e patch.

## HugePages

Para HugePages no Linux, use ASMM e deixe `MEMORY_TARGET` e `MEMORY_MAX_TARGET` em zero. AMM não usa HugePages da forma esperada.

## Referências oficiais

- [AutoUpgrade Processing Modes](https://docs.oracle.com/en/database/oracle/oracle-database/26/upgrd/about-autoupgrade-processing-modes.html)
- [Preparing to Upgrade Oracle Database](https://docs.oracle.com/en/database/oracle/oracle-database/26/upgrd/tasks-prepare-upgrade-oracle-database.html)
- [Grid Infrastructure Installation and Upgrade Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-infrastructure-installation-and-upgrade-guide-linux.pdf)
