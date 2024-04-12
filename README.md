# Домашнее задание к занятию «Индексы»


### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Решение 1
[ratio](https://github.com/sash3939/Index/assets/156709540/12a8fb43-f20b-4f73-929d-78c4d59d312b)

SELECT 
    SUM(index_length) / SUM(data_length) * 100 AS index_ratio
FROM 
    information_schema.tables
WHERE 
    table_schema = 'sakila';

---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Решение 2
EXPLAIN ANALYZE
SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
FROM sakila.payment p, sakila.rental r, sakila.customer c, sakila.inventory i, sakila.film f
WHERE date(p.payment_date) = '2005-07-30' 
  AND p.payment_date = r.rental_date 
  AND r.customer_id = c.customer_id 
  AND i.inventory_id = r.inventory_id;

необходимо добавление INDEX для payment_date:
**CREATE INDEX idx_payment_date ON sakila.payment (payment_date);**

EXPLAIN ANALYZE                        
SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_payment
FROM sakila.payment p
JOIN sakila.rental r ON p.payment_date = r.rental_date
JOIN sakila.customer c ON r.customer_id = c.customer_id
JOIN sakila.inventory i ON r.inventory_id = i.inventory_id
JOIN sakila.film f ON i.film_id = f.film_id
WHERE payment_date >= '2005-07-30' and payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY)

-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=13.6..13.7 rows=602 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=13.6..13.6 rows=602 loops=1)
        -> Window aggregate with buffering: sum(sakila.payment.amount) OVER (PARTITION BY sakila.c.customer_id,sakila.f.title )   (actual time=11.6..13.3 rows=642 loops=1)
            -> Sort: sakila.c.customer_id, sakila.f.title  (actual time=11.5..11.6 rows=642 loops=1)
                -> Stream results  (cost=1022 rows=645) (actual time=0.0621..11 rows=642 loops=1)
                    -> Nested loop inner join  (cost=1022 rows=645) (actual time=0.0542..10.4 rows=642 loops=1)
                        -> Nested loop inner join  (cost=799 rows=645) (actual time=0.0503..9.15 rows=642 loops=1)
                            -> Nested loop inner join  (cost=576 rows=645) (actual time=0.0461..7.92 rows=642 loops=1)
                                -> Nested loop inner join  (cost=351 rows=634) (actual time=0.0305..1.99 rows=634 loops=1)
                                    -> Filter: ((sakila.r.rental_date >= TIMESTAMP'2005-07-30 00:00:00') and (sakila.r.rental_date < <cache>(('2005-07-30' + interval 1 day))))  (cost=129 rows=634) (actual time=0.0205..0.688 rows=634 loops=1)
                                        -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date < '2005-07-31 00:00:00')  (cost=129 rows=634) (actual time=0.0183..0.464 rows=634 loops=1)
                                    -> Single-row index lookup on c using PRIMARY (customer_id=sakila.r.customer_id)  (cost=0.25 rows=1) (actual time=0.00182..0.00185 rows=1 loops=634)
                                -> Index lookup on p using idx_payment_date (payment_date=sakila.r.rental_date)  (cost=0.254 rows=1.02) (actual time=0.00811..0.00913 rows=1.01 loops=634)
                            -> Single-row index lookup on i using PRIMARY (inventory_id=sakila.r.inventory_id)  (cost=0.246 rows=1) (actual time=0.00169..0.00172 rows=1 loops=642)
                        -> Single-row index lookup on f using PRIMARY (film_id=sakila.i.film_id)  (cost=0.246 rows=1) (actual time=0.00167..0.0017 rows=1 loops=642)


[Explain](https://github.com/sash3939/Index/assets/156709540/ae6d33e0-785c-42ef-9850-905fa4772102)


- перечислите узкие места;

1. Фильтрация таблицы payment по дате платежа, если нет индекса на столбец payment_date.
2. Sort: Сортировка результатов запроса может быть неэффективной операцией, особенно если данных много.
3. Covering index scan on f using idx_title: Использование индекса idx_title может быть не эффективным, если данных в title много.

- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
                SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_payment
FROM sakila.payment p
JOIN sakila.rental r ON p.rental_id = r.rental_id
JOIN sakila.customer c ON r.customer_id = c.customer_id
JOIN sakila.inventory i ON r.inventory_id = i.inventory_id
JOIN sakila.film f ON i.film_id = f.film_id
WHERE DATE(p.payment_date) = '2005-07-30';

[optimyze](https://github.com/sash3939/Index/assets/156709540/f118c805-b2ed-4d88-b1ec-ab26c2863ca1)


---

## Дополнительные задания (со звёздочкой*)
Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

*Приведите ответ в свободной форме.*

### Решение 3

- GIN (Generalized Inverted Index): Индекс общего обращения используется в PostgreSQL для индексации данных типа массивов, JSON и полнотекстового поиска.

- GiST (Generalized Search Tree): Индекс обобщенного дерева поиска используется в PostgreSQL для обработки геометрических и других типов данных.

- BRIN (Block Range Index): Индекс диапазона блоков используется в PostgreSQL для индексации упорядоченных данных по диапазонам.

- SP-GiST (Space-Partitioned Generalized Search Tree): Индекс пространственно-разбитого обобщенного дерева поиска используется в PostgreSQL для индексации геометрических данных и других типов данных.

