---
title: Transporte, plug e clonagem de PDB
description: Compatibilidade, arquivos XML e PDB, COPY, MOVE, OMF, hot clone e snapshot copy
---

# Transporte, plug e clonagem de PDB

## O que você precisa entender

Uma PDB pode ser movida logicamente entre CDBs por `UNPLUG`/`PLUG` ou copiada por clonagem.

| Recurso | O que faz |
|---|---|
| XML | Guarda metadados e caminhos; os datafiles continuam separados |
| Arquivo `.pdb` | Pacote comprimido com XML e datafiles |
| `COPY` | Copia os arquivos; é o comportamento padrão |
| `MOVE` | Move os arquivos e remove a origem após o sucesso |
| `NOCOPY` | Reutiliza os arquivos onde estão |
| Hot clone | Clona uma PDB aberta em `READ WRITE`; exige Local Undo e `ARCHIVELOG` |
| Snapshot copy | Usa arquivos esparsos/snapshot copy-on-write do storage |

Antes do plug, confirme versão, `COMPATIBLE`, character set, opções instaladas e endian format. Um plug direto não converte datafiles entre plataformas com endian diferente.

## Como escolher o método

O ponto principal não é decorar a sintaxe, mas separar **movimentação de metadados**, **cópia física** e **continuidade de serviço**.

| Cenário | Método mais natural | Efeito na origem |
|---|---|---|
| Transferir uma PDB fechada e manter os arquivos originais | Unplug em XML + plug com `COPY` | Arquivos originais permanecem |
| Entregar um pacote único para transporte | Unplug em `.pdb` | Origem fica desconectada da CDB |
| Reutilizar datafiles já posicionados no destino | Plug com `NOCOPY` | Não há uma segunda cópia de segurança |
| Mover arquivos para outro diretório no mesmo storage | Plug com `MOVE` | Originais são removidos após o sucesso |
| Criar outra PDB sem desligar a origem | Hot clone | Origem continua `READ WRITE` quando os pré-requisitos são atendidos |
| Criar rapidamente clones descartáveis | Snapshot copy | Apenas blocos modificados ocupam espaço adicional |

O manifesto XML descreve a PDB, mas não carrega os datafiles. O arquivo `.pdb` é um archive que reúne manifesto e arquivos, facilitando transporte, porém exigindo espaço para gerar e extrair o pacote. Em ambos os casos, o destino precisa aceitar a configuração registrada no manifesto.

`COPY`, `MOVE` e `NOCOPY` tratam do destino físico dos arquivos, não da compatibilidade lógica. `COPY` é a escolha conservadora porque preserva a origem. `NOCOPY` deve ser usado somente quando os arquivos já estão no local definitivo e existe outra forma de recuperá-los; uma falha posterior deixa menos opções de retorno.

## Test case 1 — validar e plugar com XML

Execute como `SYS` em um ambiente descartável.

```sql
-- Na CDB de origem
alter pluggable database lab_src close immediate;
alter pluggable database lab_src unplug into '/tmp/lab_src.xml';
```

Copie o XML e os datafiles para o destino. Antes do plug:

```sql
set serveroutput on
declare
  l_ok boolean;
begin
  l_ok := dbms_pdb.check_plug_compatibility(
    pdb_descr_file => '/tmp/lab_src.xml',
    pdb_name       => 'LAB_DST');
  dbms_output.put_line(case when l_ok then 'COMPATIVEL' else 'INCOMPATIVEL' end);
end;
/

select type, message, status
  from pdb_plug_in_violations
 where name = 'LAB_DST'
 order by time;
```

Com OMF ativo, o destino é definido por `DB_CREATE_FILE_DEST`:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.xml'
  copy;

alter pluggable database lab_dst open;
```

Sem OMF e sem `PDB_FILE_NAME_CONVERT`, informe a conversão:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.xml'
  copy
  file_name_convert = ('/origem/lab_src/', '/destino/lab_dst/');
```

## Test case 2 — gerar um único arquivo transportável

```sql
alter pluggable database lab_src close immediate;
alter pluggable database lab_src
  unplug into '/tmp/lab_src.pdb';
```

O `.pdb` contém metadados e datafiles. No destino:

```sql
create pluggable database lab_dst
  using '/tmp/lab_src.pdb';
```

## Test case 3 — clone local

Consulte primeiro os pré-requisitos:

```sql
select log_mode from v$database;
select property_value
  from database_properties
 where property_name = 'LOCAL_UNDO_ENABLED';
```

Em `ARCHIVELOG` com Local Undo, a origem pode permanecer aberta:

```sql
create pluggable database lab_clone from certlab;
alter pluggable database lab_clone open;
```

Em `NOARCHIVELOG`, feche a origem e abra-a em `READ ONLY` antes do clone.

Durante um hot clone, o Oracle copia uma imagem consistente e usa redo para reproduzir as mudanças ocorridas enquanto a origem continua recebendo DML. Por isso Local Undo e `ARCHIVELOG` são relevantes: cada PDB mantém seu próprio undo e os archived logs preservam as mudanças necessárias até o término da cópia. Remover logs cedo demais pode interromper o processo.

Snapshot copy segue outro princípio. O clone compartilha blocos imutáveis com a origem e grava apenas diferenças, usando sparse files ou capacidade equivalente do storage. Ele economiza tempo e espaço, mas cria dependência tecnológica e operacional do snapshot base; não deve ser tratado como uma cópia independente para recuperação.

## Compatibilidade e abertura

`DBMS_PDB.CHECK_PLUG_COMPATIBILITY` retorna apenas verdadeiro ou falso. A explicação detalhada fica em `PDB_PLUG_IN_VIOLATIONS`, que pode mostrar avisos e erros sobre versão, componentes, parâmetros e opções instaladas. Um aviso nem sempre bloqueia a abertura; uma violação pendente pode exigir correção ou abertura em modo de upgrade.

Depois do plug, a primeira abertura completa a integração da PDB ao destino. Além de consultar `V$PDBS`, confirme:

- serviço registrado no listener;
- objetos inválidos e componentes do registry;
- destino real dos datafiles;
- violações ainda pendentes;
- backup inicial da nova PDB;
- `SAVE STATE`, caso a PDB deva reabrir automaticamente com a CDB.

## Como validar e limpar

```sql
select name, open_mode, restricted
  from v$pdbs
 where name in ('LAB_SRC','LAB_DST','LAB_CLONE');

alter pluggable database lab_clone close immediate;
drop pluggable database lab_clone including datafiles;
```

## Pontos de prova

- O destino não pode estar em release inferior à origem.
- Endian diferente exige migração/conversão suportada; plug direto não converte blocos.
- `DBMS_PDB.CHECK_PLUG_COMPATIBILITY` valida o manifesto antes do plug.
- Com OMF, o Oracle administra o destino dos arquivos.
- Hot clone: Local Undo + `ARCHIVELOG` + archived logs preservados durante a cópia.
- Snapshot copy depende de suporte a arquivos esparsos ou snapshot no storage.

## Referências oficiais

- [Plugging In an Unplugged PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/plugging-in-a-pdb.html)
- [Cloning a PDB](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/cloning-a-pdb.html)
- [Removing a PDB e arquivo `.pdb`](https://docs.oracle.com/en/database/oracle/oracle-database/26/multi/removing-a-pdb.html)
