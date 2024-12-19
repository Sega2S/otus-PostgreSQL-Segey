Описание/Пошаговая инструкция выполнения домашнего задания:

Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
## Цель: Создать триггер для поддержки витрины в актуальном состоянии

В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).

Есть запрос для генерации отчета – сумма продаж по каждому товару.

БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.

Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

```
postgres=# CREATE DATABASE otus;
CREATE DATABASE

-- ДЗ тема: триггеры, поддержка заполнения витрин

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;
CREATE SCHEMA
SET search_path = pract_functions, publ
SET
-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
CREATE TABLE

-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.

```
Заполняем данными таблицу витрины:
```
INSERT INTO good_sum_mart SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
INSERT 0 2

SELECT * FROM good_sum_mart;

        good_name         |  sum_salen
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

Создадим триггер на вставку в таблицу sales:
```
CREATE OR REPLACE FUNCTION add_sales() RETURNS TRIGGER AS
$BODY$
begin
	if  not exists (select 1 from good_sum_mart GM inner join goods G on G.good_name=GM.good_name where G.goods_id=new.good_id)
	then 
	INSERT into good_sum_mart(good_name,sum_sale) select good_name,0 from goods where goods_id=new.good_id;
	end if;
	update good_sum_mart set sum_sale=(SELECT sum(G.good_price * S.sales_qty) FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id where G.goods_id = new.good_id)
	where good_name = (select GM.good_name from good_sum_mart GM inner join goods G on G.good_name=GM.good_name where G.goods_id=new.good_id);
	RETURN new;   
END;
$BODY$
language plpgsql;
CREATE FUNCTION

CREATE TRIGGER trigger_add_sales after INSERT ON sales FOR EACH row EXECUTE PROCEDURE add_sales();
CREATE TRIGGER
```

Создадим триггер для обновления наименования товара в таблице goods:
```
CREATE OR REPLACE FUNCTION update_name_goods() RETURNS TRIGGER AS
$BODY$
begin
	if  exists (select 1 from good_sum_mart GM inner join goods G on G.good_name=GM.good_name where G.goods_id=old.goods_id)
	then 
	update good_sum_mart set good_name = new.good_name where good_name in (select GM.good_name from good_sum_mart GM inner join goods G on G.good_name=GM.good_name where G.goods_id=old.goods_id);	
	end if;
	RETURN new;   
END;
$BODY$
language plpgsql;
CREATE FUNCTION

CREATE TRIGGER trigger_update_name_goods BEFORE UPDATE ON goods FOR EACH row EXECUTE PROCEDURE update_name_goods();
CREATE TRIGGER
```
Создадим триггер для актуализации цен витрины:
```
CREATE OR REPLACE FUNCTION update_price_goods() RETURNS TRIGGER AS
$BODY$
begin
	if  exists (select 1 from good_sum_mart GM inner join goods G on G.good_name=GM.good_name where G.goods_id=old.goods_id)
	then 
	update good_sum_mart set sum_sale=(SELECT sum(G.good_price * S.sales_qty) FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id where G.good_name = new.good_name)
	where good_name=new.good_name;
	end if;
	RETURN new;   
END;
$BODY$
language plpgsql;
CREATE FUNCTION

CREATE TRIGGER trigger_update_price_goods AFTER UPDATE ON goods FOR EACH row EXECUTE PROCEDURE update_price_goods();
CREATE TRIGGER
```
Проверяем добавление товара:
```
INSERT INTO goods (goods_id, good_name, good_price) VALUES (3, 'стул компьютерный', 5500);
INSERT 0 1
INSERT INTO sales (good_id, sales_qty) VALUES (3, 2);
INSERT 0 1

SELECT * FROM goods;
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
        3 | стул компьютерный        |      5500.00
(3 rows)

SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
 стул компьютерный        |     11000.00
(3 rows)
```

Проверяем добавление ппродаж:
```
INSERT INTO sales (good_id, sales_qty) VALUES (1, 100), (2, 2), (3, 5);
INSERT 0 3

SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |       115.50
 Автомобиль Ferrari FXX K | 555000000.03
 стул компьютерный        |     38500.00
(3 rows)
```

Проверяем обновление наименования товаров и стоимость:
```
UPDATE goods SET good_name = 'стул компьютерный', good_price=5000 where goods_id=3;
UPDATE 1

SELECT * FROM goods;
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
        3 | стул компьютерный        |      5000.00
(3 rows)

SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |       115.50
 Автомобиль Ferrari FXX K | 555000000.03
 стул компьютерный        |     35000.00
(3 rows)
```
Все работает