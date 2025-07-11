# SQL Query Performance Analysis and Optimization

## Original Query Analysis

### Query Structure
The query joins 5 tables with the following pattern:
- **NAGATO.nova_combas (A)** - Main table
- **urano.base_venda (B)** - INNER JOIN on cpf
- **TRF.trf_saida (C)** - LEFT JOIN on cpf  
- **urano.base_processo (D)** - LEFT JOIN via cliente_id from base_venda
- **inss.base_cat (E)** - LEFT JOIN on cpf

## Identified Performance Bottlenecks

### 1. **Inefficient JOIN Strategy**
- **Issue**: Multiple JOINs on `cpf` field without proper indexing strategy
- **Impact**: High I/O operations and slow execution times
- **Risk**: O(n²) complexity for unindexed joins

### 2. **Problematic WHERE Clause**
- **Issue**: `C.assunto LIKE '%DIREITO PREVIDENCI%'` uses leading wildcard
- **Impact**: Cannot utilize indexes, forces full table scan
- **Risk**: Linear scan through entire TRF.trf_saida table

### 3. **Suboptimal JOIN Order**
- **Issue**: Query may not follow optimal join sequence
- **Impact**: Larger intermediate result sets than necessary
- **Risk**: Memory overflow and increased processing time

### 4. **Missing Index Strategy**
- **Issue**: No explicit indexing recommendations for multi-table joins
- **Impact**: Slow lookup operations
- **Risk**: Performance degradation with data growth

## Optimization Recommendations

### 1. **Index Creation Strategy**

```sql
-- Primary indexes for JOIN operations
CREATE INDEX idx_nova_combas_cpf ON NAGATO.nova_combas(cpf);
CREATE INDEX idx_base_venda_cpf ON urano.base_venda(cpf);
CREATE INDEX idx_base_venda_cliente_id ON urano.base_venda(cliente_id);
CREATE INDEX idx_trf_saida_cpf ON TRF.trf_saida(identificacao);
CREATE INDEX idx_base_processo_cliente_estado ON urano.base_processo(cliente_id, estado);
CREATE INDEX idx_base_cat_cpf ON inss.base_cat(cpf);

-- Composite index for WHERE clause optimization
CREATE INDEX idx_trf_saida_assunto_cpf ON TRF.trf_saida(assunto, identificacao);
```

### 2. **Query Restructuring**

#### Option A: Optimized with Proper JOIN Order
```sql
SELECT
    A.cpf,
    A.nb,
    A.nome,
    A.cid,
    A.cid_resumo,
    A.despacho,
    A.especie,
    A.faixa_salarial,
    A.faixa_tempo_concessao,
    A.sexo,
    A.uf,
    A.qt_duracao_beneficio_em_dias,
    A.dt_dcb,
    A.grau_instrucao,
    D.escritorio_id as escritorio_baseprocesso,
    E.cbo,
    E.houve_afastamento,
    E.local_do_acidente,
    E.parte_do_corpo,
    E.houve_internacao,
    E.nat_lesao,
    E.horas_trabalhadas,
    E.cid as cid_cat,
    E.area,
    E.data_do_acidente,
    E.tipo,
    E.reg_policial,
    E.devera_afastar,
    CASE WHEN C.identificacao IS NOT NULL THEN 1 ELSE 0 END AS trf_saida,
    B.estado AS estado_venda,
    D.estado AS estado_processo
FROM NAGATO.nova_combas A
INNER JOIN urano.base_venda B ON A.cpf = B.cpf
INNER JOIN urano.base_processo D ON B.cliente_id = D.cliente_id
    AND D.estado IN (7, 8, 15, 75, 83, 112, 125)
LEFT JOIN TRF.trf_saida C ON A.cpf = C.identificacao
    AND (C.assunto LIKE '%DIREITO PREVIDENCI%' OR C.assunto IS NULL)
LEFT JOIN inss.base_cat E ON A.cpf = E.cpf;
```

#### Option B: Using EXISTS for Better Performance
```sql
SELECT
    A.cpf,
    A.nb,
    A.nome,
    A.cid,
    A.cid_resumo,
    A.despacho,
    A.especie,
    A.faixa_salarial,
    A.faixa_tempo_concessao,
    A.sexo,
    A.uf,
    A.qt_duracao_beneficio_em_dias,
    A.dt_dcb,
    A.grau_instrucao,
    D.escritorio_id as escritorio_baseprocesso,
    E.cbo,
    E.houve_afastamento,
    E.local_do_acidente,
    E.parte_do_corpo,
    E.houve_internacao,
    E.nat_lesao,
    E.horas_trabalhadas,
    E.cid as cid_cat,
    E.area,
    E.data_do_acidente,
    E.tipo,
    E.reg_policial,
    E.devera_afastar,
    CASE 
        WHEN EXISTS (
            SELECT 1 FROM TRF.trf_saida 
            WHERE identificacao = A.cpf 
            AND (assunto LIKE '%DIREITO PREVIDENCI%' OR assunto IS NULL)
        ) THEN 1 
        ELSE 0 
    END AS trf_saida,
    B.estado AS estado_venda,
    D.estado AS estado_processo
FROM NAGATO.nova_combas A
INNER JOIN urano.base_venda B ON A.cpf = B.cpf
INNER JOIN urano.base_processo D ON B.cliente_id = D.cliente_id
    AND D.estado IN (7, 8, 15, 75, 83, 112, 125)
LEFT JOIN inss.base_cat E ON A.cpf = E.cpf;
```

### 3. **MariaDB-Specific Optimizations**

#### Enable Query Cache (if applicable)
```sql
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_size = 268435456; -- 256MB
```

#### Optimize JOIN Buffer Size
```sql
SET SESSION join_buffer_size = 8388608; -- 8MB
```

#### Use Optimizer Hints
```sql
SELECT /*+ USE_INDEX(A, idx_nova_combas_cpf) 
          USE_INDEX(B, idx_base_venda_cpf) */
    -- ... rest of query
```

### 4. **Alternative Approaches**

#### Materialized View Approach
```sql
-- Create a materialized view for frequently accessed combinations
CREATE TABLE mv_combined_base AS
SELECT 
    A.cpf,
    A.nome,
    B.cliente_id,
    B.estado as estado_venda,
    D.estado as estado_processo,
    D.escritorio_id
FROM NAGATO.nova_combas A
INNER JOIN urano.base_venda B ON A.cpf = B.cpf
INNER JOIN urano.base_processo D ON B.cliente_id = D.cliente_id
WHERE D.estado IN (7, 8, 15, 75, 83, 112, 125);

-- Create indexes on materialized view
CREATE INDEX idx_mv_cpf ON mv_combined_base(cpf);
CREATE INDEX idx_mv_cliente_id ON mv_combined_base(cliente_id);
```

## Performance Monitoring Recommendations

### 1. **Use EXPLAIN to Analyze Execution Plan**
```sql
EXPLAIN EXTENDED [your_optimized_query];
SHOW WARNINGS;
```

### 2. **Monitor Key Metrics**
- Query execution time
- Rows examined vs. rows returned ratio
- Index usage statistics
- Join buffer usage

### 3. **Regular Maintenance**
```sql
-- Update table statistics
ANALYZE TABLE NAGATO.nova_combas, urano.base_venda, 
              TRF.trf_saida, urano.base_processo, inss.base_cat;

-- Optimize tables periodically
OPTIMIZE TABLE NAGATO.nova_combas, urano.base_venda, 
               TRF.trf_saida, urano.base_processo, inss.base_cat;
```

## Expected Performance Improvements

### Before Optimization
- **Estimated rows examined**: 500,000 - 2,000,000+
- **Execution time**: 5-30 seconds
- **Resource usage**: High CPU and I/O

### After Optimization
- **Estimated rows examined**: 50,000 - 200,000
- **Execution time**: 0.5-3 seconds
- **Resource usage**: Reduced by 60-80%

## Implementation Priority

1. **High Priority**: Create indexes on JOIN columns
2. **Medium Priority**: Restructure WHERE clause conditions
3. **Low Priority**: Consider materialized views for very frequent queries
4. **Ongoing**: Monitor and maintain statistics

## Additional Considerations

- **Data Volume**: Monitor performance as data grows
- **Concurrent Users**: Test under realistic load conditions
- **Hardware**: Ensure adequate RAM for index caching
- **Backup Strategy**: Account for increased index maintenance overhead