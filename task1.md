# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.022..0.022 rows=0 loops=1)"
"  Recheck Cond: (category IS NULL)"
"  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.018..0.018 rows=0 loops=1)"
"        Index Cond: (category IS NULL)"
"Planning Time: 26.474 ms"
"Execution Time: 0.074 ms"
   *Объясните результат:*
   [План показывает последовательное сканирование таблицы, так как BRIN индекс подходит для диапазонов данных, но не помогает найти NULL значения.
Индекс не ускоряет поиск NULL, поскольку NULL не индексируется.]

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ["Bitmap Heap Scan on t_books  (cost=17.94..252.86 rows=1 width=33) (actual time=2.194..2.196 rows=0 loops=1)"
"  Recheck Cond: (((author)::text = 'SYSTEM'::text) AND ((category)::text = 'INDEX'::text))"
"  ->  BitmapAnd  (cost=17.94..17.94 rows=72 width=0) (actual time=2.184..2.185 rows=0 loops=1)"
"        ->  Bitmap Index Scan on idx_author_title  (cost=0.00..5.54 rows=149 width=0) (actual time=2.182..2.183 rows=0 loops=1)"
"              Index Cond: ((author)::text = 'SYSTEM'::text)"
"        ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.15 rows=72135 width=0) (never executed)"
"              Index Cond: ((category)::text = 'INDEX'::text)"
"Planning Time: 0.462 ms"
"Execution Time: 2.250 ms"]
   
   *Объясните результат (обратите внимание на bitmap scan):*
   [Используется Bitmap Heap Scan, где индексы по обоим полям сокращают объем проверяемых данных.
BRIN индекс эффективен при низкой селективности, что видно из ускорения запроса.]

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ["Sort  (cost=3155.14..3155.15 rows=6 width=7) (actual time=100.925..100.928 rows=6 loops=1)"
"  Sort Key: category"
"  Sort Method: quicksort  Memory: 25kB"
"  ->  HashAggregate  (cost=3155.00..3155.06 rows=6 width=7) (actual time=100.861..100.865 rows=6 loops=1)"
"        Group Key: category"
"        Batches: 1  Memory Usage: 24kB"
"        ->  Seq Scan on t_books  (cost=0.00..2780.00 rows=150000 width=7) (actual time=0.011..25.661 rows=150000 loops=1)"
"Planning Time: 0.324 ms"
"Execution Time: 101.018 ms"]
   
   *Объясните результат:*
   [Последовательное сканирование таблицы. Индексы не используются, так как запрос требует обработки всех строк.]

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ["Aggregate  (cost=3155.04..3155.05 rows=1 width=8) (actual time=44.238..44.241 rows=1 loops=1)"
"  ->  Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=0) (actual time=44.148..44.149 rows=0 loops=1)"
"        Filter: ((author)::text ~~ 'S%'::text)"
"        Rows Removed by Filter: 150000"
"Planning Time: 0.248 ms"
"Execution Time: 44.302 ms"]
   
   *Объясните результат:*
   [Запрос выполняется последовательным сканированием. Индекс BRIN подходит для равенства, но не ускоряет паттерн LIKE.]

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ["Aggregate  (cost=3531.88..3531.89 rows=1 width=8) (actual time=102.353..102.355 rows=1 loops=1)"
"  ->  Seq Scan on t_books  (cost=0.00..3530.00 rows=750 width=0) (actual time=102.291..102.345 rows=1 loops=1)"
"        Filter: (lower((title)::text) ~~ 'o%'::text)"
"        Rows Removed by Filter: 149999"
"Planning Time: 0.430 ms"
"Execution Time: 102.695 ms"]
   
   *Объясните результат:*
   [Новый индекс значительно ускоряет запрос, так как хранит предвычисленные значения.]

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ["Bitmap Heap Scan on t_books  (cost=5.54..431.38 rows=1 width=33) (actual time=0.021..0.021 rows=0 loops=1)"
"  Recheck Cond: ((author)::text = 'SYSTEM'::text)"
"  Filter: ((category)::text = 'INDEX'::text)"
"  ->  Bitmap Index Scan on idx_author_title  (cost=0.00..5.54 rows=149 width=0) (actual time=0.018..0.018 rows=0 loops=1)"
"        Index Cond: ((author)::text = 'SYSTEM'::text)"
"Planning Time: 0.507 ms"
"Execution Time: 0.059 ms"]
   
   *Объясните результат:*
   [Составной индекс значительно ускоряет запрос, так как объединяет фильтры по category и author.]