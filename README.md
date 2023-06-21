# Домашнее задание к занятию «Индексы»

---


### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Ответ:

```sql
SELECT SUM(index_length)/SUM(data_length) * 100
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'sakila'
```
### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Ответ:

В результате получаем такой план запроса:
```
-> Limit: 200 row(s)  (cost=0..0 rows=0) (actual time=5528..5528 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=5528..5528 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=5528..5528 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2607..5359 rows=642000 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=2606..2654 rows=642000 loops=1)
                    -> Stream results  (cost=10.5e+6 rows=16.1e+6) (actual time=0.241..2068 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=10.5e+6 rows=16.1e+6) (actual time=0.237..1737 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=8.85e+6 rows=16.1e+6) (actual time=0.234..1530 rows=642000 loops=1)
                                -> Nested loop inner join  (cost=7.24e+6 rows=16.1e+6) (actual time=0.23..1272 rows=642000 loops=1)
                                    -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=0.22..59.3 rows=634000 loops=1)
                                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.68 rows=16086) (actual time=0.0202..5.04 rows=634 loops=1)
                                            -> Table scan on p  (cost=1.68 rows=16086) (actual time=0.0128..3.27 rows=16044 loops=1)
                                        -> Hash
                                            -> Covering index scan on f using idx_title  (cost=103 rows=1000) (actual time=0.0304..0.147 rows=1000 loops=1)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1) (actual time=0.0012..0.00175 rows=1.01 loops=634000)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=250e-6 rows=1) (actual time=217e-6..241e-6 rows=1 loops=642000)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=250e-6 rows=1) (actual time=124e-6..161e-6 rows=1 loops=642000)

```
Время выполнения запроса очень велико.
Видно, что происходит перебор всей таблицы.


Нужно добавить индекс по полю payment_date:
```sql
CREATE INDEX payment_date_index
ON payment (payment_date)
```

И переписать запрос, используя соединения с таблицами, убрать неиспользуемые таблицы и убрать функцию date для того, чтобы индекс использовался:

```sql
EXPLAIN ANALYZE
SELECT concat(c.last_name, ' ', c.first_name), sum(p.amount)
FROM payment p
JOIN customer c on p.customer_id = c.customer_id
WHERE payment_date >= '2005-07-30' and payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY)
GROUP BY p.customer_id
```
Тогда план запроса будет таким:
```
-> Limit: 200 row(s) (actual time=2.28..2.31 rows=200 loops=1)
-> Table scan on <temporary> (actual time=2.27..2.29 rows=200 loops=1)
-> Aggregate using temporary table (actual time=2.27..2.27 rows=391 loops=1)
-> Nested loop inner join (cost=507 rows=634) (actual time=0.0307..1.84 rows=634 loops=1)
-> Index range scan on p using payment_date_index over ('2005-07-30 00:00:00' <= payment_date < '2005-07-31 00:00:00'), with index condition: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < <cache>(('2005-07-30' + interval 1 day)))) (cost=286 rows=634) (actual time=0.023..0.953 rows=634 loops=1)
-> Single-row index lookup on c using PRIMARY (customer_id=p.customer_id) (cost=0.25 rows=1) (actual time=0.00121..0.00123 rows=1 loops=634)
```
Время выполнения значительно уменьшилось и теперь используются первичные ключи и индекс payment_date_index.
