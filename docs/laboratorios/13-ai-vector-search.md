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

Embedding é uma representação numérica na qual proximidade procura refletir semelhança semântica. A dimensão, o tipo numérico e a forma de normalização pertencem ao modelo. Vetores produzidos por modelos diferentes não devem ser misturados na mesma comparação apenas porque possuem a mesma dimensão.

O banco não interpreta sozinho o significado de cada posição. Ele armazena e compara vetores com uma métrica. O pipeline completo inclui preparação do texto, escolha do modelo, geração do embedding, persistência, consulta e avaliação da qualidade dos resultados.

## Métricas de distância

| Métrica | Interpretação comum | Observação |
|---|---|---|
| `COSINE` | Compara o ângulo entre vetores | Útil quando direção importa mais que magnitude |
| `DOT` | Produto interno | Frequentemente usado com vetores normalizados |
| `EUCLIDEAN` | Distância geométrica L2 | Considera direção e magnitude |
| `MANHATTAN` | Soma das diferenças absolutas | Métrica L1 |

Menor valor de distância representa maior proximidade. Para produto interno, confira a convenção da função/modelo. A métrica da consulta deve ser a mesma usada para avaliar o modelo e criar o índice; trocar métrica pode mudar a ordem dos resultados e impedir uso do índice.

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

`FETCH EXACT` força avaliação exata e fornece referência para medir qualidade. Em poucos dados, a varredura exata pode ser mais rápida que manter um índice. Em volume grande, ela compara o vetor de consulta com todas as linhas elegíveis, elevando CPU e latência.

Use filtros relacionais para reduzir candidatos quando eles representam regras reais, como idioma, tenant, categoria ou período. O filtro não deve ser inventado apenas para acelerar, pois pode excluir o conteúdo semanticamente mais próximo.

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

HNSW constrói um grafo de vizinhança na Vector Pool e favorece baixa latência, consumindo memória para a estrutura. IVF divide o espaço em partições vetoriais e consulta subconjuntos mais prováveis; tende a oferecer outro equilíbrio entre construção, memória e precisão. O melhor índice depende de volume, frequência de atualização, latência e recursos disponíveis.

Busca aproximada troca parte do recall por desempenho. `TARGET ACCURACY` orienta quantos candidatos ou caminhos serão examinados; valor maior geralmente aumenta trabalho e aproxima o resultado do exact search. Meça recall comparando um conjunto de consultas aproximadas com o resultado exato, não apenas pelo tempo.

Antes de criar HNSW, confira a Vector Pool:

```sql
select name, value
  from v$parameter
 where name = 'vector_memory_size';

select index_name, index_type, status
  from user_indexes
 where table_name = 'LAB_PRODUTO_VECTOR';
```

Sem memória vetorial configurada, o laboratório exato continua válido, mas a criação ou população do índice in-memory não demonstra o cenário completo.

## Embeddings dentro do banco

Modelos importados no banco usam o formato **ONNX**. Depois, `VECTOR_EMBEDDING` transforma texto em vetor:

```sql
select vector_embedding(<MODELO_ONNX> using 'texto para converter' as data)
  from dual;
```

O modelo define dimensão e formato esperados; não invente esses valores na tabela.

O modelo ONNX é importado como objeto no banco e precisa ser compatível com as operações suportadas. A função `VECTOR_EMBEDDING` executa inferência perto dos dados, reduzindo movimentação de texto. Isso não elimina governança: versão do modelo, tokenizer, tamanho máximo de entrada e pré-processamento precisam ser registrados para gerar vetores comparáveis.

## Quantização e hybrid search

- `INT8` reduz o espaço do vetor em comparação com `FLOAT32`, com possível perda de precisão.
- Hybrid search combina similaridade vetorial com filtros/predicados relacionais na mesma consulta.

Quantização reduz a precisão numérica de cada componente para economizar memória e acelerar operações. Ela pode alterar o ranking, por isso deve ser avaliada com dataset representativo. Armazenar `FLOAT32`, `FLOAT64`, `INT8` ou vetor binário é uma decisão de modelo e custo, não apenas de sintaxe.

```sql
select id, descricao
  from lab_produto_vector
 where id >= 2
 order by vector_distance(embedding, :query_vector, cosine)
fetch first 5 rows only;
```

Hybrid search também pode combinar busca textual e vetorial, usando relevância lexical para termos exatos e similaridade para significado. Em aplicações reais, filtros de autorização devem ser aplicados junto da busca; proximidade vetorial nunca concede acesso a uma linha.

## Como validar o laboratório

Execute primeiro o exact search e registre ordem e distância. Depois crie o índice, use `FETCH APPROX`, confira o plano com `DBMS_XPLAN` e compare resultados. Teste ainda uma métrica incompatível para observar que o índice pode deixar de ser escolhido. A validação deve considerar latência, plano, recall e consumo da Vector Pool.

## Limpeza

```sql
drop table lab_produto_vector purge;
```

## Referências oficiais

- [Oracle AI Vector Search — índice e busca completa](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/sql-quick-start-using-vector-embedding-model-uploaded-database.html)
- [Busca vetorial exata](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/perform-exact-similarity-search.html)
- [Categorias HNSW e IVF](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/manage-different-categories-vector-indexes.html)
