# Домашнее задание к занятию "Индексы" - Дорофеев Максим

## Задание 1

Процентное отношение общего размера всех индексов к общему размеру всех таблиц (БД sakila).

    SELECT
      ROUND(SUM(INDEX_LENGTH) * 100 / SUM(DATA_LENGTH), 2) AS indexes_to_tables_pct
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = 'sakila'
      AND TABLE_TYPE = 'BASE TABLE';

Результат: 54.68

## Задание 2

### Исходный запрос (EXPLAIN ANALYZE)

    SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name),
           SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
    FROM payment p, rental r, customer c, inventory i, film f
    WHERE DATE(p.payment_date) = '2005-07-30'
      AND p.payment_date = r.rental_date
      AND r.customer_id = c.customer_id
      AND i.inventory_id = r.inventory_id;

Кратко по плану до оптимизации:
- ~14564 ms, обработано ~642000 строк, на выходе 391
- Inner hash join (no condition): film не связан JOIN-ом → декартово произведение
- DATE(payment_date) / cast → table scan по payment
- WINDOW + DISTINCT вместо нормальной группировки
- слабая связь payment_date = rental_date

### Узкие места
1. Нет связи inventory.film_id = film.film_id (лишние строки).
2. DATE() на колонке мешает индексу.
3. Устаревшие запятые в FROM вместо JOIN.
4. Тяжёлые WINDOW/DISTINCT.

### Оптимизация

Индекс:

    CREATE INDEX idx_payment_date ON payment (payment_date);

Запрос:

    SELECT
      CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
      f.title,
      SUM(p.amount) AS amount_sum
    FROM payment p
    JOIN rental r ON r.rental_id = p.rental_id
    JOIN customer c ON c.customer_id = r.customer_id
    JOIN inventory i ON i.inventory_id = r.inventory_id
    JOIN film f ON f.film_id = i.film_id
    WHERE p.payment_date >= '2005-07-30'
      AND p.payment_date <  '2005-07-31'
    GROUP BY c.customer_id, c.last_name, c.first_name, f.title;

Кратко по плану после:
- ~15 ms, 634 строки
- Index range scan по idx_payment_date
- обычные nested loop JOIN по PRIMARY
- обычные nested loop JOIN по PRIMARY
