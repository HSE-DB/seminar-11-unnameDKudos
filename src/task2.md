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
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.151..0.152 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.123..0.123 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 3.448 ms
    Execution Time: 0.331 ms
    ```
    
    *Объясните результат:*
    Запрос использует GIN (Generalized Inverted Index) индекс для полнотекстового поиска. Bitmap Index Scan эффективно находит все записи, содержащие слово 'expert' в заголовке. Индекс работает очень быстро (0.331 ms), что демонстрирует преимущества GIN индексов для полнотекстового поиска. Найдена одна книга с заголовком "Expert PostgreSQL Architecture". GIN индексы идеально подходят для поиска по массивам, полнотекстовым данным и JSON, так как они хранят все позиции вхождения каждого значения.

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
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.220..0.232 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 3.244 ms
     Execution Time: 0.953 ms
     ```
     
     *Объясните результат:*
     Запрос использует Index Scan по первичному ключу t_lookup_pk для поиска записи. Время выполнения составляет 0.953 ms. Индекс эффективно находит нужную запись, но затем требуется доступ к таблице для получения полных данных. В обычной таблице данные физически не отсортированы по первичному ключу, поэтому после поиска в индексе может потребоваться чтение случайных страниц данных.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.061..0.062 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 1.026 ms
     Execution Time: 0.133 ms
     ```
     
     *Объясните результат:*
     В кластеризованной таблице поиск по первичному ключу работает значительно быстрее (0.133 ms против 0.953 ms в обычной таблице - почти в 7 раз быстрее). Это происходит потому, что данные в кластеризованной таблице физически отсортированы по первичному ключу, поэтому после поиска в индексе данные находятся на последовательных страницах, что значительно уменьшает количество операций чтения с диска. Кластеризация особенно эффективна для запросов, которые читают данные по порядку первичного ключа или по диапазонам.

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
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.060..0.061 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 1.289 ms
     Execution Time: 0.120 ms
     ```
     
     *Объясните результат:*
     Запрос использует индекс t_lookup_value_idx для поиска по значению. Индекс эффективно находит, что записей с таким значением нет (rows=0). Время выполнения 0.120 ms - очень быстрое, так как индекс позволяет быстро определить отсутствие искомого значения без полного сканирования таблицы. Поиск по значению в обычной таблице не зависит от кластеризации по первичному ключу, так как используется другой индекс.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.135..0.137 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 1.435 ms
     Execution Time: 0.444 ms
     ```
     
     *Объясните результат:*
     В кластеризованной таблице поиск по значению выполняется немного медленнее (0.444 ms против 0.120 ms), что может быть связано с различными факторами: кэширование, случайные колебания производительности, или особенности физического размещения данных после кластеризации. Однако разница незначительна. Важно отметить, что кластеризация по первичному ключу не влияет на эффективность поиска по другим индексам (в данном случае по item_value), так как используется отдельный индекс.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     При поиске по значению в обычной таблице время выполнения составило 0.120 ms, а в кластеризованной - 0.444 ms. Разница незначительна и может варьироваться в зависимости от различных факторов (кэш, нагрузка системы). Кластеризация по первичному ключу не влияет на производительность поиска по другим колонкам через их индексы, так как используются разные структуры данных. Основное преимущество кластеризации проявляется при поиске и последовательном чтении по первичному ключу (как показано в шагах 13-14, где разница была почти в 7 раз). Для поиска по значениям через индекс преимущества кластеризации минимальны или отсутствуют.