Для выполнения домашнего задания взята демо бд «Авиаперевозки» https://postgrespro.ru/education/demodb.
Структура таблиц https://edu.postgrespro.ru/demo-20161013.pdf

Скачиваем архив с БД и распаковываем:
```
wget https://edu.postgrespro.ru/demo-small.zip
sudo apt install unzip
unzip demo-small.zip

postgres=# \i  demo-small-20170815.sql
```
## Реализовать прямое соединение двух или более таблиц
Запрос возврашает город, название аэропорта и кол-во вылетевших рейсов на дату '2017-08-01'

```
SELECT a.city,
       a.airport_name,
       count(f.status)
FROM airports a
JOIN flights f ON a.airport_code = f.departure_airport
AND f.scheduled_departure::DATE = '2017-08-01'
GROUP BY city,
         airport_name;

       city       |     airport_name     | count
------------------+----------------------+-------
 Абакан           | Абакан               |     4
 Анадырь          | Анадырь              |     2
 Анапа            | Витязево             |     3
 Архангельск      | Талаги               |     5
 Астрахань        | Астрахань            |     2
 Барнаул          | Барнаул              |     2
 Белгород         | Белгород             |     6
 Белоярский       | Белоярский           |     2
 Благовещенск     | Игнатьево            |     1
 Братск           | Братск               |     1
 Брянск           | Брянск               |    10
 Бугульма         | Бугульма             |     3
 Владивосток      | Владивосток          |     3
 Владикавказ      | Беслан               |     2
 Волгоград        | Гумрак               |     5
 Воркута          | Воркута              |     4
 Воронеж          | Воронеж              |     2
 Геленджик        | Геленджик            |     3
```
## Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

Запрос показывает общее количество рейсов в статусе Departed (Самолет уже вылетел и находится в воздухе) из каждого аэропорта:
``` 
demo=# SELECT a.airport_name, count(f.flight_id)
FROM airports a
LEFT JOIN flights f ON a.airport_code = f.departure_airport
WHERE status = 'Departed'
GROUP BY 1
ORDER BY 2 DESC;

   airport_name   | count
------------------+-------
 Домодедово       |     7
 Шереметьево      |     7
 Сургут           |     4
 Пулково          |     4
 Элиста           |     2
 Омск-Центральный |     2
 Пермь            |     2
 Сочи             |     2
 Хомутово         |     2
 Храброво         |     2
 Бегишево         |     2
 Нефтеюганск      |     1
 Нижневартовск    |     1
 Новый Уренгой    |     1
 Норильск         |     1
 Ноябрьск         |     1
 Хабаровск-Новый  |     1
 Ханты-Мансийск   |     1
 Петрозаводск     |     1
 Псков            |     1
 Богашёво         |     1
 Ростов-на-Дону   |     1
 Рощино           |     1
 Беслан           |     1
 Бесовец          |     1
 Талаги           |     1
```
## Реализовать кросс соединение двух или более таблиц
 
Результат запроса возвращает все возможные напраления перелетов из всех аэропортов:
```
demo=# SELECT a1.airport_name, a2.airport_name
FROM airports a1
CROSS JOIN airports a2
WHERE a1.airport_code <> a2.airport_code
ORDER BY a1.airport_name;

     airport_name     |     airport_name
----------------------+----------------------
 Абакан               | Чульман
 Абакан               | Якутск
 Абакан               | Витязево
 Абакан               | Барнаул
 Абакан               | Мурманск
 Абакан               | Байкал
 Абакан               | Иркутск
 Абакан               | Братск
 Абакан               | Чита
 Абакан               | Магадан
 Абакан               | Игнатьево
 Абакан               | Гумрак
 Абакан               | Сочи
 Абакан               | Ростов-на-Дону
 Абакан               | Омск-Центральный
 Абакан               | Череповец
 Абакан               | Толмачёво
 Абакан               | Уфа
 Абакан               | Оренбург-Центральный
 Абакан               | Казань

```
## Реализовать полное соединение двух или более таблиц
```
Результат запроса возвращает информацию о пассажирах (номер билета, класс,стоимость и данные пассажира), которые летят классом обслуживания Business
SELECT tf.ticket_no, tf.fare_conditions, tf.amount, t.passenger_name, t.contact_data
FROM ticket_flights tf
FULL JOIN tickets t ON  tf.ticket_no = t.ticket_no
WHERE tf.fare_conditions = 'Business';

   ticket_no   | fare_conditions |  amount   |     passenger_name      |                                     contact_data
---------------+-----------------+-----------+-------------------------+---------------------------------------------------------------------------------------
 0005432000990 | Business        |  18500.00 | ALINA VOLKOVA           | {"email": "volkova.alina_03101973@postgrespro.ru", "phone": "+70582584031"}
 0005432000991 | Business        |  18500.00 | MAKSIM ZHUKOV           | {"email": "m-zhukov061972@postgrespro.ru", "phone": "+70149562185"}
 0005432000998 | Business        |  18500.00 | ILYA POPOV              | {"email": "popov_ilya.1971@postgrespro.ru", "phone": "+70003638926"}
 0005432001003 | Business        |  18500.00 | VALENTINA NIKITINA      | {"email": "nikitinavalentina.1975@postgrespro.ru", "phone": "+70794132478"}
 0005432001016 | Business        |  18500.00 | ANZHELIKA ABRAMOVA      | {"phone": "+70518693810"}
 0005432001019 | Business        |  18500.00 | VITALIY ALEKSEEV        | {"email": "alekseev-v.101975@postgrespro.ru", "phone": "+70816024815"}
 0005432001025 | Business        |  18500.00 | MARIYA FEDOROVA         | {"email": "m.fedorova-011974@postgrespro.ru", "phone": "+70868265157"}
 0005432001029 | Business        |  18500.00 | VLADIMIR ANTONOV        | {"email": "vladimir.antonov.121972@postgrespro.ru", "phone": "+70254846003"}
```
## Реализовать запрос, в котором будут использованы разные типы соединений

Результат возвращает количество пассажиров, которые вылетали с аэропорта YKS (Якутск)
```
demo=# SELECT count(*)mit 10;
FROM (ticket_flights t
      JOIN flights f ON t.flight_id = f.flight_id)
LEFT JOIN boarding_passes b ON t.ticket_no = b.ticket_no
AND t.flight_id = b.flight_id
WHERE f.departure_airport = 'YKS';
 count
-------
  2584
(1 row)

```
