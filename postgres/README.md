# Реляционные СУБД (на примере PostgreSQL)

[Презентация](https://docs.google.com/presentation/d/1pBgfRYDzzZgrXdsKZUiLSJa_Loj2-VpUHQUiU_4oJWs/edit?usp=sharing)

Данный раздел состоит из трех частей. Для специалистов начального, среднего и
продвинутого уровня. Другими словами, разделы содержат список тем, в котроых
специалист, соответствующего уровня, должен ориентироваться. Структуру оглавления
можно использовать в качестве карты обучения в будущем.

## Содержание

1. [Введение](#Введение)
2. [Подготовка окружения](#Подготовка-окружения)
3. [Начальный уровень](#Начальный-уровень)
   * [Нормальные формы](#Нормальные-формы)
   * [Отношения](#Отношения)
   * [Транзакции](#Транзакции)
   * [SQL](#sql)
   * [Индексы](#Индексы)
   * [План выполнения запроса](#План-выполнения-запроса)
4. [Средний уровень](#Средний-уровень)
   * [Представления](#Представления)
   * [Триггеры](#Триггеры)
   * [Хранимые процедуры](#Хранимые-процедуры)
5. [Продвинутый уровень](#Продвинутый-уровень)
   * [Партицонирование](#Партиционирование)
   * [Репликация](#Репликация)
   * [Балансировка нагрузки](#Балансировка-нагрузки)
   * [Интеграция со сторонними БД](#Интеграция-со-сторонними-БД)
   * [Резервное копирование](#Резервное-копирование)
   * [Шардинг](#Шардинг)

## Введение

Идея реляционных СУБД состоит в представлении многомерных данных на плоскости.
Т.е. в виде плоских таблиц и отношений между ними. Такой подход позволяет упрощать
достаточно сложные по вложенности структуры данных. Эта технология появилась в
начале 80-х годов 20-го столетия, достаточно быстро завоевала популярность, и на
данный момент является стандартом дефакто во многих областях обработки данных,
однако в области BigData постепенно вымещается современными NoSQL системами
управления данных. Пока реляционные СУБД являются наиболее распространенными
системами управления данными.

В качестве представителей мира реляционых СУБД можно назвать: PostgreSQL,
MySQL, Oracle, DB/2, MSSQL, SyBase и т.д.

## Подготовка окружения

Окружение для выполнения примеров организованно на основе системы контейнеризации
Docker. Это позволит Вам опробовать СУБД в работе с минимальным воздействием на
Вашу систему и не потребует достаточно трудоемкой установки и настройки СУБД.

### Запуск PostgreSQL
```sh
# Создаем том для данных
$ docker volume create pgdata
```

```sh
# Запускаем контейнер с СУБД
 $ docker run --name postgres -d -v pgdata:/var/lib/postgresql/ postgres:10.1
```

### Загрузка данных

```sh
# Создаем БД
$ docker run -it --rm --link postgres postgres:10.1 \
  psql -h postgres -U postgres -c "CREATE DATABASE store"
```

```sh
# Создаем структуру БД
 $ cat structure.sql | docker run -i --rm --link postgres postgres:10.1 \
   psql -h postgres -U postgres -d store
```

```sh
# Загружаем данныe компаний
 $ cat companies.csv | docker run -i --rm --link postgres postgres:10.1 \
   psql -h postgres -U postgres -d store -c "\COPY companies FROM STDIN WITH CSV"
```

```sh
# Загружаем данныe компаний
 $ cat products.csv | docker run -i --rm --link postgres postgres:10.1 \
   psql -h postgres -U postgres -d store -c "\COPY products FROM STDIN WITH CSV"
```

```sh
# Загружаем данныe предложений
 $ cat sentences.csv | docker run -i --rm --link postgres postgres:10.1 \
   psql -h postgres -U postgres -d store -c "\COPY sentences FROM STDIN WITH CSV"
```

### Запуск консоли СУБД
```sh
$ docker run -it --rm --link postgres postgres:10.1 \
  psql -h postgres -U postgres -d store
```

## Начальный уровень

### Нормальные формы
> Нужно для корректного проектирования структуры БД

**Нормализация данных** - это процесс разделения неструктурированных данных на отдельные таблицы и определения связей между ними (таблицами).

Существует 5 нормальных форм. Чем больше номер нормальной формы, тем более
идеальной является структура данных в БД. На практике применяется, в основном, только первые три
формы, оставшиеся две, как правило, приводят к чрезмерному расходу вычислительных
ресурсов для их поддержания.

- [[Очень хорошее описание нормальных форм с примерами]](https://support.microsoft.com/ru-ru/help/283878/description-of-the-database-normalization-basics)

### Отношения
> Нужно для четкого понимания связанности данных в БД

Существует ограниченное кол-во видов отношений между таблицами, а именно:

**один-к-одному** - cлучай когда каждой записи в левой таблице соответствует только одна запись в правой.

**один-ко-многим** - cлучай когда одной записи в левой таблице соответствует много записей в правой.
В свою очередь, каждой записи в правой таблице соответствует только одна запись в левой.

**многие-ко-многим** - cлучай когда одной записи в левой таблице соответствует много записей в правой.
В свою очередь, каждой записи в правой таблице соответствует много записей в левой.
Такая схема в реляционных БД создается при помощи вспомагательной таблицы,
которую называют промежуточной или кросс-таблицей.

Так же существует возможность создания полиморфных связей. Это когда одно и то же
ключевое поле используется для связи с несколькими таблицами. Но данный вид
связи нежелателен, и является одним из видов денормализации данных. Используется,
в основном, для экономии вычислительных ресурсов.

## Транзакции
> Нужны для корректного проведения групповых операций обновления, вставки и удаления

Принцип работы транзакций - либо все, либо ничего. Т.е. если хоть одна из группы команд внутри одной транзакции приведет к ошибке, все комманды из этой группы будут отменены.

> Нужно всегда помнить, если требуется сделать изменения связанных данных в нескольких
таблицах одновременно - ИСПОЛЬЗУЙ ТРАНЗАКЦИИ.

> Не делайте длинных по времени и по вычислительному объему транзакций. Это ведет
к чрезмерному расходу вычислительных ресурсов и к блокировкам других
операций. Старайтесь разбивать транзакции на более мелкие и короткие.

- [[Более детальное описание термина *транзакция*]](https://ru.wikipedia.org/wiki/%D0%A2%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D0%B8%D1%8F_(%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%D0%B8%D0%BA%D0%B0))

#### Демонстрация работы механизма транзакций

Попробуем модифицировать данные в таблице *products* используя транзакцию,
и удостовериться, что никто другой, кроме нас, не видит, сделанных нами
изменений до тех пор, пока мы не завершим транзакцию.


```sql
-- Отобразим исходное состояние данных:
SELECT * FROM products WHERE company_id IN (6,7) ORDER by company_id, name;
```

```
 id |              name               | company_id
----+---------------------------------+------------
  5 | Pasta - Tortellini, Fresh       |          6
  3 | Turkey - Breast, Boneless Sk On |          6
  2 | Broccoli - Fresh                |          7
  4 | Ecolab - Medallion              |          7
(4 rows)
```

```sql
-- Начало транзакции
BEGIN;
-- Модифицируем таблицу внутри транзакции
UPDATE products SET company_id = 7 WHERE company_id = 6;

-- Тут же, внутри транзакции, проверим данные
SELECT * FROM products WHERE company_id IN (6,7) ORDER BY company_id, name;
```

```
 id |              name               | company_id
----+---------------------------------+------------
  2 | Broccoli - Fresh                |          7
  4 | Ecolab - Medallion              |          7
  5 | Pasta - Tortellini, Fresh       |          7
  3 | Turkey - Breast, Boneless Sk On |          7
(4 rows)
```
Для нас таблица изменилась

```sql
-- Cделаем выборку из другой консоли:
SELECT * FROM products WHERE company_id IN (6,7) ORDER BY company_id, name;
```

```
 id |              name               | company_id
----+---------------------------------+------------
  5 | Pasta - Tortellini, Fresh       |          6
  3 | Turkey - Breast, Boneless Sk On |          6
  2 | Broccoli - Fresh                |          7
  4 | Ecolab - Medallion              |          7
(4 rows)
```

Другие пользователи(соединения с БД) не видят наших изменений до тех пор, пока мы не применим нашу транзакцию

```sql
-- Применение тразакции
COMMIT;
```

```sql
-- Иначе, для отмены всех сделанных нами изменений, внутри транзакции, следует выполнить
ROLLBACK;
```
### SQL
> Язык запросов - нужен для работы с данными, а точнее, для их вставки, чтения, изменения и удаления.

Содержит 4 основных оператора:
#### INSERT

Вставка данных в таблицу.

Оператор *INSERT* сотоит из секций:
```sql
-- Вставить в
INSERT INTO table_name (filed1, fieldn)

-- Список значений
VALUES (value1, valuen);
```
#### SELECT

Выборка данных из таблицы.

Оператор *SELECT* состоит из секций:

```sql
-- Список полей
SELECT field1, fieldn

-- Базовая таблица
FROM table_one

-- Таблицы для связывания
LEFT/RIGHT/FULL JOIN table_two ON table_one.filed = table_two.field

-- Условия отбора данных (фильтры)
WHERE ...

-- Группировка/агрегация
GROUP BY field1, fieldn

-- Фильтр по уже сгруппированным данным
HAVING ...

-- Сортировка
ORDER BY field1 ASC/DESC, field2 ASC/DESC

-- Ограничение результатов
LIMIT value

-- Сдвиг от первого результата
OFFSET value
```

"Больное" место большинства начинающих (а иногда и довольно опытных) программистов - это
связывание данных.
Достаточно большое число последних просто не понимают как это происходит
и возлагают все надежы на ORM.

Рассмотрим основные три типа связывания данных.

```sql
-- Таблица companies
SELECT * FROM companies;
```

```
 id |    name
----+-------------
  1 | Youtags
  2 | Babblestorm
  3 | Jaxworks
  4 | Aivee
  5 | Gevee
  6 | Twimbo
  7 | Jaxbean
  8 | Voonix
  9 | Edgetag
(9 rows)
```

```sql
-- Таблица products
SELECT * FROM products;
```

```
 id |              name               | company_id
----+---------------------------------+------------
  1 | Cheese - St. Paulin             |          9
  2 | Broccoli - Fresh                |          7
  4 | Ecolab - Medallion              |          7
  6 | Rice Paper                      |          8
  7 | Latex Rubber Gloves Size 9      |        149
  8 | Crackers - Soda / Saltins       |        197
  9 | Higashimaru Usukuchi Soy        |        101
  3 | Turkey - Breast, Boneless Sk On |          7
  5 | Pasta - Tortellini, Fresh       |          7
(9 rows)
```

*INNER JOIN*

Смысл INNER JOIN - сопоставить данные по ключам из обеих
таблиц, отбросив несовпадения:
```sql
SELECT * FROM companies INNER JOIN products ON companies.id = products.company_id;
```

```
 id |  name   | id |              name               | company_id
----+---------+----+---------------------------------+------------
  9 | Edgetag |  1 | Cheese - St. Paulin             |          9
  7 | Jaxbean |  2 | Broccoli - Fresh                |          7
  7 | Jaxbean |  4 | Ecolab - Medallion              |          7
  8 | Voonix  |  6 | Rice Paper                      |          8
  7 | Jaxbean |  3 | Turkey - Breast, Boneless Sk On |          7
  7 | Jaxbean |  5 | Pasta - Tortellini, Fresh       |          7
(6 rows)
```

*LEFT JOIN*

 Смысл LEFT JOIN - сопоставить данные по ключам из обеих
таблиц, не отбрасывая несовпадения слева. Или другими словами взять все слева
и подставить то, что нашлось справа.
```sql
SELECT * FROM companies LEFT JOIN products ON companies.id = products.company_id;
```

```
 id |    name     | id |              name               | company_id
----+-------------+----+---------------------------------+------------
  9 | Edgetag     |  1 | Cheese - St. Paulin             |          9
  7 | Jaxbean     |  2 | Broccoli - Fresh                |          7
  7 | Jaxbean     |  4 | Ecolab - Medallion              |          7
  8 | Voonix      |  6 | Rice Paper                      |          8
  7 | Jaxbean     |  3 | Turkey - Breast, Boneless Sk On |          7
  7 | Jaxbean     |  5 | Pasta - Tortellini, Fresh       |          7
  6 | Twimbo      |    |                                 |
  2 | Babblestorm |    |                                 |
  5 | Gevee       |    |                                 |
  4 | Aivee       |    |                                 |
  1 | Youtags     |    |                                 |
  3 | Jaxworks    |    |                                 |
(12 rows)
```

*RIGHT JOIN*

Смысл RIGHT JOIN - сопоставить данные по ключам из обеих
таблиц, не отбрасывая несовпадения справа. Или другими словами взять все справа
и подставить то, что нашлось слева.
```sql
SELECT * FROM companies RIGHT JOIN products ON companies.id = products.company_id;
```

```
 id |  name   | id |              name               | company_id
----+---------+----+---------------------------------+------------
  9 | Edgetag |  1 | Cheese - St. Paulin             |          9
  7 | Jaxbean |  2 | Broccoli - Fresh                |          7
  7 | Jaxbean |  4 | Ecolab - Medallion              |          7
  8 | Voonix  |  6 | Rice Paper                      |          8
    |         |  7 | Latex Rubber Gloves Size 9      |        149
    |         |  8 | Crackers - Soda / Saltins       |        197
    |         |  9 | Higashimaru Usukuchi Soy        |        101
  7 | Jaxbean |  3 | Turkey - Breast, Boneless Sk On |          7
  7 | Jaxbean |  5 | Pasta - Tortellini, Fresh       |          7
(9 rows)
```

*FULL JOIN*

Смысл FULL JOIN - сопоставить данные по ключам из обеих
таблиц, не отбрасывая несовпадения как слева, так и справа.
```sql
SELECT * FROM companies FULL JOIN products ON companies.id = products.company_id;
```

```
 id |    name     | id |              name               | company_id 
----+-------------+----+---------------------------------+------------
  9 | Edgetag     |  1 | Cheese - St. Paulin             |          9
  7 | Jaxbean     |  2 | Broccoli - Fresh                |          7
  6 | Twimbo      |  3 | Turkey - Breast, Boneless Sk On |          6
  7 | Jaxbean     |  4 | Ecolab - Medallion              |          7
  6 | Twimbo      |  5 | Pasta - Tortellini, Fresh       |          6
  8 | Voonix      |  6 | Rice Paper                      |          8
    |             |  7 | Latex Rubber Gloves Size 9      |        149
    |             |  8 | Crackers - Soda / Saltins       |        197
    |             |  9 | Higashimaru Usukuchi Soy        |        101
  2 | Babblestorm |    |                                 |           
  5 | Gevee       |    |                                 |           
  4 | Aivee       |    |                                 |           
  1 | Youtags     |    |                                 |           
  3 | Jaxworks    |    |                                 |           
(14 rows)
```

#### UPDATE

Обновить данные

Оператор *UPDATE* состоит из секций:
```sql
-- Обновить
UPDATE table_name

-- Список полей и значений для обновления
SET field1=..., filedn=...

-- Условия отбора данных (фильтры) для обновления
WHERE ...
```
#### DELETE

Удалить данные

Оператор *DELETE* состоит из секций:
```sql
-- Удалить из
DELETE FROM table_name

-- Условия удаления
WHERE ...
```
### Индексы
> Нужны для ускорения поиска и связывания данных

Практически любая современная реляционная СУБД предоставляет программисту
достаточно широкий набор индексов для различных нужд. Например, для поиска по
текстам или для поиска по гео-данным и т.д.

Более подробную информацию по индексам можно получить из документации на конкретную СУБД.

- [[Серия статей о индексах в PostgreSQL]](https://habrahabr.ru/company/postgrespro/blog/326096/)

#### Пример демонcтрирующий эффект от использования индекса

```sql
-- Проверим время выполнения запроса без индекса
EXPLAIN ANALYZE SELECT * FROM sentences WHERE sentence ILIKE '%onusor%';
```

```
                                                QUERY PLAN
----------------------------------------------------------------------------------------------------------
 Seq Scan on sentences  (cost=0.00..2810.00 rows=10 width=94) (actual time=0.023..138.249 rows=1 loops=1)
   Filter: ((sentence)::text ~~* '%onusor%'::text)
   Rows Removed by Filter: 99999
 Planning time: 0.472 ms
 Execution time: 138.266 ms
(5 rows)

Time: 139.214 ms
```
Полученный результат говорит, что время выполнения составило 139,214 мс. 
По плану выполнения запроса мы видим, что для выполнения данного запроса
был применен последовательный перебор данных с фильтрацией. Всего в данной
таблице 100 000 записей. При фильтрации было отброшено 99 999 записей.
Другими словами, не самый эффективный способ поиска.

```sql
-- Создадим индекс для поиска по тексту
CREATE EXTENSION pg_trgm;
CREATE INDEX ON sentences USING GIN (sentence gin_trgm_ops);

-- Повторим запрос (СУБД автоматически подбирает самый подходящий индекс для запроса)
EXPLAIN ANALYZE SELECT * FROM sentences WHERE sentence ILIKE '%onusor%';
```

```
                                                           QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on sentences  (cost=52.08..89.80 rows=10 width=94) (actual time=0.407..0.407 rows=1 loops=1)
   Recheck Cond: ((sentence)::text ~~* '%onusor%'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on sentences_sentence_idx  (cost=0.00..52.08 rows=10 width=0) (actual time=0.398..0.398 rows=1 loops=1)
         Index Cond: ((sentence)::text ~~* '%onusor%'::text)
 Planning time: 0.419 ms
 Execution time: 0.435 ms
(7 rows)

Time: 1.378 ms
```
Результат 1.378 мс. против 139,214 мс

### План выполнения запроса
> Нужен для анализа эффективности индексов и самого запроса

Проверить план выполнения запроса очень просто, для этого достаточно к запросу
добавить слово EXPLAIN. 

```sql
EXPLAIN SELECT * FROM sentences WHERE sentence ILIKE '%onusor%';
```

```
 Результат читается снизу вверх:

 explain select * from sentences where sentence ilike '%onusor%';
                                      QUERY PLAN
---------------------------------------------------------------------------------------
 Bitmap Heap Scan on sentences  (cost=52.08..89.80 rows=10 width=94)
   Recheck Cond: ((sentence)::text ~~* '%onusor%'::text)
   ->  Bitmap Index Scan on sentences_sentence_idx  (cost=0.00..52.08 rows=10 width=0)
         Index Cond: ((sentence)::text ~~* '%onusor%'::text)
```

## Средний уровень
### Представления
> Нужны для автоматизации частых выборок данных

Как правило, существуют два вида представлений: классические и материализованные.
Классические представления это ничто иное, как сохраненный на сервере запрос
к результатам которого можно обращаться как к таблице. Обычные представления
можно использовать только для чтения, но некоторые СУБД позволяют модифицировать
данные в исходной таблице при наличии первичного ключа. Маетриализованные представления
отличаются тем, что в них результаты выборки хранятся статически, а не выбираются
при каждом обращении. Для материалзованных представлений можно создавать индексы.
Благодаря этой возможности, материализованные представления являются мощным
инструментом для оптимизации поисковых запросов для данных которые не часто меняются.
Материализованные представления требуют обновления данных, в случае, если данные
в исходных таблицах были изменены.

```sql
-- Классическое представление
CREATE VIEW sorted_companies AS SELECT * FROM companies ORDER BY name ASC;

-- Обращение к представлению как к таблице
SELECT * FROM sorted_companies;
```

```sql
-- Классическое представление
CREATE MATERIALIZED VIEW materilized_sorted_companies AS SELECT * FROM companies ORDER BY name ASC;

-- Обращение к представлению как к таблице
SELECT * FROM materialized_sorted_companies;

-- После обновления данных в исходной таблице материализованное представление следует обновить
REFRESH MATERIALIZED VIEW materialized_sorted_companies;
```

### Триггеры
> Нужны для создания ответных реакций на изменения в данных.

Триггеры позволяют сеелать любые запрограммированные нами действия при наступлении
определенного события в таблице. Например, сделать что-то автоматически после вставки данных,
или перед обновлениием записи какого-то определенного типа или после ее удаления и т.д.

> СТАРАЙТЕСЬ НЕ ИСПОЛЬЗОВАТЬ ТРИГГЕРЫ, если это возможно. Большое кол-во триггеров
делает работу с БД труднопредсказуемой. Но это не табу, есть случаи, когда
триггеры являются очень гибким и производительным инструментом.

- [Более подробно о триггерах в PostgreSQL](https://www.postgresql.org/docs/10/static/sql-createtrigger.html)

### Хранимые процедуры
> Нужны для серверной обработки данных, а так же для очень сложных выборок.

Хранимые процедуры позволяют производить вычисления на стороне сервера.
Делать сложные выборки и т.д. Каждая СУБД имеет свой синтаксис языка
для создания хранимых процедур, который, как правило, имеет название подобное
PL/SQL. В современном мире, использование (особенно это касается отладки)
весьма затруднительно. Также, данный язык не позволяет делать параллельные
вычисления. И я встречаю все реже и реже БД в которых бизнес-логика основана на
хранимых процедурах. Но как иструмент для описания действий триггеров и выполнения
сложных выборок они вполне подходят.

- [Более подробно о хранимых процедурах в PostgreSQL](https://www.postgresql.org/docs/10/static/plpgsql.html)

## Продвинутый уровень
### Партиционирование
> Нужно для ускорения доступа к очень длинным таблицам

Идея партиционирования состоит в том, чтобы очень длинные таблицы разбить
на подтаблицы, но для пользователя исходная таблица остается монолитной.
Это позволяет выполнять части запросов параллельно с каждой подтаблицей отедльно.

> В PostgreSQL 10 механизм партиционирования несколько изменился, поэтому имейте это в виду.
Старая документация будет не действительна.

- [Очень хорошая статья о партиционировании данных в PostgreSQL](https://habrahabr.ru/post/273933/)

### Репликация
> Нужна для резервирования(отказоустойчивости) и для обеспечения возможности балансировки нагрузки

Реплиикация - это процедура создания и поддержания точных копий основного сервера.
Используется как для "гарячей" замены в случае выхода основного сервера из
строя, так и для балансировки запросов. Существует два вида репликации:

* Мастер - Мастер
* Мастер - Слейв

В случае *Мастер - Мастер* - сервера абсолютно идентичны, и чтение/запись может проводится на
любой из них. Это достаточно редкий вид репликации, потому как он требует
механизма поддержки транзакций между серверами. Для реализации такого вида репликации
существуют готовые кластерные решения. Одно из них [BDR](http://bdr-project.org/docs/stable/)

Более распространненым видом репликации в реляционных БД является конфигурация
*Мастер - Cлейв*. В данном случае, запись можно производить на основной сервер, а
чтение с любой реплики. Такая конфигурация более проста в организации, и позволяет
достаточно простыми средствами организовать балансировку запросов на чтение 
(равномерное распределение запросов на чтение между серверами)

PostgreSQL поддерживает несколько механизмов репликации: поточную и логическую.
Более подробно об этом можно прочитать в документации на PostgreSQL.

### Балансировка нагрузки
> Нужна для распределения нагрузки между узлами кластера (репликами)

Некоторые СУБД имеют данный механизм в своем составе, для некоторых существуют
сторонние инструменты. PostgreSQL не содержит встроенного механизма
балансировки и использует внешние. Наиболее популярными являются
[PgPool](https://www.pgpool.net/) и [PgBouncer](https://pgbouncer.github.io/)


### Интеграция со сторонними БД
> Нужна для обмена данными с другими СУБД

Достаточно часты случаи, когда приходится работать с данными во внешних БД или
предоставлять данные во внешние БД. Одним из популярных механизмов интеграции БД
является федерализация таблиц. Другими словами, для пользователя данные из внешней БД
выглядят как обычная таблица, с которой он может выполнять любые запросы
(в реальности с определенными ограничениями)

- [Более подробно о федерализации в PostgreSQL](https://www.slideshare.net/jim_mlodgenski/postgresql-fedaration)

### Резервное копирование
> Нужно для восстановления данных в случае сбоя

Организация резервного копирования достаточно сложный процесс, особенно, для
больших БД. Это уже традицонно, что большинство БД не имеет нормальных
отработанных механизмов резервного копирования в своем составе
(особенно это касается свободных продуктов).
Как правило, данная задача решается сторонними инструментами.
Советую обратить внимание на [PgBackRest](http://pgbackrest.org/)

### Шардинг
> Нужен для масштабирования БД для больших нагрузок и объемов данных

Рано или поздно наступает время, когда один сервер, даже самый мощный, не
справляется с нагрузкой. А механизм репликации способен эффективно решить
лишь проблемы с чтением относительно небольших таблиц. Когда же данные становятся
настолько большими, что работа с ними становится неэффективна, прибегают к шардированию.
Шардирование реляционных БД -это очень больной вопрос, как для программиста,
так и для администратора. Почти всегда, это достаточно "кастомное" решение
называемое в кругу посвященных "велосипедом".
Существуют попытки более системного подхода к шардированию, но как показывает практика,
они все равно очень сложны в поддержке. На практике удается достичь определенного
компромиcса, добавив в систему NoSQL решения.

Готовые механизмы шардирования
- [Postgres-XL](https://www.postgres-xl.org/)
- [Citus](https://www.citusdata.com/)