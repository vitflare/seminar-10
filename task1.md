# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```
   Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=37.633..37.635 rows=1 loops=1)
   Filter: ((title)::text = 'Oracle Core'::text)
   Rows Removed by Filter: 149999
   Planning Time: 0.611 ms
   Execution Time: 37.662 ms
   ```
   *Объясните результат:*
   
   Использовалось последовательное сканирование, при котором бд пришлось просмотреть все 150000 записей в таблице. `cost` - оценочная стоимость операции, `row` - ожидаемое количество найденных строк, `width` - примерный размер строки (в байтах)

   `Rows Removed by Filter: 149999` - из 150 000 строк было отфильтровано 149 999 и найдена всего одна, `actual time=37.633..37.635` - реальное время выполнения около 37.6 мс

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   Созданы новые индексы

   ![pic](https://github.com/vitflare/seminar-10/blob/seminar-10/shiverskikh-elizaveta/screenshots/1.png)

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*

   ![pic](https://github.com/vitflare/seminar-10/blob/seminar-10/shiverskikh-elizaveta/screenshots/2.png)
   
   *Объясните результат:*

   Индекс `t_books_id_pk` является уникальным индексом на столбце `book_id`. Это соответствует первичному ключу таблицы
   
   Индекс `t_books_title_idx` создан на столбце `title`. Это поможет ускорить поиск книг по названию
   
   Индекс `t_books_active_idx` создан на столбце `is_active`. Это позволит быстрее фильтровать книги по признаку активности

   С помощью созданных индексов мы должны повысить производительность выполнения запросов за счет того, что эти три индекса покрывают основные сценарии поиска и фильтрации данных

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ```
   workshop.public> ANALYZE t_books
   [2024-11-26 16:24:49] completed in 127 ms
   ```

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```
   Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.302..0.304 rows=1 loops=1)
   Index Cond: ((title)::text = 'Oracle Core'::text)
   Planning Time: 2.233 ms
   Execution Time: 0.411 ms
   ```
   
   *Объясните результат:*

   В этом плане база данных использует индекс `t_books_title_idx`, который мы создали на столбце `title`. Вместо последовательного сканирования всей таблицы, теперь база данных может быстро найти нужную строку, используя индекс

   Вместо `Seq Scan` мы видим `Index Scan`, что означает использование индекса
   
   `cost=0.42..8.44` значительно ниже, чем в предыдущем случае `cost=0.00..3100.00`
   
   `actual time=0.302..0.304 ms` гораздо быстрее, чем `actual time=37.633..37.635 ms`

   Таким образом видно, что использование индекса помогло ускорить поиск

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
   Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=7.032..7.035 rows=1 loops=1)
   Index Cond: (book_id = 18)
   Planning Time: 0.924 ms
   Execution Time: 7.139 ms
   ```
   
   *Объясните результат:*
   
   `Index Scan using t_books_id_pk on t_books`
   
   Это значит, что база данных использует индекс t_books_id_pk для поиска данных в таблице t_books
   
   `Index Cond: (book_id = 18)`
   
   Показывает, что поиск происходит по значению book_id = 18.
Использование индекса позволяет быстро найти нужную запись, не выполняя полное сканирование всей таблицы.
    
    `cost=0.42..8.44`
    
    Это оценка стоимости выполнения операции, показывающая, что поиск по индексу гораздо дешевле, чем полное сканирование таблицы.
    
    `rows=1`
    
    Ожидается, что будет найдена только одна запись, соответствующая условию.
    
    `width=33` 
    
    Предполагаемый размер каждой найденной записи в байтах.
    
    `actual time=7.032..7.035 rows=1 loops=1` 
    
    Запрос фактически выполнялся 7 миллисекунд и вернул 1 строку.
    
    `Planning Time: 0.924 ms`
    
    Время, потраченное на планирование запроса, составило около 1 миллисекунды.
    
    `Execution Time: 7.139 ms` 
    
    Общее время выполнения запроса, включая планирование, составило около 7 миллисекунд.

    Главное, что стоит заметить, что использовался `Index Scan`

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```
   Seq Scan on t_books  (cost=0.00..2725.00 rows=74245 width=33) (actual time=0.036..30.156 rows=74489 loops=1)
   Filter: is_active
   Rows Removed by Filter: 75511
   Planning Time: 0.453 ms
   Execution Time: 33.156 ms
   ```
   
   *Объясните результат:*
   Видно, что использовался `Seq Scan`. Хоть индекс `t_books_active_idx` и был создан, база данных не использовала его в этом запросе. Это произошло, потому что поиск по индексу в данном случае может быть менее эффективным, чем просто последовательное сканирование таблицы.

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   
   ![pic](https://github.com/vitflare/seminar-10/blob/seminar-10/shiverskikh-elizaveta/screenshots/3.png)

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*

    Индексы были удалены

    ![pic](https://github.com/vitflare/seminar-10/blob/seminar-10/shiverskikh-elizaveta/screenshots/4.png)

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:

    a. `WHERE title = $1 AND category = $2`
    ```
    CREATE INDEX IF NOT EXISTS idx_books_title_category 
    ON t_books (title, category);
    ```
    b. `WHERE title = $1`
    ```
    CREATE INDEX IF NOT EXISTS idx_books_title
    ON t_books (title);
    ```
    c. `WHERE category = $1 AND author = $2`
    ```
    CREATE INDEX IF NOT EXISTS idx_books_category_author
    ON t_books (category, author);
    ```
    d. `WHERE author = $1 AND book_id = $2`
    ```
    CREATE INDEX IF NOT EXISTS idx_books_author_id
    ON t_books (author, book_id);
    ```
    
    *Объясните ваше решение:*

    Создание индексов на часто используемых столбцах фильтрации (`title, category, author`) значительно ускорит выполнение соответствующих запросов

    Составные индексы (`idx_books_title_category, idx_books_category_author, idx_books_author_id`) оптимизируют поиск по комбинациям полей, что важно для сложных условий WHERE

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    a. `WHERE title = $1 AND category = $2`

    - с индексом:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE title = 'Book_100' AND category = 'Technology';

    Index Scan using idx_books_title on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.057..0.058 rows=1 loops=1)
    Index Cond: ((title)::text = 'Book_100'::text)
    Filter: ((category)::text = 'Technology'::text)
    Planning Time: 0.146 ms
    Execution Time: 0.077 ms
    ```
    - без индекса:
    ```
    DROP INDEX idx_books_title_category;
    
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE title = 'Book_100' AND category = 'Technology';

    Index Scan using idx_books_title on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.095..0.096 rows=1 loops=1)
    Index Cond: ((title)::text = 'Book_100'::text)
    Filter: ((category)::text = 'Technology'::text)
    Planning Time: 0.618 ms
    Execution Time: 0.170 ms
    ```
    В данном случае происходит поиск по идндексу, так как у нас присутствует еще один индекс, который навешан на столбец `title`. Однако даже в таком случае этот запрос медленее, чем с индексом `idx_books_title`

    b. `WHERE title = $1`
    - с индексом:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE title = 'Book_100';

    Index Scan using idx_books_title on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.062..0.063 rows=1 loops=1)
    Index Cond: ((title)::text = 'Book_100'::text)
    Planning Time: 0.249 ms
    Execution Time: 0.084 ms
    ```
    - без индекса:
    ```
    DROP INDEX idx_books_title;
    
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE title = 'Book_100';

    Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=0.027..12.276 rows=1 loops=1)
    Filter: ((title)::text = 'Book_100'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.074 ms
    Execution Time: 12.294 ms
    ```
    c. `WHERE category = $1 AND author = $2`
    - с индексом:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE category = 'Fiction' AND author = 'Author_100';

    Bitmap Heap Scan on t_books  (cost=4.60..110.97 rows=30 width=33) (actual time=0.432..0.543 rows=38 loops=1)
    Recheck Cond: (((category)::text = 'Fiction'::text) AND ((author)::text = 'Author_100'::text))
    Heap Blocks: exact=37
    ->  Bitmap Index Scan on idx_books_category_author  (cost=0.00..4.59 rows=30 width=0) (actual time=0.401..0.401 rows=38 loops=1)
        Index Cond: (((category)::text = 'Fiction'::text) AND ((author)::text = 'Author_100'::text))
    Planning Time: 0.348 ms
    Execution Time: 0.648 ms
    ```
    В данном запросе оптимизатор использовал `Bitmap Scan` так как селективность условий низкая. Можно видеть, что нам вернуло 38 записей. Хоть это и небольшое число относительно всего количества строк в бд, этого достаточно, чтобы была использована `Bitmap Scan` вместо простого `Index Scan`. Во внутреннем запросе `Bitmap` использует рассматриваемый индекс для более быстрого определения страниц, где находятся нужные строки
    - без индекса:
    ```
    DROP INDEX idx_books_category_author;

    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE category = 'Fiction' AND author = 'Author_100';

    Bitmap Heap Scan on t_books  (cost=5.54..428.27 rows=30 width=33) (actual time=0.229..1.767 rows=38 loops=1)
    Recheck Cond: ((author)::text = 'Author_100'::text)
    Filter: ((category)::text = 'Fiction'::text)
    Rows Removed by Filter: 114
    Heap Blocks: exact=145
    ->  Bitmap Index Scan on idx_books_author_id  (cost=0.00..5.54 rows=149 width=0) (actual time=0.116..0.117 rows=152 loops=1)
        Index Cond: ((author)::text = 'Author_100'::text)
    Planning Time: 1.379 ms
    Execution Time: 1.872 ms
    ```
    Можно заметить, что здесь используется индекс `idx_books_author_id`, однако так как он был создан не под этот запрос, время исполнения все также высоко

    d. `WHERE author = $1 AND book_id = $2`
    - с индексом:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE author = 'Author_100' AND book_id = 9931;

    Index Scan using idx_books_author_id on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.127..0.128 rows=1 loops=1)
    Index Cond: (((author)::text = 'Author_100'::text) AND (book_id = 9931))
    Planning Time: 0.745 ms
    Execution Time: 0.208 ms
    ```
    - без индекса:
    ```
    DROP INDEX idx_books_author_id;

    EXPLAIN ANALYZE
    SELECT * FROM t_books
    WHERE author = 'Author_100' AND book_id = 9931;

    Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.388..0.390 rows=1 loops=1)
    Index Cond: (book_id = 9931)
    Filter: ((author)::text = 'Author_100'::text)
    Planning Time: 0.298 ms
    Execution Time: 0.407 ms
    ```
    
    *Объясните результаты:*

    Как мы видим, созданные индексы существенно ускоряют выполнение всех тестовых запросов по сравнению с последовательным сканированием таблицы. Время выполнения во всех случаях использования индексов в более чем 2 раза быстрее, чем без них 

    Таким образом, предложенные индексы обеспечивают оптимальную производительность

13. Выполните регистронезависимый поиск по началу названия:

    **NB: все индексы из прошлый пунктов дропнуты, остался только t_books_id_pk PRIMARY KEY**
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=85.369..85.370 rows=0 loops=1)
    Filter: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 2.921 ms
    Execution Time: 85.474 ms
    ```
    
    *Объясните результат:*

    Так как в этот раз не используется никакой индекс, база данных вынуждена выполнить полное последовательное сканирование всех 150 000 строк таблицы. Фильтрация применяется к каждой строке, чтобы проверить, соответствует ли значение `title` шаблону `Relational%` с учетом регистра. 

    Проблема в данном случае заключается в том, что отсутствует индекс, который мог бы ускорить поиск по начальной части названия книги

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    Индекс создан

    ![pic](https://github.com/vitflare/seminar-10/blob/seminar-10/shiverskikh-elizaveta/screenshots/5.png)

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```
    Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=33) (actual time=57.743..57.744 rows=0 loops=1)
    Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 3.654 ms
    Execution Time: 57.866 ms
    ```
    
    *Объясните результат:*

    Несмотря на наличие индекса `t_books_up_title_idx` на верхнем регистре названия книг, база данных не использует его в данном запросе. Вместо этого она продолжает выполнять полное последовательное сканирование всех 150 000 строк

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=83.433..83.435 rows=1 loops=1)
    Filter: ((title)::text ~~* '%Core%'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.813 ms
    Execution Time: 83.559 ms
    ```
    
    *Объясните результат:*

    Аналогичная пункту выше ситуация

17. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ```
    [2BP01] ERROR: cannot drop index t_books_id_pk because constraint t_books_id_pk on table t_books requires it Подсказка: You can drop constraint t_books_id_pk on table t_books instead. Где: SQL statement "DROP INDEX t_books_id_pk" PL/pgSQL function inline_code_block line 9 at EXECUTE
    ```
    
    *Объясните результат:*

    При создании таблицы t_books был определен первичный ключ на столбце `book_id`. Этот первичный ключ `t_books_id_pk` автоматически создает уникальный индекс на столбце `book_id`
    
    Удаление индекса `t_books_id_pk` напрямую невозможно, так как это нарушит целостность первичного ключа таблицы

18. Создайте индекс для оптимизации суффиксного поиска:

    **NB: t_books_up_title_idx дропнут**
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*

    Вариант 1:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';

    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=79.702..79.702 rows=0 loops=1)
    Filter: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 0.172 ms
    Execution Time: 79.722 ms
    ```
    Видно, что скорость уменьшилась (по сравнению с таким запросом без использования индексов в пункте выше), но не намного. Все также оптимизатор считает `Seq Scan` выгоднее
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';

    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=77.169..77.172 rows=1 loops=1)
    Filter: ((title)::text ~~* '%Core%'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.454 ms
    Execution Time: 77.226 ms
    ```
    Скорость запроса уменьшилась (по сравнению с запросом с использованием индекса t_books_up_title_idx), однако также не намного. Все также оптимизатор считает `Seq Scan` выгоднее

    Вариант 2:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';

    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.034..0.035 rows=0 loops=1)
    Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.024..0.025 rows=1 loops=1)
        Index Cond: ((title)::text ~~* 'Relational%'::text)
    Planning Time: 0.335 ms
    Execution Time: 0.080 ms
    ```
    Использование индекса `t_books_trgm_idx`, основанного на триграммах, привело к значительному улучшению производительности. Время выполнения запроса на поиск по префиксу снизилось до 0.080 мс, что более чем в 900 раз быстрее, чем в случае с индексом на перевернутом названии
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';

    Bitmap Heap Scan on t_books  (cost=21.57..76.78 rows=15 width=33) (actual time=0.039..0.041 rows=1 loops=1)
    Recheck Cond: ((title)::text ~~* '%Core%'::text)
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..21.56 rows=15 width=0) (actual time=0.025..0.025 rows=1 loops=1)
        Index Cond: ((title)::text ~~* '%Core%'::text)
    Planning Time: 0.332 ms
    Execution Time: 0.095 ms
    ```
    Аналогичные результаты получаются и для поиска по подстроке в названии книги. Время выполнения запроса снизилось до 0.095 мс, что более чем в 800 раз быстрее, чем в случае с индексом на перевернутом названии.
    
    *Объясните результаты:*

    Таким образом, использование индекса на основе триграмм (`t_books_trgm_idx`) оказалось намного более эффективным для оптимизации суффиксного поиска, чем индекс на перевернутом названии (`t_books_rev_title_idx`). 
    
    Это можно связать с тем фактом, что индекс на триграммах лучше подходит для поиска по шаблонам и префиксам / суффиксам, в то время как индекс на перевернутом названии не обеспечивает значительного ускорения для таких случаев

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*

    Вариант 1:
    ```
    Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=18.013..18.016 rows=1 loops=1)
    Filter: ((title)::text = 'Oracle Core'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.763 ms
    Execution Time: 18.051 ms
    ```
    Вариант 2:
    ```
    Bitmap Heap Scan on t_books  (cost=116.57..120.58 rows=1 width=33) (actual time=0.144..0.145 rows=1 loops=1)
    Recheck Cond: ((title)::text = 'Oracle Core'::text)
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..116.57 rows=1 width=0) (actual time=0.112..0.112 rows=1 loops=1)
        Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.792 ms
    Execution Time: 0.282 ms
    ```
    
    *Объясните результат:*

    Второй вариант с использованием индекса `t_books_trgm_idx` демонстрирует гораздо более высокую производительность по сравнению с простым последовательным сканированием таблицы. Использование соответствующих индексов позволяет ускорить выполнение точных поисковых запросов

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*

    **NB: результаты взяты из пункта 18, где рассматривался этот запрос**

    Вариант 1:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';

    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=79.702..79.702 rows=0 loops=1)
    Filter: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 0.172 ms
    Execution Time: 79.722 ms
    ```
    Вариант 2:
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';

    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.034..0.035 rows=0 loops=1)
    Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.024..0.025 rows=1 loops=1)
        Index Cond: ((title)::text ~~* 'Relational%'::text)
    Planning Time: 0.335 ms
    Execution Time: 0.080 ms
    ```
    
    *Объясните результат:*
    
    Использование индекса `t_books_trgm_idx`, основанного на триграммах, привело к значительному улучшению производительности. Время выполнения запроса на поиск по префиксу снизилось до 0.080 мс, что более чем в 900 раз быстрее, чем в случае с индексом на перевернутом названии

    Индекс на основе триграмм эффективен для регистронезависимого поиска по префиксу, так как он использует специальные алгоритмы сравнения строк, основанные на частичном совпадении триграмм. Это делает такой индекс гораздо более подходящим для данной задачи, чем простой индекс на верхнем регистре названия

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE title LIKE 'Book_%'
    ORDER BY title DESC
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```
    Limit  (cost=0.42..1.05 rows=10 width=33) (actual time=0.152..0.155 rows=10 loops=1)
    ->  Index Scan using t_books_desc_idx on t_books  (cost=0.42..9474.32 rows=149985 width=33) (actual time=0.151..0.153 rows=10 loops=1)
        Filter: ((title)::text ~~ 'Book_%'::text)
        Rows Removed by Filter: 3
    Planning Time: 1.786 ms
    Execution Time: 0.253 ms
    ```
    
    *Объясните результат:*

    В этом плане выполнения, база данных использует индекс `t_books_desc_idx`, который отсортирован в обратном порядке по `title`, что позволяет выполнять запрос эффективнее с помощью сортировки результатов в обратном порядке без доп сортировок. Вместо дополнительной сортировки результатов, база данных может напрямую использовать отсортированные данные из индекса

    Такой индекс особенно полезен в ситуации. когда нужно ограниченное количество последних записей
