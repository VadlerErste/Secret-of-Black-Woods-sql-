# Secret-of-Black-Woods-sql-
Аналитика влияния характеристик игроков и их игровых персонажей  на покупку внутриигровой валюты, оценка активности игроков при совершении внутриигровых покупок

[LysenkovVV_CourseWork_150525.sql](https://github.com/user-attachments/files/33123550/LysenkovVV_CourseWork_150525.sql)

Хост: **rc1b-wcoijxj3yxfsf3fs.mdb.yandexcloud.net**
URL: **jdbc:postgresql://rc1b-wcoijxj3yxfsf3fs.mdb.yandexcloud.net:6432/data-analyst-fantasy**
База данных: **data-analyst-fantasy**
Порт:6432

* Проект «Секреты Тёмнолесья»
 * Цель проекта: изучить влияние характеристик игроков и их игровых персонажей 
 * на покупку внутриигровой валюты «райские лепестки», а также оценить 
 * активность игроков при совершении внутриигровых покупок
 * 
 * Автор: Вадим Лысенков
 * Дата: 15.05.2025(версия 4.0)


-- Часть 1. Исследовательский анализ данных
-- Задача 1. Исследование доли платящих игроков
-- 1.1. Доля платящих пользователей по всем данным:
--общее количество пользователей
SELECT COUNT(id) as count_all_users, --количество зарегистрированных игроков
       SUM(payer) as count_pay_users,--количество платящих игроков
       ROUND((AVG(payer)::numeric),2) as part_pay_all --доля платящих игроков от общего количества зарегистрированных игроков 
from fantasy.users;
-- 1.2. Доля платящих пользователей в разрезе расы персонажа:
SELECT r.race,
	   COUNT(u.id) as cnt_users_race, --количество зарегистрированных игроков
       SUM(u.payer) as cnt_pay_users_race,
       ROUND((AVG(u.payer)::numeric),2) as part_pay_all_race --доля платящих игроков от общего количества зарегистрированных игроков 
from fantasy.users as u
left join fantasy.race as r on u.race_id= r.race_id
group by r.race
order by part_pay_all_race desc;
-- Задача 2. Исследование внутриигровых покупок
-- 2.1. Статистические показатели по полю amount:
SELECT COUNT(DISTINCT transaction_id) AS total_amnt,--общее количество покупок;
       SUM(amount) AS sum_amnt,--суммарная стоимость всех покупок;
	   MAX(amount)AS max_amnt,--максимальная стоимость покупки;
	   MIN(amount)AS min_amnt,--минимальная стоимость покупки;
	   AVG(amount)AS avg_amnt,--среднее значение стоимости покупки;
       STDDEV(amount)AS total_amnt,
       PERCENTILE_DISC(0.5) WITHIN GROUP(ORDER BY amount) AS med--стандартное отклонение стоимости покупки
	   FROM fantasy.events
	   WHERE amount!=0;
-- 2.2: Аномальные нулевые покупки:
	   SELECT (SELECT COUNT(transaction_id)---общее количество нулевых покупок
       FROM fantasy.events
       WHERE amount ='0'),
       	(SELECT COUNT(transaction_id)
             FROM fantasy.events
             WHERE amount=0)/(COUNT(transaction_id)::float)*100 AS part_null_transaction
      FROM fantasy.events;      
-- 2.3: Сравнительный анализ активности платящих и неплатящих игроков:
-- Расчет количества платящих игроков и количества их покупок
SELECT 
    CASE 
        WHEN u.payer = 1 THEN 'Платящий'
        ELSE 'Неплатящий'
    END AS payer_type,---группа пользователей
    COUNT(DISTINCT u.id) AS count_user,---количество уникальных игроков
    COUNT(e.transaction_id) AS total_transactions,---общее количество покупок
    COALESCE(SUM(e.amount::numeric), 0) AS total_amount,---сумма всех покупок
    ROUND(COALESCE(COUNT(e.transaction_id)::numeric / NULLIF(COUNT(DISTINCT u.id), 0), 0),2) AS avg_transactions_user,---среднее количество покупок на 1 игрока
    ROUND(COALESCE(SUM(CAST(e.amount AS numeric))::numeric / NULLIF(COUNT(DISTINCT u.id), 0), 0),2) AS avg_amount_user---средняя стоимость покупки на 1 игрока
FROM fantasy.users AS u
LEFT JOIN fantasy.events e ON u.id = e.id
WHERE amount > 0
GROUP BY u.payer;	
-- 2.4: Популярные эпические предметы:
--общее количество внутриигровых продаж в абсолютном и относительном значении
SELECT distinct i.game_items,
       COUNT(e.transaction_id)as abs_transaction,---количество внутриигровых продаж в абсолютном значении 
 	   ROUND(COUNT(e.transaction_id)::numeric/(select COUNT(transaction_id) from fantasy.events where amount > 0),5)as rel_transaction,---количество внутриигровых продаж в относительном значении
 	   ROUND(COUNT(distinct e.id)::numeric/(select COUNT(distinct id) from fantasy.users),5) as part_users---доля игроков, которые хотя бы раз покупали этот предмет
 	   from fantasy.items as i
 	   left join fantasy.events as e on i.item_code = e.item_code
	   left join fantasy.users as u on e.id = u.id
	   where amount!=0
group by i.game_items
order by part_users desc;
-- Часть 2. Решение ad hoc-задач
-- Задача 1. Зависимость активности игроков от расы персонажа:
 with total_users as(
select r.race,
	   COUNT(distinct e.id) as cnt_pay_users,--Количество игроков с ненулевыми покупками
	   COUNT(e.amount) as cnt_pay_not_null,--количество ненулевых покупок
 	   SUM(e.amount) as sum_total_amount ---общая сумма покупок
 	from fantasy.users as u 
left join fantasy.race as r on u.race_id = r.race_id
left join fantasy.events as e on u.id = e.id
where e.amount >0
group by r.race
),
race_stat AS(
select  r.race,
COUNT(distinct u.id) as cnt_reg_users,--общее количество зарегистрированных игроков для каждой расы)
SUM(e.amount)as sum_amount--сумма покупок по расам
from fantasy.users as u 
left join fantasy.race as r on r.race_id = u.race_id--общее количество зарегистрированных игроков для каждой расы)
left join fantasy.events as e on u.id = e.id
group by r.race
),
pay_buy_user_stat AS(
select  r.race,
COUNT(distinct e.id)as cnt_pay_buy_users---количество платящих игроков с покупками
from fantasy.users as u 
left join fantasy.race as r on r.race_id = u.race_id
left join fantasy.events as e on u.id = e.id
where amount>0 and payer=1
group by r.race
)
select tu.race,
      rs.cnt_reg_users,
      tu.cnt_pay_users,
      ROUND(tu.cnt_pay_users::NUMERIC/rs.cnt_reg_users,5)as part_pay_users,---доля игроков с покупками от всех игроков
      ROUND(pbs.cnt_pay_buy_users::NUMERIC/tu.cnt_pay_users,3) as part_pay_buy_users,--Доля платящих от игроков с покупками
      ROUND(cnt_pay_not_null::NUMERIC/tu.cnt_pay_users,3) as avg_nn_pays,--Среднее количество покупок на игрока
      ROUND(rs.sum_amount::NUMERIC/cnt_pay_not_null,3) as avg_amount_one,--Cредняя стоимость одной покупки  
      ROUND(tu.sum_total_amount::numeric/tu.cnt_pay_users,3) as avg_amount_one_us---Средняя общая стоимость покупок на игрока
      from total_users as tu
      left join race_stat as rs on tu.race = rs.race
      left join pay_buy_user_stat as pbs on tu.race = pbs.race;
    
    

-- Задача 2: Частота покупок
-- Напишите ваш запрос здесь
