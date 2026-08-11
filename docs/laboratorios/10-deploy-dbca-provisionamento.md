---
title: Deploy — DBCA, templates e provisionamento silencioso
description: Pré-requisitos, CDB, OMF, templates, FRA, ARCHIVELOG e character set
---

# Deploy — DBCA, templates e provisionamento silencioso

## Decisões de provisionamento

| Necessidade | Opção |
|---|---|
| Criar CDB em modo silencioso | `-createAsContainerDatabase true` |
| Somente validar pré-requisitos | `-executePrereqs` |
| Criar muitas bases idênticas rapidamente | Template com datafiles (seed template) |
| Gerenciar nomes/localização dos arquivos | OMF / `DB_CREATE_FILE_DEST` |
| Recuperação e archivelogs centralizados | FRA + Enable Archiving |

## O que o DBCA decide

DBCA não apenas executa `CREATE DATABASE`. Ele prepara estrutura de diretórios, parâmetros, password file, listener/serviços, CDB, PDBs e scripts posteriores. Em modo silencioso, cada escolha precisa estar explícita ou vir de template; repetir um comando sem registrar essas decisões dificulta reproduzir o ambiente.

Separe quatro camadas antes de começar:

1. software e Oracle Home já instalados;
2. pré-requisitos de sistema operacional, memória, grupos e storage;
3. configuração da instância e do CDB;
4. criação das PDBs e serviços usados pelas aplicações.

`-executePrereqs` valida a camada de infraestrutura sem criar o banco. Ele não comprova que tamanho de SGA, FRA e datafiles atendem ao workload; capacidade continua sendo decisão do projeto.

## Test case 1 — validar sem criar banco

No host Oracle:

```bash
dbca -silent -executePrereqs -databaseConfigType SINGLE
```

Esse modo verifica o ambiente e não cria a base. Se o instalador reclamar que não há listener, configure ou inicie o listener antes de repetir.

Leia o relatório completo, inclusive checks marcados como fixable. Alguns fixups precisam de `root`, reinício de sessão ou ajuste de kernel. Não trate `-ignorePrereqFailure` como solução permanente; ele apenas força avanço com risco conhecido.

## Test case 2 — resposta mínima de uma CDB

Exemplo conceitual; ajuste paths, memória e senhas fora do arquivo:

```bash
dbca -silent -createDatabase \
  -gdbname LABCDB \
  -sid LABCDB \
  -createAsContainerDatabase true \
  -numberOfPDBs 1 \
  -pdbName LABPDB \
  -storageType FS \
  -useOMF true \
  -datafileDestination /u02/app/oracle \
  -recoveryAreaDestination /u03/app/oracle \
  -enableArchive true \
  -characterSet AL32UTF8
```

Nunca coloque a senha real no script versionado; use prompt, wallet ou mecanismo seguro da ferramenta.

Em automação real, confira as opções suportadas pela versão com `dbca -help`. Parâmetros mudam entre releases e alguns são condicionais ao tipo de storage, edição ou template. Mantenha response file sem segredos e injete credenciais por canal protegido no momento da execução.

## Test case 3 — confirmar o resultado

```sql
select name, cdb, log_mode, open_mode
  from v$database;

select name, open_mode
  from v$pdbs
 order by con_id;

select name, value
  from v$parameter
 where name in ('db_create_file_dest','db_recovery_file_dest');

select value database_charset
  from nls_database_parameters
 where parameter = 'NLS_CHARACTERSET';
```

Com OMF, os arquivos da seed e das novas PDBs ficam sob o destino administrado pelo Oracle. Sem OMF, a criação de PDB precisa de conversão/destino explícito quando não há padrão configurado.

OMF controla nomes e localização com base em destinos como `DB_CREATE_FILE_DEST`; o DBA administra o objeto lógico e o Oracle gera o nome físico. FRA centraliza arquivos de recuperação, mas só funciona com capacidade e política coerentes. Habilitar `ARCHIVELOG` aumenta recuperabilidade e também cria consumo contínuo que precisa de backup e limpeza controlada.

Depois da criação, valide também:

```sql
select name, value
  from v$parameter
 where name in ('control_files','compatible','local_undo_enabled');

select comp_name, version_full, status
  from dba_registry
 order by comp_name;
```

Confirme registro dos serviços com o listener e faça um teste usando exatamente o connect string que a aplicação utilizará. Conectar localmente como `SYSDBA` não valida resolução de nome, listener ou serviço da PDB.

## Templates

- Structure Only: guarda definição, sem copiar datafiles.
- Include Datafiles: seed template; entrega uma cópia pronta e é mais rápido para bases idênticas.

Template structure-only descreve parâmetros e estrutura, mas o DBCA ainda cria datafiles e executa a construção. Template com datafiles clona uma imagem preparada e acelera provisionamento, porém carrega decisões da origem. Antes de reutilizá-lo, revise character set, componentes, patches, caminhos, senhas e dados que não deveriam ser replicados.

## Character set

Escolha o character set na criação. Alterar o character set do CDB root depois é uma operação restrita e complexa; `AL32UTF8` é a escolha usual para workloads multilíngues e recursos modernos.

O character set do CDB influencia todas as PDBs e a representação de dados `CHAR`, `VARCHAR2` e `CLOB`. O national character set trata `NCHAR`, `NVARCHAR2` e `NCLOB` separadamente. Collation, idioma da sessão e território são configurações distintas; escolher `AL32UTF8` não define automaticamente ordenação linguística.

## Referências oficiais

- [Creating a CDB with DBCA](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/creating-and-configuring-an-oracle-database.html)
- [DBCA Command Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/dbca-command.html)
- [PDB Storage e OMF](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/overview-of-pdb-creation.html)
