---
title: Application Containers — ciclo de vida e compartilhamento
description: Application Root, Application PDB, INSTALL, SYNC, PATCH, UPGRADE e SHARING
---

# Application Containers — ciclo de vida e compartilhamento

## Arquitetura

Um Application Container agrupa uma aplicação comum e várias Application PDBs:

```text
CDB$ROOT
  └─ APP_ROOT                 Application Root
       ├─ APP_PDB1            tenant 1
       └─ APP_PDB2            tenant 2
```

O Application Root guarda a definição mestre versionada. Cada Application PDB sincroniza essa definição e mantém seus dados específicos.

Application Container organiza uma aplicação compartilhada dentro da CDB. O Application Root é o ponto de administração do modelo; Application PDBs funcionam como tenants. A versão de uma aplicação é registrada por operações `BEGIN ...` e `END ...`, permitindo ao Oracle saber qual mudança pertence a install, patch ou upgrade.

Criar objetos fora de uma operação da aplicação não os transforma automaticamente em objetos comuns versionados. Da mesma forma, alterar o root não atualiza instantaneamente todas as Application PDBs: `SYNC` reproduz as mudanças pendentes em cada tenant.

## Tipos de compartilhamento

| `SHARING` | Metadados | Dados |
|---|---|---|
| `METADATA` | Compartilhados | Separados por Application PDB |
| `DATA` | Compartilhados | Centralizados no Application Root |
| `EXTENDED DATA` | Compartilhados | Dados comuns mais dados locais |
| `NONE` | Não compartilhados | Não compartilhados |

`METADATA` mantém uma definição comum e segmentos de dados locais, adequado quando cada tenant possui suas linhas. `DATA` centraliza dados no root e expõe o mesmo conteúdo. `EXTENDED DATA` permite dados comuns no root e extensão local. `NONE` cria objeto comum apenas no sentido administrativo da operação, sem os links de compartilhamento.

A escolha é estrutural. Trocar depois o modo de compartilhamento pode exigir recriação/migração; escolha com base em isolamento, volume de DML, necessidade de dados comuns e comportamento das consultas cross-container.

## Test case — criar o container

No `CDB$ROOT`, com OMF ativo:

```sql
create pluggable database lab_app_root
  as application container
  admin user lab_app_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

alter pluggable database lab_app_root open;
alter session set container=lab_app_root;
```

Crie duas Application PDBs:

```sql
create pluggable database lab_app_pdb1
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

create pluggable database lab_app_pdb2
  admin user lab_pdb_admin
  identified by "<SENHA_FORTE_EXCLUSIVA>";

alter pluggable database all open;
```

As PDBs criadas dentro do Application Root pertencem a esse Application Container. Elas não devem ser confundidas com PDBs comuns conectadas diretamente ao `CDB$ROOT`. Um Application Seed opcional pode acelerar criação de tenants já sincronizados com determinada versão.

## Instalar a versão inicial

No Application Root:

```sql
alter pluggable database application lab_sales_app
  begin install '1.0';

create table lab_produtos (
  id    number primary key,
  nome  varchar2(100),
  preco number(10,2)
) sharing=metadata;

alter pluggable database application lab_sales_app
  end install '1.0';
```

Sincronize cada Application PDB:

```sql
alter session set container=lab_app_pdb1;
alter pluggable database application lab_sales_app sync;

alter session set container=lab_app_pdb2;
alter pluggable database application lab_sales_app sync;
```

Durante `BEGIN INSTALL`, o DDL é capturado como parte da versão inicial. `END INSTALL` fecha a operação no root, mas cada Application PDB continua em seu estado anterior até `SYNC`. Consulte `DBA_APPLICATIONS` no root e no tenant para enxergar essa diferença.

Se um tenant estiver fechado ou o sync falhar, os demais podem avançar e ele permanecer pendente. Por isso a implantação precisa registrar versão por PDB, erros de sync e caminho de correção.

## Aplicar patch

No Application Root:

```sql
alter session set container=lab_app_root;

alter pluggable database application lab_sales_app
  begin patch 1 minimum version '1.0';

alter table lab_produtos add categoria varchar2(50);

alter pluggable database application lab_sales_app
  end patch 1;
```

Sincronize novamente as PDBs. O patch existe no root antes de produzir efeito nos tenants.

Patch representa mudança compatível dentro da linha da aplicação e recebe número, além de versão mínima. Upgrade move a aplicação para uma versão declarada e é adequado a transição mais ampla. Ambos registram DDL no root e exigem sync; a diferença expressa o ciclo de vida e as regras de compatibilidade.

Uma Application PDB pode precisar aplicar patches em sequência. Não force um tenant diretamente à versão final sem verificar os caminhos registrados. Objetos e dados locais também podem exigir scripts de transformação que sejam seguros quando executados em cada PDB.

## Validação

```sql
select app_name, app_version, app_status
  from dba_applications
 order by app_name;

select object_name, object_type, sharing, status
  from dba_objects
 where object_name = 'LAB_PRODUTOS';
```

Execute a validação em cada container e compare `APP_VERSION`/status. Depois teste DDL compartilhado, dados locais e consulta a partir do Application Root. `SHARING` exibido no dicionário comprova o tipo do link; não comprova que todos os tenants sincronizaram com sucesso.

## Operação e recuperação

Backup do `CDB$ROOT` não substitui backup dos dados locais das Application PDBs. Mudanças de aplicação devem ter rollback lógico ou backup compatível com o alcance. Se apenas um tenant falhar no sync, investigue-o antes de repetir toda a instalação no root.

Para relatórios globais, `CONTAINERS()` pode consultar objetos metadata-linked nas PDBs abertas. Para poda por chave, Container Map acrescenta roteamento. Esses mecanismos consomem a arquitetura criada aqui, mas não substituem INSTALL/SYNC/PATCH.

## Limpeza

Feche e remova primeiro as Application PDBs; depois remova o Application Root, sempre com nomes previamente validados.

## Referências oficiais

- [About Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/application-containers2.html)
- [Creating Application Containers](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-application-containers1.html)
