
1.  Развернул PostgreSQL на вм
2. Создал таблицы,сотрудников,отделов,локаций,проектов,таблицу связи сотрудников и проектов.
```
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department_id INT,
    manager_id INT
);

CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(50),
    location_id INT
);

CREATE TABLE locations (
    location_id INT PRIMARY KEY,
    city VARCHAR(50),
    country VARCHAR(50)
);

CREATE TABLE projects (
    project_id INT PRIMARY KEY,
    project_name VARCHAR(50),
    budget DECIMAL(10,2)
);

CREATE TABLE employee_projects (
    employee_id INT,
    project_id INT,
    role VARCHAR(50),
    PRIMARY KEY (employee_id, project_id)
);
```
3. Реализовал прямое соединение. Этот запрос показывает только тех сотрудников, у которых указан отдел, и только те отделы, в которых есть сотрудники. Сотрудники без отдела и отделы без сотрудников не будут включены в результат.Выдал список сотрудников с названиями их отделов.
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;
```
4. Реализовал левостороннее соединение. Этот запрос показывает всех сотрудников из таблицы employees, даже если у них нет соответствия в таблице departments. Для таких сотрудников поля из таблицы departments будут NULL. Выдало всех сотрудников, даже если у них не указан отдел.
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```
5. Реализовал кросс-соединение. Кросс-соединение создает декартово произведение строк из обеих таблиц. Каждая строка из таблицы employees соединяется с каждой строкой из таблицы projects. Выдало все возможные комбинации сотрудников и проектов.
```
SELECT e.first_name, e.last_name, p.project_name
FROM employees e
CROSS JOIN projects p;
```
6. Реализовал полное соединение. Полное соединение возвращает все строки из обеих таблиц, соединяя их где есть соответствие. Для строк без соответствия в другой таблице соответствующие поля будут NULL. Выдало все отделы и всех сотрудников, даже если нет соответствий
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```
7. Реализовал запрос, в котором будут использованы разные типы соединений.Получили информацию о проектах, сотрудниках и отделах с использованием INNER, LEFT и RIGHT JOIN
```
SELECT 
    p.project_name,
    p.budget,
    e.first_name AS employee_first_name,
    e.last_name AS employee_last_name,
    d.department_name,
    l.city AS department_location
FROM projects p
INNER JOIN employee_projects ep ON p.project_id = ep.project_id
LEFT JOIN employees e ON ep.employee_id = e.employee_id
RIGHT JOIN departments d ON e.department_id = d.department_id
LEFT JOIN locations l ON d.location_id = l.location_id
ORDER BY p.project_name, d.department_name;
```


