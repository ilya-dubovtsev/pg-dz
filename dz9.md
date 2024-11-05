## ДЗ №9
https://www.db-fiddle.com/f/eQ8zNtAFY88i8nB4GRB65V/0
```
CREATE TABLE salary (
    fk_employee int NOT NULL,
    amount int NOT NULL,
    from_date date NOT NULL,
    to_date date NOT NULL,
    fk_grade int NOT NULL references grade(id),
    CONSTRAINT pk_primary PRIMARY KEY (fk_employee, from_date),
    CONSTRAINT salaries_fk FOREIGN KEY (fk_employee) REFERENCES employee(id) ON UPDATE RESTRICT ON DELETE RESTRICT
);

insert into salary (fk_employee,amount,from_date,to_date,fk_grade)
values
(1,100000,'20240101','20240131',1),
(1,200000,'20240201','20240229',2),
(1,300000,'20240301','20991231',3),(2,200000,'20230101','20240131',2),
(3,200000,'20240301','20240131',2);
```
1. Проанализировать данные о зарплатах сотрудников с использованием оконных функций.

а. На сколько было увеличение с предыдущей зарплатой

```
SELECT *, LAG(amount) OVER(PARTITION BY fk_employee ORDER BY from_date)
FROM salary;

fk_employee	amount	from_date	to_date	fk_grade	lag
1	100000	2024-01-01	2024-01-31	1	null
1	200000	2024-02-01	2024-02-29	2	100000
1	300000	2024-03-01	2099-12-31	3	200000
2	200000	2023-01-01	2024-01-31	2	null
3	200000	2024-03-01	2024-01-31	2	null
```

б. если это первая зарплата - вместо NULL вывести 0
```
SELECT *, COALESCE(LAG(amount) OVER(PARTITION BY fk_employee ORDER BY from_date), 0) as diff
FROM salary;

fk_employee	amount	from_date	to_date	fk_grade	diff
1	100000	2024-01-01	2024-01-31	1	0
1	200000	2024-02-01	2024-02-29	2	100000
1	300000	2024-03-01	2099-12-31	3	200000
2	200000	2023-01-01	2024-01-31	2	0
3	200000	2024-03-01	2024-01-31	2	0
```

