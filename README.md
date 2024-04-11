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

-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=5008..5008 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=5008..5008 rows=391 loops=1)
        -> Window aggregate with buffering: sum(sakila.payment.amount) OVER (PARTITION BY sakila.c.customer_id,sakila.f.title )   (actual time=3296..4797 rows=642000 loops=1)
            -> Sort: sakila.c.customer_id, sakila.f.title  (actual time=3296..3369 rows=642000 loops=1)
                -> Stream results  (cost=22.6e+6 rows=16.1e+6) (actual time=5.89..2553 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=22.6e+6 rows=16.1e+6) (actual time=5.71..2156 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=20.9e+6 rows=16.1e+6) (actual time=5.48..1932 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=19.3e+6 rows=16.1e+6) (actual time=4.42..1667 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=3.38..105 rows=634000 loops=1)
                                    -> Filter: (cast(sakila.p.payment_date as date) = '2005-07-30')  (cost=1.9 rows=16086) (actual time=1.19..41.4 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.9 rows=16086) (actual time=1.17..38.5 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=1.52..2 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=sakila.p.payment_date)  (cost=1 rows=1) (actual time=0.00153..0.00229 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=sakila.r.customer_id)  (cost=0.001 rows=1) (actual time=219e-6..245e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=sakila.r.inventory_id)  (cost=925e-6 rows=1) (actual time=179e-6..205e-6 rows=1 loops=642000)

[Explain](https://github.com/sash3939/Index/assets/156709540/87e35f47-b96f-49f1-bb5d-1bad1b563299)

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

