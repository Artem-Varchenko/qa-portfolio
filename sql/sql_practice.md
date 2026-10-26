# SQL Practice Log

## Basics: SELECT, WHERE, ORDER BY, LIMIT

### 1. Останні 4 фільми (за роком)
```sql
-- Задача: last four movies, most recent first
-- Пастка: спершу зробив WHERE year >= 2010, тобто вгадав дані, а не вимогу.
-- "Останні N" це завжди ORDER BY + LIMIT, без хардкоду значень.
SELECT * FROM movies
ORDER BY year DESC
LIMIT 4;
```

### 2. Міста на захід від Чикаго
```sql
-- Задача: cities west of Chicago, west to east
-- Пастка: взяв latitude замість longitude.
-- Захід-схід = longitude, північ-південь = latitude.
SELECT * FROM north_american_cities
WHERE longitude < -87.629798
ORDER BY longitude;
```

## JOIN

### 1. Базовий INNER: зв'язок не через id
```sql
-- Задача: будівлі, у яких є працівники
-- Пастка: ON не обов'язково йде через id. Тут зв'язок через назву будівлі.
-- DISTINCT потрібен, бо в одній будівлі багато працівників
SELECT DISTINCT buildings.building_name
FROM buildings
JOIN employees ON buildings.building_name = employees.building;
```

### 2. LEFT JOIN: зберегти рядки без пари
```sql
-- Задача: всі будівлі та ролі, включно з порожніми будівлями
-- Пастка: "including empty" = LEFT, бо INNER викине будівлі без працівників
SELECT DISTINCT buildings.building_name, employees.role
FROM buildings
LEFT JOIN employees ON buildings.building_name = employees.building;
```

### 3. Знайти те, у чого немає пари
```sql
-- Задача: будівлі без жодного працівника
-- Головний патерн. Три частини працюють разом:
-- FROM buildings: зберігаємо саме будівлі (те, що шукаємо)
-- LEFT JOIN: дозволяє NULL там, де пари нема
-- WHERE ... IS NULL: лишає тільки ті, де пари не знайшлось
SELECT buildings.building_name
FROM buildings
LEFT JOIN employees ON employees.building = buildings.building_name
WHERE employees.building IS NULL;
```

### 4. Self JOIN: працівники, які заробляють більше за свого менеджера (LeetCode 181, MySQL)
```sql
-- Задача: знайти працівників, у яких зарплата більша, ніж у їхнього менеджера
-- Self JOIN: таблиця з'єднується сама з собою. Одна копія грає роль
-- працівника, друга менеджера. Ролі задає ON, а не назва аліасу:
-- e.id = m.managerId означає, що e це менеджер, а m його підлеглий,
-- тому SELECT m.name повертає саме працівника.
-- Нюанс: умова на m.salary у WHERE відкидає рядки без пари,
-- тож LEFT JOIN тут фактично працює як INNER JOIN,
-- а m.managerId IS NOT NULL це зайва (але нешкідлива) перевірка.
SELECT m.name AS Employee
FROM Employee e
LEFT JOIN Employee m ON e.id = m.managerId
WHERE m.salary > e.salary AND m.managerId IS NOT NULL;
```

### 5. Self JOIN + GROUP BY: кількість підлеглих і їхній середній вік (LeetCode 1731, MySQL)
```sql
-- Задача: для кожного менеджера вивести кількість прямих підлеглих
-- і їхній середній вік, округлений до цілого, з сортуванням по employee_id
-- Пастка: спершу згрупував по e.reports_to, тобто менеджерів за ЇХНІМ начальником.
-- Alice і Bob обидва підпорядковані Michael, тому злились в одну групу:
-- Bob зник з результату, а в Alice вийшло 3 підлеглих замість 2.
-- Урок: GROUP BY має йти по тому, для кого рахуємо результат (менеджер e).
-- Механіка: e.employee_id = r.reports_to робить e менеджером, а r підлеглим.
-- INNER JOIN відкидає тих, у кого немає підлеглих, і це якраз потрібно.
-- ROUND(AVG(x)) без другого аргументу округляє до цілого.
SELECT e.employee_id, e.name,
       COUNT(r.reports_to) AS reports_count,
       ROUND(AVG(r.age)) AS average_age
FROM Employees e
JOIN Employees r ON e.employee_id = r.reports_to
GROUP BY e.employee_id
ORDER BY employee_id;
```

## Aggregation: COUNT, SUM, GROUP BY

### 1. Кількість працівників по ролях
```sql
-- Задача: number of Employees of each role in the studio
-- Каркас: GROUP BY розкладає рядки по купках, COUNT рахує кожну окремо
SELECT role, COUNT(role)
FROM employees
GROUP BY role;
```

### 2. Порожній результат ≠ зламаний запит
```sql
-- Задача: total years employed by all Engineers
-- Пастка: спершу написав role = "Engineers" (з різних міркувань додав s).
-- Синтаксис правильний, але WHERE нічого не знайшов, бо в даних "Engineer".
-- Урок: якщо результат порожній, спершу звіряй значення в WHERE з даними,
-- а не одразу лізь міняти логіку запиту.
SELECT SUM(years_employed)
FROM employees
WHERE role = 'Engineer';
```

### 3. JOIN + GROUP BY + SUM(вираз): усе разом
```sql
-- Задача: total domestic + international sales attributed to each director
-- Пастка 1: GROUP BY movie_id замість GROUP BY director. Кожен movie_id
-- унікальний, тому кожна "група" = один фільм, і SUM нічого не сумує.
-- Пастка 2: "total sales" = одне число, тому SUM(a + b) в одному виразі,
-- а не два окремих SUM(a), SUM(b) в різних колонках.
SELECT movies.director,
       SUM(boxoffice.domestic_sales + boxoffice.international_sales) AS total_sales
FROM movies
LEFT JOIN boxoffice ON movies.id = boxoffice.movie_id
GROUP BY movies.director;
```

### 4. COUNT(DISTINCT): унікальні предмети кожного вчителя (LeetCode 2356, MySQL)
```sql
-- Задача: для кожного вчителя порахувати кількість унікальних предметів
-- Пастка 1: спершу згрупував по dept_id і додав HAVING cnt != 1.
-- На прикладі цифри збіглися випадково і тест пройшов, але логіка неправильна.
-- Урок: один пройдений тест не доводить, що запит правильний.
-- Пастка 2: написав DISTINCT COUNT(...) замість COUNT(DISTINCT ...),
-- це синтаксична помилка. DISTINCT пишеться всередині дужок.
-- Пастка 3: рахував dept_id (кафедри) замість subject_id (предмети).
-- Механіка: COUNT(DISTINCT subject_id) дивиться тільки на subject_id,
-- а не на весь рядок. Рядки (1, 2, 3) і (1, 2, 4) дають той самий предмет 2,
-- тому він рахується один раз.
SELECT teacher_id, COUNT(DISTINCT subject_id) AS cnt
FROM Teacher
GROUP BY teacher_id;
```
