## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=28.608..292.616 rows=499323 loops=1)"
"  Recheck Cond: (category = 'A'::text)"
"  Heap Blocks: exact=8334"
"  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=26.661..26.662 rows=499323 loops=1)"
"        Index Cond: (category = 'A'::text)"
"Planning Time: 0.420 ms"
"Execution Time: 329.362 ms"
    
    *Объясните результат:*
    Большое количество данных замедлило процесс

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
Query returned successfully in 2 secs 457 msec.

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on test_cluster  (cost=5550.12..20106.21 rows=497767 width=39) (actual time=34.956..217.576 rows=499323 loops=1)"
"  Recheck Cond: (category = 'A'::text)"
"  Heap Blocks: exact=4162"
"  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5425.68 rows=497767 width=0) (actual time=33.894..33.894 rows=499323 loops=1)"
"        Index Cond: (category = 'A'::text)"
"Planning Time: 0.294 ms"
"Execution Time: 257.882 ms"
    
    *Объясните результат:*
   Кластеризация улучшает производительность запросов за счет упорядочения данных в таблице.


6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    Кластеризация улучшает производительность запросов за счет упорядочения данных в таблице.
