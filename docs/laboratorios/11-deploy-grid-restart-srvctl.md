---
title: Deploy — Grid Infrastructure, Oracle Restart e SRVCTL
description: Instalação image-based, CLUVFY, dry-run, switchGridHome, recursos e serviços
---

# Deploy — Grid Infrastructure, Oracle Restart e SRVCTL

## Modelo mental

| Ferramenta | Função principal |
|---|---|
| `gridSetup.sh` | Instalar, configurar ou atualizar Grid Infrastructure |
| `cluvfy` / `runcluvfy.sh` | Verificar pré-requisitos e readiness |
| `srvctl` | Registrar e administrar recursos Oracle |
| `crsctl` | Diagnosticar e administrar a pilha Clusterware/OHAS |

## Instalação image-based

Crie o novo Grid home, descompacte a imagem **dentro dele** e execute dali:

```bash
mkdir -p /u01/app/26ai/grid
cd /u01/app/26ai/grid
unzip /stage/grid_home.zip
./gridSetup.sh
```

Grupos comuns:

- `oinstall`: owner do Oracle Inventory.
- `asmadmin`: administração do ASM.

## Test case 1 — checagens antes da mudança

```bash
cluvfy stage -pre hacfg -n node1 -verbose

runcluvfy.sh stage -pre crsinst \
  -upgrade -rolling \
  -src_crshome /u01/app/19c/grid \
  -dest_crshome /u01/app/26ai/grid \
  -dest_version 26 -n node1,node2 -verbose
```

Dry-run do upgrade:

```bash
./gridSetup.sh -dryRunForUpgrade -responseFile /stage/gi_upgrade.rsp
```

O dry-run verifica readiness e não executa o upgrade. Não é suportado para Oracle Restart standalone.

## Test case 2 — switch de Grid home

Execute a partir do novo home preparado:

```bash
./gridSetup.sh -switchGridHome
```

Esse comando altera o Grid home ativo; não equivale a aplicar um RU no home antigo.

## Test case 3 — registrar recursos

```bash
srvctl add database \
  -db PROD \
  -oraclehome /u01/app/oracle/product/26ai/dbhome_1

srvctl modify database \
  -db PROD \
  -spfile '+DATA/PROD/spfileprod.ora'

srvctl add listener -listener LISTENER -endpoints TCP:1521
srvctl add service -db SALES -service REPORTING
```

Após mover o SPFILE, reinicie pelo `srvctl` para confirmar que o recurso usa a nova configuração.

Quando `-endpoints` é omitido ao adicionar um listener, a configuração padrão normalmente usa TCP/1521.

## Test case 4 — observar dependências

```bash
crsctl status resource -t
srvctl config database -db PROD
srvctl config service -db SALES
```

No Oracle Restart, o ASM precisa iniciar antes das bases que dependem dos diskgroups.

Em instalação RAC, o root script começa no nó local onde o instalador foi iniciado; nós remotos podem ser agrupados para execução paralela conforme o fluxo do instalador.

## Referências oficiais

- [Grid Infrastructure dry-run upgrade](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/about-oracle-clusterware-dry-run-upgrade-mode.html)
- [Grid Infrastructure Command-Line Options](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-command-line-options.html)
- [Grid Infrastructure Installation and Upgrade Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-infrastructure-installation-and-upgrade-guide-linux.pdf)
