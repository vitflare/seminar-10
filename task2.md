# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   ```
   workshop.public> ANALYZE t_books
   [2024-11-27 00:08:51] completed in 134 ms
   workshop.public> ANALYZE t_books_part
   [2024-11-27 00:08:51] completed in 268 ms
   ```

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
   Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1033.99 rows=1 width=32) (actual time=0.018..7.137 rows=1 loops=1)
   Filter: (book_id = 18)
   Rows Removed by Filter: 49998
   Planning Time: 1.242 ms
   Execution Time: 7.167 ms
   ```
   
   *Объясните результат:*

   Использовалось последовательное сканирование первой партиции `t_books_part_1`, так как book_id = 18 попадает в диапазон первой партиции (от MINVALUE до 50000). `cost` - оценочная стоимость операции, `row` - ожидаемое количество найденных строк, `width` - примерный размер строки (в байтах)

   `Rows Removed by Filter: 49998` - из 50000 строк партиции удалено 49998 строк, не соответствующих условию, `actual time=0.018..7.137` - реальное время выполнения около 7.1 мс

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```
    Append  (cost=0.00..3102.01 rows=3 width=33) (actual time=4.337..14.400 rows=1 loops=1)
    ->  Seq Scan on t_books_part_1  (cost=0.00..1033.99 rows=1 width=32) (actual time=4.336..4.337 rows=1 loops=1)
        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
        Rows Removed by Filter: 49998
    ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=4.929..4.929 rows=0 loops=1)
        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
        Rows Removed by Filter: 50000
    ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=5.129..5.129 rows=0 loops=1)
        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
        Rows Removed by Filter: 50001
    Planning Time: 0.751 ms
    Execution Time: 14.505 ms
   ```
   
   *Объясните результат:*
   
   `Append` означает, что PostgreSQL выполняет поиск по всем партициям таблицы

   Книга найдена в первой партиции, однако можно видеть, что оптимизатор прошелся по всем партициям. Следовательно, производительность низкая из-за полного сканирования всех партиций

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*

   Создан индекс

   ![pic](6)

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.152..0.284 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx1 on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.151..0.152 rows=1 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
     ->  Index Scan using t_books_part_2_title_idx1 on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.045..0.045 rows=0 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx1 on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.084..0.085 rows=0 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 2.201 ms
    Execution Time: 0.395 ms
   ```
   
   *Объясните результат:*
   
   В сравнение с предыдущим пунктом, где вызывался этот же запрос, можно видеть, что теперь оптимизатор использовал `Index Scan` по `t_books_part_2_title_idx1`, вместо просмотра всех строк, что в разы ускорило получение результата

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   
   Индекс удален

   ![pic](7)

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   
   Созданы индексы непосредственно внутри партиций

   ![pic](8)

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.308..0.589 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.307..0.309 rows=1 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.139..0.139 rows=0 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.137..0.137 rows=0 loops=1)
        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 2.489 ms
    Execution Time: 0.848 ms
    ```
    
    *Объясните результат:*

    Если сравнивать результат с пунктом, где не использовалс индекс, то результат улучшился: оптимизатор использовал `Index Scan` по индексам партиций, вместо просмотра всех строк, что в разы ускорило получение результата. Но в сравнении с пунктом, где использовался один индекс, время выполнения немного увеличилось (с 0.395 мс до 0.848 мс). Это связано с тем, что теперь база смотрит индексы во всех партициях, однако это по прежнему можно считать эффективным индексным поиском

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*

    Индексы удалены

    ![pic](9)

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    
    Индекс создан

    ![pic](10)

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```
    Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.129..0.130 rows=1 loops=1)
    Index Cond: (book_id = 11011)
    Planning Time: 1.101 ms
    Execution Time: 0.197 ms
    ```
    
    *Объясните результат:*

    Создание индекса по `book_id` в партиционированной таблице значительно ускоряет поиск, позволяя находить записи практически мгновенно. Можно видеть, что мы просмотрели только первую партицию и дальше не пошли. Так как партиции созданы с помощью деления `id`, то база точно знает, что в остальных двух партициях такого значения не будет. Благодаря этому запрос настолько быстро выполнился

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    
    Индекс создан

    ![pic](11)

15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books  (cost=843.44..2820.89 rows=75245 width=33) (actual time=3.531..22.165 rows=75067 loops=1)
    Recheck Cond: is_active
    Heap Blocks: exact=1225
    ->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..824.63 rows=75245 width=0) (actual time=3.359..3.359 rows=75067 loops=1)
        Index Cond: (is_active = true)
    Planning Time: 0.549 ms
    Execution Time: 24.866 ms
    ```
    
    *Объясните результат:*

    - `Bitmap Index Scan on t_books_active_idx`:
    
        Первым делом база использует созданный индекс `t_books_active_idx` и находит все записи, где `is_active` = true
    
    - `Bitmap Heap Scan on t_books`:
    
        После индексного сканирования выполняется проход по куче (heap)

        `Recheck Cond: is_active` - повторная проверка условия
    
        `Heap Blocks: exact=1225` - количество блоков памяти, затронутых при сканировании
    
        Фактическое время сканирования: 22.165 мс

    - Общая статистика:
    
        Время планирования: 0.549 мс

        Общее время выполнения: 24.866 мс
    
        Найдено 75,067 активных книг из 150,000 

        Метод `Bitmap Scan` - это компромисс между полным последовательным сканированием и чистым индексным сканированием,

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*

    Индекс создан

    ![pic](12)

17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```
    HashAggregate  (cost=3475.00..3485.00 rows=1000 width=42) (actual time=80.375..80.469 rows=1003 loops=1)
    Group Key: author
    Batches: 1  Memory Usage: 193kB
    ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=21) (actual time=0.015..26.743 rows=150000 loops=1)
    Planning Time: 1.184 ms
    Execution Time: 80.676 ms
    ```
    
    *Объясните результат:*

    Выполняется последовательное сканирование таблицы и используется хеш-вгрегация для группировки, где ключ = `author`

    Общая производительность:
    - Общее время выполнения: 80.676 мс
    - Требуется полное сканирование таблицы
    - Высокие накладные расходы на группировку

    В данном случае созданный ранее индекс не использовался из-за характера операции агрегации (MAX) и специфики работы оптимизатора

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```
    Limit  (cost=0.42..56.67 rows=10 width=10) (actual time=0.511..0.937 rows=10 loops=1)
    ->  Result  (cost=0.42..5625.42 rows=1000 width=10) (actual time=0.510..0.933 rows=10 loops=1)
        ->  Unique  (cost=0.42..5625.42 rows=1000 width=10) (actual time=0.508..0.929 rows=10 loops=1)
              ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5250.42 rows=150000 width=10) (actual time=0.506..0.831 rows=1344 loops=1)
                    Heap Fetches: 2
    Planning Time: 0.396 ms
    Execution Time: 0.996 ms
    ```
    
    *Объясните результат:*
    
    - `Index Only Scan`: Использует ранее созданный составной индекс `t_books_author_title_index`
    - `Heap Fetches: 2`: Минимальное количество обращений к таблице
    - Возвращено 10 уникальных авторов из 1344 просканированных строк
    - Минимальное время выполнения и планирования

    В отличие от предыдущего запроса, здесь созданный индекс использовался, так как индекс начинается с author, что идеально подходит для операций, связанных с сортировкой или фильтрацией по автору. Помимо этого, посмотрим на операции в запросе, которые используют созданный ранее индекс:
    - `DISTINCT author`: Требует уникальных значений автора
    - `ORDER BY author`: Сортировка по первому полю индекса

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```
    Sort  (cost=3100.29..3100.33 rows=15 width=21) (actual time=48.859..48.859 rows=1 loops=1)
        Sort Key: author, title 
        Sort Method: quicksort  Memory: 25kB
        ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=21) (actual time=48.808..48.808 rows=1 loops=1)
            Filter: ((author)::text ~~ 'T%'::text)
            Rows Removed by Filter: 149999
    Planning Time: 0.847 ms
    Execution Time: 48.922 ms
    ```
    
    *Объясните результат:*
    
    Запрос выполнялся медленно по нескольким причинам:
    - Полное сканирование таблицы (Seq Scan):
    
        - Просмотрено 150 000 строк
    
        - Для каждой строки проверяется условие LIKE 'T%'
    
        - Это очень ресурсоемкая операция


    - Операция сортировки:

        - Требуется отсортировать найденные строки по author и title
        - Сортировка выполняется в памяти (quicksort)
        - Занимает дополнительное время

    - Отсутствие подходящего индекса:
        - Нет индекса, который бы помогал в фильтрации по author
        - Нет индекса для быстрой сортировки

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ```
    workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                     VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true)
    [2024-11-27 01:18:25] 1 row affected in 6 ms
    workshop.public> COMMIT
    [2024-11-27 01:18:25] [25P01] there is no transaction in progress
    [2024-11-27 01:18:25] completed in 12 ms
    ```

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    
    Индекс создан

    ![pic](13)

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.15 rows=1 width=21) (actual time=0.255..0.258 rows=1 loops=1)
        Index Cond: (category IS NULL)
    Planning Time: 2.201 ms
    Execution Time: 0.496 ms
    ```
    
    *Объясните результат:*
    
    - Полный индекс по категории
    - Время выполнения: 0.496 мс
    - Стоимость операции: 0.29..8.15

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    Индекс создан

    ![pic](13)

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.98 rows=1 width=21) (actual time=0.038..0.039 rows=1 loops=1)
    Planning Time: 0.860 ms
    Execution Time: 0.072 ms
    ```
    
    *Объясните результат:*
    - Содержит только NULL значения в категории
    - Время выполнения: 0.072 мс (в 7 раз быстрее, чем при индексе по категории)
    - Стоимость операции: 0.12..7.98 (ниже, чем при индексе по категории)

    Частичный индекс стал выгоднее, так как:
    - Меньший размер индекса
    - Более быстрый поиск NULL значений
    - Экономия дискового пространства
    - Более эффективная работа с редкими значениями 

    Другими словами, частичные индексы выгодны для нечастых или специфических условий поиска, как в нашем случае

25. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    ```sql
    -- Создание индекса
    workshop.public> CREATE UNIQUE INDEX t_books_selective_unique_idx
                     ON t_books(title)
                     WHERE category = 'Science'
    [2024-11-27 01:27:53] completed in 85 ms

    -- Тест индекса
    workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                     VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true)
    [2024-11-27 01:28:24] 1 row affected in 3 ms

    -- Попытка вставить дубликат
    [2024-11-27 01:28:45] [23505] ERROR: duplicate key value violates unique constraint "t_books_selective_unique_idx"
    [2024-11-27 01:28:45] Подробности: Key (title)=(Unique Science Book) already exists.

    -- Вставка такого же названия для другой категории
    workshop.public> INSERT INTO t_books (book_id, title, author, category, is_active)
                     VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true)
    [2024-11-27 01:29:53] 1 row affected in 5 ms
    ```
    
    *Объясните результат:*
    - Уникальность действует только для книг в категории "Science"
    - Можно иметь книги с одинаковым названием в разных категориях
    - Индекс защищает от дублей внутри одной категории
    
    Этот пример демонстрирует возможности частичных уникальных индексов в бд, позволяющих создавать гибкие ограничения уникальности.