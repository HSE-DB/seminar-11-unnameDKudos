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
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.025..0.025 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.018..0.018 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.923 ms
   Execution Time: 0.124 ms
   ```
   
   *Объясните результат:*
   Запрос использует BRIN индекс для поиска NULL значений. Выполняется Bitmap Index Scan по индексу t_books_brin_cat_idx с условием category IS NULL, затем Bitmap Heap Scan для получения полных строк. В таблице нет записей с NULL категорией (rows=0), поэтому выполнение очень быстрое (0.124 ms). BRIN индекс эффективно отфильтровал все блоки, которые не могут содержать NULL значения.

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
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=15.494..15.495 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.163..0.164 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 1.399 ms
   Execution Time: 15.641 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Запрос использует bitmap scan с BRIN индексом по категории. Bitmap Index Scan находит потенциально подходящие блоки (12250 строк в 1225 lossy блоках), но из-за неточности BRIN индекса требуется перепроверка всех 150000 строк (Rows Removed by Index Recheck: 150000). Дополнительно применяется фильтр по автору на уровне кучи. Результат: записей не найдено. BRIN индекс эффективен для больших таблиц с естественной кластеризацией данных, но может быть неточным.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=31.599..31.601 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=31.510..31.513 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.007..9.191 rows=150000 loops=1)
   Planning Time: 1.215 ms
   Execution Time: 31.925 ms
   ```
   
   *Объясните результат:*
   Для получения уникальных категорий используется последовательное сканирование всей таблицы (Seq Scan), так как BRIN индекс не подходит для операций DISTINCT. Затем выполняется HashAggregate для группировки по категориям и Sort для упорядочивания. Найдено 6 уникальных категорий. BRIN индексы не используются для этой операции, так как они не поддерживают эффективный поиск уникальных значений.
   
9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=8.836..8.837 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=8.832..8.832 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 1.130 ms
   Execution Time: 8.945 ms
   ```
   
   *Объясните результат:*
   Запрос с префиксным поиском (LIKE 'S%') не может использовать BRIN индекс по автору, так как BRIN индексы не поддерживают эффективный поиск по шаблонам. Выполняется последовательное сканирование всей таблицы с фильтрацией. Все 150000 строк проверяются, но ни одна не соответствует условию (author начинается с 'S' в верхнем регистре), так как в данных авторы имеют формат 'Author_N'. BRIN индексы эффективны только для точных совпадений и диапазонов, но не для паттернов LIKE.

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
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=32.952..32.953 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=32.941..32.944 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 1.629 ms
   Execution Time: 33.058 ms
   ```
   
   *Объясните результат:*
   Несмотря на наличие индекса t_books_lower_title_idx на LOWER(title), PostgreSQL не использует его для запроса с оператором LIKE. Это происходит потому, что индексы B-tree могут использоваться для префиксных поисков только с операторами типа >= и <, но не с LIKE. Для эффективного поиска по шаблонам в PostgreSQL используются специализированные индексы, такие как pg_trgm (trigram) или полнотекстовые индексы. В данном случае выполняется последовательное сканирование всех 150000 строк с применением функции LOWER к каждой строке, что значительно медленнее.

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
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=1.123..1.123 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8841
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.153..0.153 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 2.304 ms
   Execution Time: 1.500 ms
   ```
   
   *Объясните результат:*
   Составной BRIN индекс (category, author) работает значительно лучше, чем отдельные индексы. Bitmap Index Scan теперь использует оба условия одновременно в индексе, что уменьшает количество строк для перепроверки с 150000 до 8841 (почти в 17 раз). Количество lossy блоков сократилось с 1225 до 73. Время выполнения уменьшилось с 15.641 ms до 1.500 ms (более чем в 10 раз). Это демонстрирует преимущество составных BRIN индексов для запросов с несколькими условиями на колонках, которые включены в индекс.