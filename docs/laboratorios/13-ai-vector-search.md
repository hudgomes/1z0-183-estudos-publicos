---
title: Oracle AI Vector Search na prática
description: VECTOR, métricas, busca exata, HNSW, quantização, hybrid search e embeddings ONNX
---

# Oracle AI Vector Search na prática

## Modelo mental

1. Transforme o conteúdo em embedding.
2. Armazene-o em uma coluna `VECTOR` com dimensão e formato conhecidos.
3. Compare o vetor de consulta com os vetores armazenados.
4. Use busca exata para precisão total ou índice vetorial para escala.

## Test case 1 — tabela e busca exata

```sql
create table lab_produto_vector (
  id        number primary key,
  descricao varchar2(200),
  embedding vector(3, float32)
);

insert into lab_produto_vector values
  (1, 'Banco de dados', vector('[1,0,0]', 3, float32));
insert into lab_produto_vector values
  (2, 'Rede de computadores', vector('[0.8,0.2,0]', 3, float32));
insert into lab_produto_vector values
  (3, 'Culinária', vector('[0,0,1]', 3, float32));
commit;

select id, descricao,
       vector_distance(embedding,
                       vector('[1,0.1,0]', 3, float32),
                       cosine) distancia
  from lab_produto_vector
 order by distancia
 fetch exact first 2 rows only;
```

Menor distância significa maior similaridade. `COSINE` é a métrica padrão quando não há outra definida; use a métrica empregada pelo modelo de embedding.

## Test case 2 — índice HNSW

Para volume representativo e Vector Pool configurado:

```sql
create vector index lab_produto_hnsw_ix
    on lab_produto_vector(embedding)
  organization inmemory neighbor graph
  distance cosine
  with target accuracy 95;
```

Consulta aproximada:

```sql
select id, descricao
  from lab_produto_vector
 order by vector_distance(
            embedding,
            vector('[1,0.1,0]', 3, float32),
            cosine)
 fetch approx first 2 rows only with target accuracy 90;
```

HNSW é `INMEMORY NEIGHBOR GRAPH`; IVF é `NEIGHBOR PARTITIONS`. Se a métrica da consulta divergir da métrica do índice, o índice não é usado e a busca pode se tornar exata.

## Embeddings dentro do banco

Modelos importados no banco usam o formato **ONNX**. Depois, `VECTOR_EMBEDDING` transforma texto em vetor:

```sql
select vector_embedding(<MODELO_ONNX> using 'texto para converter' as data)
  from dual;
```

O modelo define dimensão e formato esperados; não invente esses valores na tabela.

## Quantização e hybrid search

- `INT8` reduz o espaço do vetor em comparação com `FLOAT32`, com possível perda de precisão.
- Hybrid search combina similaridade vetorial com filtros/predicados relacionais na mesma consulta.

```sql
select id, descricao
  from lab_produto_vector
 where id >= 2
 order by vector_distance(embedding, :query_vector, cosine)
 fetch first 5 rows only;
```

## Limpeza

```sql
drop table lab_produto_vector purge;
```

## Referências oficiais

- [Oracle AI Vector Search — índice e busca completa](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/sql-quick-start-using-vector-embedding-model-uploaded-database.html)
- [Busca vetorial exata](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/perform-exact-similarity-search.html)
- [Categorias HNSW e IVF](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/manage-different-categories-vector-indexes.html)
