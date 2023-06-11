# Домашнее задание к занятию «Индексы»

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

![](https://github.com/VolkovMixail/12.5/blob/main/img/12-5-1.png)

---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
![](https://github.com/VolkovMixail/12.5/blob/main/img/12-5-2-1.png)
Узкие места наблюдаются в момент использования оконных функций OVER и PARTITION BY, а  так же в  момент фильтрации вывода, в сравнении колонок из разных таблиц.
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы. Так же избыточная таблица - film.
```sql
create index day_of_payment on payment(payment_date);
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) 
from payment p
join rental r on r.rental_id = p.rental_id 
join customer c ON c.customer_id = p.customer_id 
join inventory i on i.inventory_id = r.inventory_id 
where date(p.payment_date) = '2005-07-30' 
group by concat(c.last_name, ' ', c.first_name); 
```
![]()

---
