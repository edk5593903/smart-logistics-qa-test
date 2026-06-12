# Блок 3: Анализ данных (SQL)

Диалект: **PostgreSQL**

---

## Задача 1: Вторая по величине уникальная зарплата

**Таблица:** `Employee (id INT PK, salary INT)`

**Условие:** найти вторую по величине уникальную зарплату; если её нет — вернуть `NULL`.

```sql
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

**Логика:**
- Внутренний подзапрос находит максимальную зарплату.
- Внешний `MAX` берёт максимум среди всех зарплат, строго меньших найденного максимума.
- Если второй уникальной зарплаты не существует, `MAX` над пустым множеством вернёт `NULL` — это покрывает требование задачи.
- `DISTINCT` не нужен: `MAX` по определению уникален.

**Альтернатива через `DENSE_RANK` (более гибко для N-й зарплаты):**

```sql
SELECT salary AS SecondHighestSalary
FROM (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM Employee
) ranked
WHERE rnk = 2
LIMIT 1;
```

> `DENSE_RANK` присваивает одинаковый ранг равным значениям, поэтому дубликаты максимума не сдвигают второе место.

---

## Задача 2: Поиск дублирующихся email-адресов

**Таблица:** `Person (id INT PK, email VARCHAR)`

**Условие:** найти все email, которые встречаются более одного раза.

```sql
SELECT email
FROM Person
GROUP BY email
HAVING COUNT(*) > 1;
```

**Логика:**
- `GROUP BY email` группирует записи по значению поля.
- `HAVING COUNT(*) > 1` оставляет только те группы, где email встречается хотя бы дважды.
- `SELECT email` возвращает сам дублирующийся адрес.
- Регистрозависимость: в PostgreSQL `VARCHAR` сравнивается с учётом регистра, поэтому `User@mail.com` и `user@mail.com` — разные значения. При необходимости регистронезависимого поиска добавить `LOWER(email)`.

---

## Задача 3: Клиенты без заказов

**Таблицы:**
- `Customers (id INT PK, name VARCHAR)`
- `Orders (id INT PK, customerId INT FK)`

**Условие:** вернуть всех клиентов, которые ни разу не делали заказов.

```sql
SELECT c.name AS Customers
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;
```

**Логика:**
- `LEFT JOIN` оставляет всех клиентов, даже тех, у кого нет совпадений в `Orders`.
- Для клиентов без заказов все поля из `Orders` будут `NULL`.
- `WHERE o.id IS NULL` фильтрует именно таких клиентов.

**Альтернатива через `NOT EXISTS` (эффективнее при большом `Orders`):**

```sql
SELECT name AS Customers
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.customerId = c.id
);
```

**Альтернатива через `NOT IN` (осторожно с NULL в подзапросе):**

```sql
SELECT name AS Customers
FROM Customers
WHERE id NOT IN (
    SELECT customerId FROM Orders WHERE customerId IS NOT NULL
);
```

> `NOT IN` возвращает пустой результат, если в подзапросе есть хотя бы один `NULL`, — поэтому `WHERE customerId IS NOT NULL` обязателен.

---

## Итог

| Задача | Ключевой приём | Подводный камень |
|--------|---------------|------------------|
| Вторая зарплата | `MAX` + подзапрос / `DENSE_RANK` | `RANK` пропускает номера при дублях — использовать `DENSE_RANK` |
| Дубли email | `GROUP BY` + `HAVING COUNT(*) > 1` | Регистрозависимость в PostgreSQL |
| Клиенты без заказов | `LEFT JOIN ... IS NULL` / `NOT EXISTS` | `NOT IN` опасен при `NULL` в `customerId` |
