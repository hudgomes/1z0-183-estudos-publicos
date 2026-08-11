---
title: RMAN — validação, corrupção e recuperação
description: VALIDATE, Data Recovery Advisor, block recovery, restore, control file, PDB e incarnations
---

# RMAN — validação, corrupção e recuperação

## Escolha o comando pelo objetivo

| Objetivo | Comando |
|---|---|
| Ler datafiles e detectar corrupção | `VALIDATE DATABASE` ou `BACKUP VALIDATE` |
| Confirmar que backups podem ser restaurados | `RESTORE DATABASE VALIDATE` |
| Diagnosticar falha conhecida | `LIST FAILURE`, `ADVISE FAILURE` |
| Ver o que o reparo fará | `REPAIR FAILURE PREVIEW` |
| Corrigir blocos isolados | `RECOVER ... BLOCK` |
| Recuperar uma PDB | Fechar PDB, `RESTORE` e `RECOVER PLUGGABLE DATABASE` |

## O que cada validação realmente comprova

`VALIDATE DATABASE` lê os arquivos atuais do banco. `BACKUP VALIDATE` percorre o mesmo caminho de leitura de um backup, mas descarta a saída; é útil para testar legibilidade e canais sem gerar peças. `RESTORE ... VALIDATE` lê as peças existentes e confirma se o RMAN consegue localizar e processar o material necessário ao restore.

`CHECK LOGICAL` acrescenta verificações de consistência intra-bloco além de checksum e danos físicos. Isso aumenta o trabalho e pode revelar corrupção lógica que uma validação física não encontra. Nenhum desses comandos prova sozinho que a aplicação está consistente: eles trabalham com blocos e backups, não com regras de negócio.

Validação também depende do escopo. É possível validar banco, PDB, tablespace, datafile, archivelogs ou backupsets. Escolha o menor escopo que responda ao risco sem deixar arquivos críticos fora do teste.

## Test case 1 — validação sem gerar backup

No RMAN:

```text
VALIDATE DATABASE CHECK LOGICAL;
RESTORE DATABASE VALIDATE;
```

Depois, no SQL:

```sql
select file#, block#, blocks, corruption_type
  from v$database_block_corruption
 order by file#, block#;
```

A view é atualizada por leituras que detectam corrupção, como RMAN validation/backup e alguns full scans.

Quando uma validação posterior lê com sucesso os blocos reparados, as entradas podem ser removidas da view. Portanto, ela representa corrupção conhecida pelo banco, não um inventário eterno de todos os problemas já ocorridos.

Para validar apenas os arquivos do tablespace:

```text
VALIDATE TABLESPACE USERS CHECK LOGICAL;
```

## Test case 2 — fluxo do Data Recovery Advisor

```text
LIST FAILURE;
ADVISE FAILURE;
REPAIR FAILURE PREVIEW;
REPAIR FAILURE;
```

O DRA é apropriado para falhas físicas diagnosticáveis. Corrupções lógicas que exigem Flashback Database não são seu caso típico de reparo automático.

`LIST FAILURE` mostra falhas abertas; `ADVISE FAILURE` calcula opções considerando backups e impacto; `REPAIR FAILURE PREVIEW` exibe os comandos antes de executá-los. O reparo pode envolver indisponibilidade ou perda de dados, por isso o preview e um backup válido continuam indispensáveis mesmo quando o advisor oferece automação.

## Test case 3 — recuperação de bloco

```text
RECOVER DATAFILE 7 BLOCK 12345;
```

Durante block media recovery o datafile pode permanecer online; apenas os blocos afetados ficam indisponíveis. É necessário backup utilizável e redo suficiente, normalmente com a base em `ARCHIVELOG`.

Block media recovery é adequado quando a corrupção está isolada. Se muitos blocos ou a estrutura do datafile estiverem comprometidos, restaurar e recuperar o arquivo inteiro tende a ser mais seguro. Para blocos ainda legíveis em um standby, Active Data Guard pode permitir reparo automático conforme a configuração.

## Test case 4 — datafile e PDB

Para um datafile não crítico:

```text
SQL 'ALTER DATABASE DATAFILE 7 OFFLINE';
RESTORE DATAFILE 7;
RECOVER DATAFILE 7;
SQL 'ALTER DATABASE DATAFILE 7 ONLINE';
```

Para uma PDB:

```text
SQL 'ALTER PLUGGABLE DATABASE CERTLAB CLOSE IMMEDIATE';
RESTORE PLUGGABLE DATABASE CERTLAB;
RECOVER PLUGGABLE DATABASE CERTLAB;
SQL 'ALTER PLUGGABLE DATABASE CERTLAB OPEN';
```

`RESTORE` devolve uma cópia física a partir do backup. `RECOVER` aplica redo para avançá-la até o ponto desejado. Confundir as duas fases resulta em arquivo antigo, ainda inconsistente com o restante do banco. Em recuperação completa, aplique todo o redo disponível; em recuperação incompleta, defina deliberadamente SCN, horário ou sequência e espere abertura com `RESETLOGS` quando exigida.

## Control file, incarnation e Active Duplicate

- Após restaurar todos os control files, monte, recupere e abra com `RESETLOGS` quando o cenário exigir.
- Para PITR anterior ao último `RESETLOGS`, selecione a incarnation ancestral correta com `RESET DATABASE TO INCARNATION`.
- Active Duplicate exige conexão de rede com a auxiliar; o serviço da auxiliar deve estar disponível mesmo quando ela está `NOMOUNT`, normalmente por registro estático no listener.
- `DB_FILE_NAME_CONVERT` ajuda quando a hierarquia de diretórios difere no destino.

SPFILE e control file exigem atenção especial porque são necessários antes de o banco conhecer toda a estrutura. Quando o control file é restaurado de backup, o RMAN pode precisar de `SET DBID`, catálogo das peças e montagem antes da recuperação. O control file restaurado também pode não conhecer backups criados depois dele.

Uma incarnation começa após `OPEN RESETLOGS`. Os backups de incarnations anteriores não desaparecem, mas uma recuperação para um ponto ancestral precisa selecionar a ramificação correta. `LIST INCARNATION` permite enxergar essa árvore antes de redefinir a incarnation corrente.

## Segurança antes de recuperar

```text
LIST BACKUP OF DATABASE SUMMARY;
LIST INCARNATION OF DATABASE;
REPORT SCHEMA;
RESTORE DATABASE PREVIEW;
```

Nunca use arquivos reais de `SYSTEM`/`SYSAUX` para provocar falha em um laboratório. Use uma PDB e tablespace descartáveis.

## Referências oficiais

- [Validating Database Files and Backups](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/validating-database-files-backups.html)
- [Backup and Recovery User's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/26/bradv/toc.htm)
- [VALIDATE — RMAN Reference](https://docs.oracle.com/en/database/oracle/oracle-database/26/rcmrf/VALIDATE.html)
