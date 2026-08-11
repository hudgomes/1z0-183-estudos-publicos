---
title: Relocação online de PDB com indisponibilidade mínima
description: Cópia online, database link, cutover e validação da PDB no destino
---

# Relocação online de PDB com indisponibilidade mínima

## Como funciona

PDB Relocation move uma PDB para outra CDB por meio de um database link. A origem pode permanecer em `READ WRITE` durante a cópia. O cutover ocorre quando a PDB de destino é aberta.

O fluxo copia data blocks, undo e redo, aplica as mudanças restantes e integra a PDB ao destino. Ele é diferente de cold clone e de unplug/plug, que exigem uma janela maior com a origem fechada.

## Fases e indisponibilidade

Relocation possui uma fase longa de cópia e uma fase curta de transição. Durante a cópia, a source PDB continua atendendo leitura e escrita. O Oracle transfere a imagem inicial e acompanha mudanças por redo. A indisponibilidade concentra-se no cutover, quando o destino aplica o restante, abre a PDB e a origem deixa de ser o local ativo.

Isso reduz downtime, mas não elimina planejamento. Quanto maior a taxa de alteração, maior o volume de redo a transportar e aplicar. Rede lenta, archived logs removidos cedo ou espaço insuficiente podem prolongar ou interromper o processo.

Relocation **move** a identidade operacional da PDB. Remote clone cria outra PDB e mantém a origem. Unplug/plug transporta uma PDB fechada. Data Guard mantém uma réplica continuamente protegida e oferece transições de papel; não é o mesmo fluxo de migração única.

## Pré-requisitos

- CDBs compatíveis e com Local Undo.
- Conectividade Oracle Net entre origem e destino.
- Usuário comum de clone com privilégios apropriados.
- Nome de serviço e database link validados.
- Espaço e destino de arquivos definidos; OMF simplifica a criação.
- Serviços da PDB planejados para o cutover.

Origem e destino precisam atender compatibilidade de versão, endian, character set, opções e patch. Local Undo e logging suportado permitem capturar alterações por PDB. O usuário do database link precisa dos privilégios exatos para clone/relocation, e a credencial deve ser protegida fora do documento.

Teste o connect identifier do host de destino e não somente do notebook do DBA. O processo servidor é quem abre o link e transfere dados. Com OMF, o destino físico fica mais simples; sem OMF, configure `FILE_NAME_CONVERT` ou parâmetros equivalentes antes de criar.

## Test case — preparar a origem

No root da origem:

```sql
create user c##lab_clone identified by "<SENHA_FORTE_EXCLUSIVA>"
  container=all;

grant create session, create pluggable database
  to c##lab_clone container=all;

select name, open_mode
  from v$pdbs
 where name = 'LAB_SOURCE_PDB';
```

A origem deve permanecer `READ WRITE` durante a fase de cópia.

Gere atividade controlada em uma tabela de laboratório e registre SCN/contagem antes e durante a cópia. Isso permite demonstrar que DML confirmado enquanto a origem estava aberta aparece no destino depois do cutover.

## Preparar o destino

No root da CDB de destino:

```sql
create database link source_cdb_link
  connect to c##lab_clone identified by "<SENHA_FORTE_EXCLUSIVA>"
  using 'SOURCE_CDB_SERVICE';

select * from dual@source_cdb_link;
```

Só continue depois que a consulta pelo link funcionar.

Uma consulta a `DUAL` valida rede, serviço e autenticação básica, mas não todos os privilégios. Confirme também que o usuário é comum no escopo exigido e que a source PDB aparece pelo link. Falha de privilégio descoberta depois de copiar parte dos dados aumenta tempo de retorno.

## Executar a relocação

```sql
create pluggable database lab_target_pdb
  from lab_source_pdb@source_cdb_link
  relocate availability normal;
```

Durante a cópia, acompanhe:

```sql
select opname, sofar, totalwork, units, elapsed_seconds
  from v$session_longops
 where totalwork > 0
   and sofar < totalwork
 order by start_time;
```

`V$SESSION_LONGOPS` mostra operações elegíveis, mas nem toda fase aparece com progresso linear. Acompanhe também alert logs das duas CDBs, espaço de storage, taxa de redo e sessões do database link. Não reinicie listeners ou remova archived logs enquanto o processo depende deles.

Conclua o cutover:

```sql
alter pluggable database lab_target_pdb open read write;
alter pluggable database lab_target_pdb save state;
```

A abertura do destino executa o cutover. Planeje drain ou reconexão das sessões, atualização/ativação de serviços e teste de escrita. Aplicações que usam serviço associado corretamente conseguem mudar com menos alteração de configuração que aplicações presas ao hostname ou SID.

## Validação

No destino:

```sql
select name, open_mode, restricted
  from v$pdbs
 where name = 'LAB_TARGET_PDB';

select pdb_name, status
  from cdb_pdbs
 where pdb_name = 'LAB_TARGET_PDB';
```

Valide também serviços, conexões da aplicação, objetos inválidos, alert log e backup da nova PDB.

Compare os dados gerados durante a cópia, usuários, jobs, database links, diretórios, ACLs e dependências externas. Alguns objetos residem na PDB, mas apontam para recursos que não foram movidos, como filesystem, wallet ou endpoint de rede.

Depois da validação, confirme o estado da PDB de origem e os artefatos registrados pelo processo antes de limpar qualquer arquivo. Mantenha backup e plano de retorno até aceitar formalmente o destino.

## AVAILABILITY NORMAL e MAX

`NORMAL` é usado quando os listeners pertencem a uma rede comum. `MAX` atende topologias com redes de listener isoladas e exige configuração compatível. Escolha pelo desenho de rede, não apenas pelo nome da opção.

Availability descreve como conexões e listeners dos dois ambientes participam da transição. `MAX` não significa genericamente “mais rápido”; ele atende um desenho específico para maximizar continuidade quando as redes de listener não são compartilhadas. Use somente com todos os pré-requisitos e serviços configurados conforme a documentação.

## Critérios de sucesso

- origem permaneceu `READ WRITE` durante a cópia;
- alterações confirmadas durante o processo existem no destino;
- cutover ocorreu dentro da janela medida;
- serviço de escrita aponta apenas para o destino;
- PDB abriu sem violações pendentes relevantes;
- novo backup foi criado e o retorno foi testado ou documentado.

## Referências oficiais

- [Relocating a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/relocating-a-pdb.html)
- [CREATE PLUGGABLE DATABASE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/CREATE-PLUGGABLE-DATABASE.html)
