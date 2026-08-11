---
title: Flashback, restore points e transporte cross-platform
description: Flashback Table, Flashback Drop, guaranteed restore points e conversão entre plataformas
---

# Flashback, restore points e transporte cross-platform

## Qual tecnologia resolve cada erro?

| Situação | Recurso |
|---|---|
| `DROP TABLE` acidental | Flashback Drop / recycle bin |
| DML incorreto sem mudança estrutural | Flashback Table |
| Reverter toda a base a um ponto conhecido | Flashback Database / restore point |
| Falha de instância | Instance recovery automática com roll forward e rollback |
| Transportar datafiles entre endian diferentes | RMAN `CONVERT` |

Essas tecnologias atuam em camadas diferentes. Flashback Table reconstrói linhas usando undo. Flashback Drop recupera um objeto preservado no recycle bin. Flashback Database reposiciona o conjunto de datafiles usando flashback logs e redo. Restore/recover parte de backups. Escolher uma tecnologia sem identificar o tipo e o alcance do erro pode aumentar a indisponibilidade.

## Dependências de Flashback

Flashback Query e Flashback Table dependem da retenção real de undo. `UNDO_RETENTION` é uma meta, não garantia absoluta quando o tablespace pode reutilizar extents por falta de espaço. Para uma janela previsível, dimensione o undo conforme o volume de alterações e acompanhe `V$UNDOSTAT`.

Flashback Database é independente do undo de linhas. Ele exige FRA e flashback logs; normalmente também opera com o banco em `ARCHIVELOG`. Um restore point normal é apenas um marcador de SCN e não garante que os logs continuarão disponíveis. Um guaranteed restore point obriga o Oracle a preservar o necessário para retornar ao ponto, mesmo se Flashback Database não estava habilitado continuamente, dentro das condições suportadas.

## Test case 1 — Flashback Table

Na PDB de laboratório, conceda ao usuário que executará o teste:

```sql
grant execute on sys.dbms_flashback to studylab;
```

Conectado como `STUDYLAB`:

```sql
create table lab_saldo (
  id    number primary key,
  saldo number not null
);

insert into lab_saldo values (1, 1000);
commit;

alter table lab_saldo enable row movement;

column scn_now new_value SCN_ANTES_DO_ERRO noprint
select dbms_flashback.get_system_change_number scn_now from dual;
```

Anote o SCN, altere o dado e recupere:

```sql
update lab_saldo set saldo = 10 where id = 1;
commit;

flashback table lab_saldo to scn &SCN_ANTES_DO_ERRO;
select * from lab_saldo;
```

Flashback Table depende de undo suficiente e não atravessa DDL estrutural, como `DROP COLUMN`, `TRUNCATE` ou mudança de partição.

`ENABLE ROW MOVEMENT` permite que o Oracle altere o rowid ao reconstruir versões anteriores. Antes de executar o flashback, use Flashback Query para conferir a imagem pretendida:

```sql
select *
  from lab_saldo as of scn &SCN_ANTES_DO_ERRO
 order by id;
```

O comando adquire locks e é transacional. Constraints, dependências e alterações estruturais posteriores ao ponto escolhido podem impedir a operação; por isso o teste de leitura histórica deve vir antes da reversão.

## Test case 2 — Flashback Drop

```sql
drop table lab_saldo;

select original_name, object_name, type
  from recyclebin
 where original_name = 'LAB_SALDO';

flashback table lab_saldo to before drop;
```

A tabela, constraints, triggers e a maioria dos índices retornam. Índices recuperados podem manter nomes `BIN$...` gerados pelo recycle bin.

O recycle bin trabalha no mesmo tablespace e pode ser purgado automaticamente sob pressão de espaço. `DROP ... PURGE`, `TRUNCATE` e objetos em tablespaces sem suporte adequado não fornecem o mesmo caminho de retorno. Flashback Drop também não substitui backup, pois a cópia reciclada reside no próprio banco.

## Test case 3 — guaranteed restore point

Antes de uma manutenção de laboratório:

```sql
create restore point grp_before_lab guarantee flashback database;

select name, guarantee_flashback_database, storage_size
  from v$restore_point;
```

Ao terminar:

```sql
drop restore point grp_before_lab;
```

Um guaranteed restore point não expira automaticamente. Se a FRA não conseguir preservar os flashback logs necessários, a instância pode parar para não quebrar a garantia.

Para retornar toda a base ao ponto, a operação é feita em estado `MOUNT`, seguida de `FLASHBACK DATABASE TO RESTORE POINT ...` e abertura conforme o fluxo indicado. Isso reverte todas as PDBs e transações da CDB, não somente um objeto. Em ambientes compartilhados, avalie o impacto global antes de escolher esse mecanismo.

## Instance recovery em uma frase

Após falha de energia, o Oracle reaplica redo (**roll forward**) e depois desfaz transações não confirmadas (**rolling back**). O banco pode abrir antes de todo rollback terminar.

Instance recovery não restaura arquivos perdidos e não usa backup para uma parada comum. Os processos de background aplicam redo dos online redo logs para recuperar blocos que estavam apenas em memória e, em seguida, desfazem transações sem commit usando undo. Media recovery é diferente: trata arquivo ausente, danificado ou antigo e normalmente combina restore com archived redo.

## Test case 4 — transporte cross-platform

Confira o endian:

```sql
select platform_id, platform_name, endian_format
  from v$transportable_platform
 order by platform_name;
```

Quando houver conversão suportada:

```text
CONVERT DATAFILE '/stage/users01.dbf'
  FROM PLATFORM 'Solaris[tm] OE (64-bit)'
  FORMAT '+DATA';
```

Em transporte incremental, compare `CHECKPOINT_CHANGE#` nos headers de origem e destino para saber se a cópia recebeu todos os incrementais.

Transportable Tablespaces move metadados e datafiles sem exportar todas as linhas. O conjunto precisa estar autocontido: índices, LOBs ou partições não podem depender de segmentos deixados fora. Valide com `DBMS_TTS.TRANSPORT_SET_CHECK` antes de colocar os tablespaces em `READ ONLY` e gerar o metadata export.

Se origem e destino tiverem endian format diferente, os blocos devem ser convertidos em plataforma suportada. A conversão pode acontecer na origem ou no destino, conforme método e versão. Character set, tipos de dados incompatíveis e versão continuam sendo verificações separadas; converter endian não resolve incompatibilidade lógica.

## Limpeza

```sql
drop table lab_saldo purge;
```

## Referências oficiais

- [Using Flashback Database and Restore Points](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/using-flasback-database-restore-points.html)
- [Performing Flashback and DBPITR](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/rman-performing-flashback-dbpitr.html)
- [FLASHBACK TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/FLASHBACK-TABLE.html)
