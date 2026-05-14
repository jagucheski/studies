# 📘 Resumo - PostgreSQL: Desempenho e Otimização

---

## 🗂️ Módulo 1 — Identificação de Gargalos e Análise de Carga

### Configuração do `postgresql.conf`
O primeiro passo é habilitar a extensão de monitoramento de queries:

```conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
pg_stat_statements.track_utility = on
```

### Views de monitoramento

**`pg_stat_statements`** — analisa as queries mais custosas:
```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, min_exec_time, max_exec_time, mean_exec_time
FROM pg_stat_statements
WHERE query LIKE 'SELECT * FROM tabela_de_clientes%';
```

**`pg_stat_activity`** — monitora conexões ativas em tempo real:
```sql
SELECT pid, usename, state, query, query_start
FROM pg_stat_activity
WHERE state = 'active';
```

**`pg_stat_database`** — visão geral de saúde do banco:
```sql
SELECT datname, numbackends, xact_commit, xact_rollback,
       blks_read, blks_hit, tup_returned, tup_fetched,
       tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database;
```

### Parâmetros de memória recomendados
```conf
shared_buffers = 4GB          -- cache principal do PostgreSQL (25% da RAM)
work_mem = 64MB               -- memória por operação de sort/hash
maintenance_work_mem = 512MB  -- para VACUUM, CREATE INDEX
effective_cache_size = 12GB   -- estimativa do cache total disponível
max_connections = 100
```

### Manutenção básica
```sql
VACUUM (VERBOSE);               -- limpa tuplas mortas
ANALYZE tabela_de_clientes;     -- atualiza estatísticas do planejador
REINDEX TABLE tabela_de_clientes; -- reconstrói índices corrompidos/inchados
```

---

## 🗂️ Módulo 2 — Índices

### Tipos de índices

| Tipo | Melhor uso | Exemplo |
|---|---|---|
| **B-Tree** | Intervalos (`>`, `<`, `BETWEEN`), padrão | `CREATE INDEX idx ON tabela (coluna)` |
| **Hash** | Igualdade exata (`=`) | `USING HASH` |
| **GIST** | Dados geográficos / geométricos | `USING GIST` |
| **SP-GiST** | Estruturas de dados não balanceadas | |
| **BRIN** | Colunas com correlação física (datas sequenciais) | |

```sql
-- B-Tree (padrão, intervalos)
CREATE INDEX idx_codigo_btree ON tabela_de_produtos (CODIGO_DO_PRODUTO);

-- Hash (igualdade)
CREATE INDEX idx_cpf_hash ON tabela_de_clientes USING HASH (CPF);

-- GIST (geográfico)
CREATE INDEX idx_geo ON tabela_de_clientes_geo USING GIST (geometria);
```

### Estratégias avançadas de índices

**Índice Parcial** — indexa apenas um subconjunto de linhas, menor e mais rápido:
```sql
CREATE INDEX idx_bairro_parcial ON tabela_de_clientes (BAIRRO)
WHERE BAIRRO = 'Centro';
```

**Índice Funcional** — indexa o resultado de uma função:
```sql
CREATE INDEX idx_nome_lower ON tabela_de_vendedores (LOWER(NOME));

-- Para funcionar, a query deve usar a mesma expressão:
SELECT * FROM tabela_de_vendedores WHERE LOWER(NOME) = 'victor';
```

**Índice Composto** — múltiplas colunas; a ordem importa:
```sql
CREATE INDEX idx_bairro_idade ON tabela_de_clientes (IDADE, BAIRRO);

-- Funciona para: IDADE + BAIRRO, ou só IDADE
-- NÃO usa o índice para: apenas BAIRRO isolado
```

### Monitoramento de índices
```sql
-- Ver uso dos índices
SELECT relname AS table_name, indexrelname AS index_name,
       idx_scan AS index_scans, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public';
```

### Manutenção de índices
```sql
REINDEX INDEX idx_bairro;       -- reconstrói um índice específico
REINDEX TABLE tabela_de_clientes; -- reconstrói todos os índices da tabela
REINDEX SCHEMA public;          -- reconstrói todos os índices do schema
VACUUM tabela_de_clientes;      -- libera espaço de tuplas mortas
ANALYZE tabela_de_clientes;     -- atualiza estatísticas
```

---

## 🗂️ Módulo 3 — Plano de Execução e Otimização de Queries

### EXPLAIN vs EXPLAIN ANALYZE

```sql
-- EXPLAIN: mostra o plano estimado (não executa)
EXPLAIN SELECT * FROM tabela_de_clientes WHERE BAIRRO = 'Centro';

-- EXPLAIN ANALYZE: executa e mostra tempo real
EXPLAIN ANALYZE SELECT * FROM tabela_de_clientes WHERE BAIRRO = 'Centro';

-- Formato JSON (mais detalhado, útil para ferramentas externas)
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;
```

**O que observar no plano:**
- `Seq Scan` em tabelas grandes → sinal de falta de índice
- Custo estimado alto → candidato a otimização
- Loops repetitivos → subquery correlacionada ineficiente
- `Index Scan` → índice sendo utilizado ✅

### CTE vs Subquery

CTEs (Common Table Expressions) podem reduzir drasticamente o custo em queries complexas:

```sql
-- Subquery correlacionada: custo 82.379.694 😱
SELECT NOME, (SELECT MAX(DATA_VENDA) FROM notas_fiscais WHERE CPF = c.CPF) AS ultima_compra
FROM tabela_de_clientes c;

-- CTE equivalente: custo 4.328 ✅
WITH ultima_venda AS (
    SELECT CPF, MAX(DATA_VENDA) AS ultima_compra
    FROM notas_fiscais
    GROUP BY CPF
)
SELECT c.NOME, uv.ultima_compra
FROM tabela_de_clientes c
JOIN ultima_venda uv ON c.CPF = uv.CPF;
```

Outro exemplo expressivo:
```sql
-- Subquery: custo 406.848.521 😱
SELECT nome_do_produto,
    (SELECT SUM(quantidade) FROM itens_notas_fiscais
     WHERE codigo_do_produto = p.codigo_do_produto) * 0.10 AS custo
FROM tabela_de_produtos p;

-- CTE: custo 14.764 ✅
WITH total_quantidade AS (
    SELECT codigo_do_produto, SUM(quantidade) AS quantidade
    FROM itens_notas_fiscais
    GROUP BY codigo_do_produto
)
SELECT p.nome_do_produto, tp.quantidade * 0.10 AS comissao
FROM tabela_de_produtos p
JOIN total_quantidade tp ON p.codigo_do_produto = tp.codigo_do_produto;
```

### CTE MATERIALIZED
Força o PostgreSQL a materializar (executar e armazenar) o resultado da CTE antes de usá-la:
```sql
WITH vendas_2022 AS MATERIALIZED (
    SELECT NF.CPF, SUM(INF.quantidade) AS quantidade
    FROM notas_fiscais NF
    JOIN itens_notas_fiscais INF ON NF.numero = INF.numero
    WHERE data_venda >= '2022-01-01'
    GROUP BY NF.CPF
)
SELECT c.NOME, v.quantidade
FROM tabela_de_clientes c
JOIN vendas_2022 v ON c.CPF = v.CPF;
```

### Filtragem antes do agrupamento
Mova filtros para o `WHERE` sempre que possível, em vez de depender só do `HAVING`:
```sql
-- Menos eficiente: custo 480
SELECT IDADE, COUNT(*) FROM tabela_de_clientes
GROUP BY IDADE HAVING COUNT(*) > 10;

-- Mais eficiente: custo 430
SELECT IDADE, COUNT(*) FROM tabela_de_clientes
WHERE IDADE <> 0
GROUP BY IDADE HAVING COUNT(*) > 10;
```

---

## 🗂️ Módulo 4 — Estrutura de Dados

### Normalização vs Desnormalização

Às vezes, **desnormalizar** (juntar tabelas relacionadas em uma só) pode melhorar performance ao eliminar JOINs:

```sql
-- Tabela desnormalizada (notas + itens juntos)
CREATE TABLE notas_fiscais_itens (
    CPF varchar(11) NOT NULL,
    MATRICULA varchar(5) NOT NULL,
    DATA_VENDA date NOT NULL,
    NUMERO serial NOT NULL,
    IMPOSTO real NOT NULL,
    CODIGO_DO_PRODUTO varchar(10) NOT NULL,
    QUANTIDADE int NOT NULL,
    PRECO real NOT NULL
);

INSERT INTO notas_fiscais_itens (...)
SELECT nf.CPF, nf.MATRICULA, nf.DATA_VENDA, nf.NUMERO,
       inf.CODIGO_DO_PRODUTO, inf.QUANTIDADE, inf.PRECO, nf.IMPOSTO
FROM notas_fiscais nf
JOIN itens_notas_fiscais inf ON nf.NUMERO = inf.NUMERO;
```

> ⚠️ Desnormalização troca espaço em disco por velocidade de leitura. Indicada para tabelas de leitura intensiva (relatórios, DW).

### Chaves Primárias e Estrangeiras

PKs criam automaticamente um índice B-Tree, com impacto enorme em JOINs:

```sql
-- Sem PK: custo 504
-- Com PK: custo 8 ✅
ALTER TABLE tabela_de_produtos ADD PRIMARY KEY (codigo_do_produto);

-- FKs ajudam o planejador a tomar decisões melhores de JOIN
ALTER TABLE tabela_de_clientes ADD CONSTRAINT unique_cpf UNIQUE (CPF);
ALTER TABLE notas_fiscais ADD CONSTRAINT fk_notas_fiscais_cpf
    FOREIGN KEY (CPF) REFERENCES tabela_de_clientes(CPF);
```

### Particionamento de Tabelas

Divide uma tabela grande em partes menores por range, melhorando consultas por período:

```sql
-- Tabela particionada por ano
CREATE TABLE notas_fiscais_part (
    CPF varchar(11) NOT NULL,
    MATRICULA varchar(5) NOT NULL,
    DATA_VENDA date,
    NUMERO int NOT NULL,
    IMPOSTO real NOT NULL
) PARTITION BY RANGE (data_venda);

-- Criar partição para cada ano
CREATE TABLE notas_fiscais_part_2022 PARTITION OF notas_fiscais_part
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
```

Resultado medido no curso:
```
-- Tabela normal:       custo 4325, tempo 55ms
-- Tabela particionada: custo 0,    tempo 0.01ms ✅
```

### Tipos de dados adequados

Usar o tipo correto ou adicionar restrições de unicidade melhora performance:

```sql
-- Sem unicidade: custo 5287
-- Com UNIQUE:    custo 8 ✅
ALTER TABLE notas_fiscais ADD COLUMN NUMERO_UNICO int;
ALTER TABLE notas_fiscais ADD CONSTRAINT UK_NUMERO_UNICO UNIQUE (NUMERO_UNICO);
```

---

## 🗂️ Módulo 5 — Projeto Prático (Caso Real)

O curso aplica todas as técnicas em uma consulta real com 5 tabelas e vários filtros, medindo o custo a cada passo:

| Técnica aplicada | Custo estimado | Melhoria |
|---|---|---|
| Consulta original (sem otimização) | **26.756** | — |
| + Índices nos filtros WHERE | **25.022** | 6% |
| + Chaves primárias nas tabelas | **3.937** | 🔥 85% |
| + CTE | 3.937 | sem ganho adicional |
| + Tabela desnormalizada | 16.729 | pior (JOIN a mais) |
| + Índices na tabela desnormalizada | 16.729 | sem ganho significativo |

**Conclusão do curso:** a técnica de maior impacto isolado foi a **definição de chaves primárias**, que reduziu o custo em 85% sozinha.

---

## 📌 Resumo das Técnicas por Impacto

| Técnica | Quando usar | Impacto |
|---|---|---|
| **Chaves primárias** | Sempre, em toda tabela | 🔥🔥🔥 Altíssimo |
| **CTEs no lugar de subqueries correlacionadas** | Subqueries com `IN` ou correlated | 🔥🔥🔥 Altíssimo |
| **Índices B-Tree** | Colunas usadas em filtros e JOINs | 🔥🔥 Alto |
| **Índices Hash** | Colunas usadas em igualdade | 🔥🔥 Alto |
| **Particionamento** | Tabelas com filtros por data/range | 🔥🔥 Alto |
| **ANALYZE** | Após grandes inserções/atualizações | 🔥 Médio |
| **Índices parciais** | Filtros repetitivos por valor fixo | 🔥 Médio |
| **Índices funcionais** | Queries com funções em colunas | 🔥 Médio |
| **Índices compostos** | Filtros multi-coluna frequentes | 🔥 Médio |
| **Tipos de dados corretos** | Colunas com restrição de unicidade | 🔥 Médio |
| **Desnormalização** | Relatórios / leitura intensiva | ⚠️ Situacional |
| **VACUUM** | Tabelas com muitas atualizações/deleções | 🔧 Manutenção |
| **REINDEX** | Índices corrompidos ou muito inchados | 🔧 Manutenção |
