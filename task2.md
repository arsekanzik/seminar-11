## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on t_books  (cost=21.03..1367.60 rows=750 width=33) (actual time=0.028..0.029 rows=1 loops=1)"
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"  Heap Blocks: exact=1"
"  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.021..0.021 rows=1 loops=1)"
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
"Planning Time: 2.644 ms"
"Execution Time: 0.093 ms"
    
    *Объясните результат:*
    GIN индекс используется для быстрого поиска, что видно по значительному уменьшению времени выполнения.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     "Index Scan using t_lookup_pk on t_lookup  (cost=0.41..8.43 rows=1 width=256) (actual time=0.059..0.061 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.172 ms"
"Execution Time: 0.112 ms"
     
     *Объясните результат:*
     Индекс позволяет выполнить быстрый поиск

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     "Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.096..0.098 rows=1 loops=1)"
"  Index Cond: ((item_key)::text = '0000000455'::text)"
"Planning Time: 0.242 ms"
"Execution Time: 0.121 ms"
     
     *Объясните результат:*
     Изменения незначительны т. к. кластеризация в данном случае не помогла

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
    "Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.041..0.042 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 0.449 ms"
"Execution Time: 0.068 ms"
     
     *Объясните результат:*
     [Ваше объяснение]

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     "Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.192..0.193 rows=0 loops=1)"
"  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
"Planning Time: 0.880 ms"
"Execution Time: 0.220 ms"
     
     *Объясните результат:*
     Кластеризация не помогла

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     [В целом, поиск быстрее в кластеризованной таблице из-за упорядоченности данных, однако из-за специфики таблицы и эффективности индексов, разница не заметна]