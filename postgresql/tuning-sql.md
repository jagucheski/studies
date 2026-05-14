# 🔧 Principais Estratégias de Tuning SQL

Tunar consultas SQL envolve otimizar comandos para melhorar o desempenho, reduzir o tempo de execução e o consumo de recursos do banco de dados. As técnicas cobrem desde boas práticas de escrita até estratégias avançadas de estrutura e indexação.

---

## 1. 📝 Práticas Fundamentais de Escrita

**Evite `SELECT *`**
Selecione apenas as colunas necessárias. Reduz tráfego de rede e sobrecarga de I/O.
```sql
-- ❌ Evite
SELECT * FROM pedidos;

-- ✅ Prefira
SELECT id, cliente, valor FROM pedidos;
```

**Use `LIMIT` / `TOP`**
Se precisar apenas de algumas linhas, limite o resultado desde a query:
```sql
SELECT id, nome FROM clientes LIMIT 10;
```

**Evite funções em colunas no `WHERE` (non-sargable)**
Funções aplicadas sobre colunas impedem o uso de índices:
```sql
-- ❌ Impede uso do índice
WHERE YEAR(data_venda) = 2024

-- ✅ Permite uso do índice
WHERE data_venda >= '2024-01-01' AND data_venda < '2025-01-01'
```

**Cuidado com `LIKE` com curinga no início**
O `%` no início força um table scan completo:
```sql
-- ❌ Table scan completo
WHERE nome LIKE '%silva'

-- ✅ Pode usar índice
WHERE nome LIKE 'silva%'
```

**Prefira `JOIN` a subqueries correlacionadas**
JOINs geralmente são mais eficientes que subconsultas que executam linha a linha:
```sql
-- ❌ Subquery correlacionada: executa N vezes
SELECT nome,
    (SELECT MAX(data_venda) FROM pedidos WHERE cpf = c.cpf) AS ultima_compra
FROM clientes c;

-- ✅ JOIN: executa uma vez
SELECT c.nome, MAX(p.data_venda) AS ultima_compra
FROM clientes c
JOIN pedidos p ON c.cpf = p.cpf
GROUP BY c.nome;
```

---

## 2. 🗂️ Indexação e Estrutura

**Crie índices nas colunas de JOIN (FKs)**
Colunas usadas em JOINs sem índice forçam Seq Scan em toda a tabela:
```sql
CREATE INDEX idx_pedidos_cpf ON pedidos (cpf);
CREATE INDEX idx_itens_numero ON itens_pedidos (numero_pedido);
```

**Não abuse de índices**
Índices têm custo em operações de escrita. Evite em:
- Tabelas pequenas (Seq Scan pode ser mais rápido)
- Colunas com poucos valores distintos (`boolean`, `status` com 2-3 valores)
- Tabelas com volume alto de `INSERT`/`UPDATE`

**Analise o plano de execução sempre**
```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT ...;
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;

-- MySQL
EXPLAIN SELECT ...;
```

O que observar:
- `Seq Scan` em tabelas grandes → falta de índice
- `Nested Loop` com muitas iterações → subquery ineficiente
- Custo estimado muito alto → candidato a reescrita

---

## 3. ⚙️ Técnicas Avançadas

**Evite conversão implícita de tipos**
Comparar tipos diferentes força o banco a converter coluna a coluna, invalidando índices:
```sql
-- ❌ Conversão implícita (cpf é VARCHAR, comparando com número)
WHERE cpf = 12345678900

-- ✅ Tipos iguais
WHERE cpf = '12345678900'
```

**Use `EXISTS` em vez de `IN` para subconsultas grandes**
`EXISTS` para assim que encontra o primeiro resultado; `IN` carrega todos os valores:
```sql
-- ❌ IN: carrega todos os CPFs da subconsulta
WHERE cpf IN (SELECT cpf FROM pedidos WHERE data_venda > '2024-01-01')

-- ✅ EXISTS: para no primeiro match
WHERE EXISTS (
    SELECT 1 FROM pedidos
    WHERE pedidos.cpf = clientes.cpf
    AND data_venda > '2024-01-01'
)
```

**CTEs no lugar de subqueries correlacionadas**
CTEs executam uma única vez e reutilizam o resultado:
```sql
WITH ultima_compra AS (
    SELECT cpf, MAX(data_venda) AS data
    FROM pedidos
    GROUP BY cpf
)
SELECT c.nome, uc.data
FROM clientes c
JOIN ultima_compra uc ON c.cpf = uc.cpf;
```

**Particionamento para tabelas gigantescas**
Divide a tabela em partes menores por range ou lista. Queries com filtro de data passam a ler apenas a partição relevante:
```sql
CREATE TABLE pedidos_part (
    cpf varchar(11),
    data_venda date,
    valor numeric(10,2)
) PARTITION BY RANGE (data_venda);

CREATE TABLE pedidos_2024 PARTITION OF pedidos_part
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Defina Chaves Primárias e Estrangeiras**
PKs criam índices automaticamente e ajudam o planejador de queries:
```sql
ALTER TABLE clientes ADD PRIMARY KEY (cpf);
ALTER TABLE pedidos ADD CONSTRAINT fk_cpf
    FOREIGN KEY (cpf) REFERENCES clientes(cpf);
```

**Filtre antes de agrupar**
Mova condições para o `WHERE` sempre que possível, em vez de depender só do `HAVING`:
```sql
-- ❌ Menos eficiente: agrupa tudo, depois filtra
SELECT idade, COUNT(*) FROM clientes
GROUP BY idade
HAVING COUNT(*) > 10;

-- ✅ Mais eficiente: filtra antes de agrupar
SELECT idade, COUNT(*) FROM clientes
WHERE idade > 0
GROUP BY idade
HAVING COUNT(*) > 10;
```

---

## 4. 🛠️ Ferramentas de Suporte

| Banco | Ferramenta |
|---|---|
| **PostgreSQL** | `EXPLAIN ANALYZE`, `pg_stat_statements`, `pg_stat_activity` |
| **SQL Server** | Database Engine Tuning Advisor, Query Store |
| **Oracle** | SQL Tuning Advisor |
| **Azure SQL** | Ajuste automático — cria e descarta índices automaticamente |
| **MySQL** | `EXPLAIN`, Performance Schema |

---

## 📊 Resumo por impacto

| Estratégia | Impacto | Complexidade |
|---|---|---|
| Chaves primárias e estrangeiras | 🔥🔥🔥 | Baixa |
| CTE no lugar de subquery correlacionada | 🔥🔥🔥 | Baixa |
| Índices nas colunas de filtro e JOIN | 🔥🔥🔥 | Baixa |
| Particionamento de tabelas | 🔥🔥🔥 | Média |
| Evitar funções no WHERE (non-sargable) | 🔥🔥 | Baixa |
| `EXISTS` em vez de `IN` | 🔥🔥 | Baixa |
| Evitar `SELECT *` | 🔥🔥 | Baixa |
| Evitar conversão implícita de tipos | 🔥🔥 | Baixa |
| `JOIN` em vez de subquery correlacionada | 🔥🔥 | Baixa |
| Filtrar no WHERE antes do GROUP BY | 🔥🔥 | Baixa |
| Evitar `LIKE '%termo'` | 🔥 | Baixa |
| Não abusar de índices em tabelas pequenas | 🔧 | Baixa |
| Analisar plano de execução | 🔧 diagnóstico | Média |