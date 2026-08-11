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

Grid Infrastructure fornece ASM e a camada que mantém recursos Oracle disponíveis. Em servidor único, Oracle Restart monitora ASM, listener, banco e serviços e respeita suas dependências. Em cluster, Clusterware acrescenta membership, interconnect, recursos distribuídos e políticas RAC.

`srvctl` trabalha com a configuração suportada dos recursos Oracle. `crsctl` opera em nível mais baixo na infraestrutura e é especialmente útil para status e diagnóstico. Iniciar uma base manualmente com SQL*Plus pode funcionar, mas ignora política, dependências e estado conhecido pelo Oracle Restart/Clusterware; a administração normal deve usar `srvctl`.

## Instalação image-based

Crie o novo Grid home, descompacte a imagem **dentro dele** e execute dali:

```bash
mkdir -p /u01/app/26ai/grid
cd /u01/app/26ai/grid
unzip /stage/grid_home.zip
./gridSetup.sh
```

Na instalação image-based, o diretório onde a imagem foi extraída **é** o novo Oracle home. Não extraia em diretório temporário esperando que o instalador copie tudo depois. O home deve estar vazio, ter owner/grupo corretos e caminho definitivo. Patches podem ser incorporados no novo home antes da configuração, permitindo adoção out-of-place e retorno mais previsível.

Grupos comuns:

- `oinstall`: owner do Oracle Inventory.
- `asmadmin`: administração do ASM.

Outros grupos separam operações, como `dba`, `oper`, `asmdba` e `asmoper`. O owner do software precisa pertencer aos grupos necessários, mas menor privilégio continua válido: administração completa de ASM não deve ser concedida apenas para permitir que um database home acesse diskgroups.

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

CLUVFY possui stages diferentes. `-pre hacfg` verifica requisitos para Oracle Restart; `-pre crsinst` atende instalação/upgrade de cluster e recebe lista de nós. Um check bem-sucedido não substitui backup do OCR, voting files, configuração ASM e inventário. Guarde a saída antes e depois para comparar mudanças.

O dry-run percorre boa parte das verificações e do fluxo do upgrade sem fazer a troca definitiva. Ele ajuda a detectar permissões, inventário, comunicação e incompatibilidades, mas ainda exige janela e plano de retorno para a execução real.

## Test case 2 — switch de Grid home

Execute a partir do novo home preparado:

```bash
./gridSetup.sh -switchGridHome
```

Esse comando altera o Grid home ativo; não equivale a aplicar um RU no home antigo.

`-switchGridHome` adota um home preparado e atualizado, seguindo o modelo out-of-place. Antes da troca, confirme que o novo home está registrado no inventory, possui patches esperados e acesso aos mesmos dispositivos/configurações. Depois, valide OHAS/CRS, ASM, listeners, databases e serviços; apenas ver processos ativos não comprova que todos os recursos atingiram o target state.

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

`srvctl add` registra o recurso; não necessariamente cria os arquivos físicos ou altera o conteúdo do SPFILE. `srvctl modify` atualiza atributos registrados. `srvctl start` e `stop` acionam o recurso respeitando dependências. Antes de modificar, capture `srvctl config ...` para saber a configuração atual e facilitar retorno.

Serviços não são apenas aliases de conexão. Eles identificam workloads, permitem políticas de failover, balanceamento, Application Continuity e administração independente da instância. Uma aplicação deve conectar ao serviço adequado, não ao SID ou serviço administrativo padrão.

## Test case 4 — observar dependências

```bash
crsctl status resource -t
srvctl config database -db PROD
srvctl config service -db SALES
```

No Oracle Restart, o ASM precisa iniciar antes das bases que dependem dos diskgroups.

Dependências possuem direção e política. ASM monta diskgroups; listener publica endpoints; database abre; serviço inicia na instância disponível. `crsctl status resource -t` mostra estado atual e destino, mas detalhes de falha ficam nos logs do recurso e da infraestrutura. Diferencie `OFFLINE` planejado de `INTERMEDIATE` ou falha repetida.

Em instalação RAC, o root script começa no nó local onde o instalador foi iniciado; nós remotos podem ser agrupados para execução paralela conforme o fluxo do instalador.

Rolling upgrade mantém parte do cluster atendendo enquanto nós são atualizados, quando a combinação de versão e patch suporta. Isso reduz indisponibilidade, mas não elimina impacto de relocação de serviços, drain de sessões e capacidade temporariamente reduzida.

## Referências oficiais

- [Grid Infrastructure dry-run upgrade](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/about-oracle-clusterware-dry-run-upgrade-mode.html)
- [Grid Infrastructure Command-Line Options](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-command-line-options.html)
- [Grid Infrastructure Installation and Upgrade Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/cwlin/grid-infrastructure-installation-and-upgrade-guide-linux.pdf)
