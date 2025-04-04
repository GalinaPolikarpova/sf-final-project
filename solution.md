-- ЗАДАНИЕ 1
-- Берем только 2022 год в анализ, т.к. для 2021 года данные не везде являются корректными - некоторые даты входов раньше дат регистрации
with a as (
	select 
		u2.id, 
		to_char(u2.date_joined, 'YYYY-MM') as cohort, 
		extract (days from u1.entry_at - u2.date_joined) as diff
	from userentry u1
	join users u2
	on u1.user_id = u2.id
	where to_char(u2.date_joined, 'YYYY-MM') >= '2022-01'
)
select 
	cohort,
	count (distinct case when diff >= 0 then id end)*100.0 / count (distinct case when diff >= 0 then id end) as "0 day",
	round (count (distinct case when diff >= 1 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "1 day",
	round (count (distinct case when diff >= 3 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "3 day",
	round (count (distinct case when diff >= 7 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "7 day",
	round (count (distinct case when diff >= 14 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "14 day",
	round (count (distinct case when diff >= 30 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "30 day",
	round (count (distinct case when diff >= 60 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "60 day",
	round (count (distinct case when diff >= 90 then id end)*100.0 / count (distinct case when diff >= 0 then id end),2) as "90 day"
from a
group by cohort

--Выводы: По результатам анализа rolling retention можно сделать вывод о том, что пользователи активно заходят на платформу первые несколько дней. 
--Основной интерес наблюдается в первые 3 дня. В этой связи кажется целесообразным рассмотреть краткосрочную подписку (возможно, пользователю нужны задачи 
--на определенную тему, либо он готовится к предстоящему собеседованию, либо у него есть только выходные, чтобы попрактиковаться) – допустим, на 1 или 3 дня. 
--Далее можно рассмотреть более длительный период – например, 14 и 30 дней. Пользователям, для которых в приоритете постоянное саморазвитие, можно предусмотреть 
--подписку сроком на полгода и год. В последнем случае целесообразно предложить хорошую скидку – скорее всего, многие пользователи перестанут заходить на платформу 
--к концу срока (это видно и из результатов анализа rolling retention), однако акцент на постоянном саморазвитии и скидке обычно отлично срабатывает при продаже.

-- ЗАДАНИЕ 2
with a as (
	select 
		user_id, 
		sum(case when type_id in (1, 23, 24, 25, 26, 27, 28, 30) then value else 0 end) as debit,
		sum(case when type_id not in (1, 23, 24, 25, 26, 27, 28, 30) then value else 0 end) as accrual
	from transaction t 
	group by user_id
)
select 
	avg(accrual) as accrual_avg, 
	avg(debit) as debit_avg, 
	avg(accrual-debit) as balance, 
	percentile_cont(0.5) within group (order by (accrual-debit)) as balance_median
from a

--Выводы: По результатам анализа метрик относительно баланса пользователя можно сделать вывод о том, что коинов начисляется значительно больше, чем списывается 
--(почти в 10 раз: 306,52 / 31,29). У активных пользователей много коинов, поэтому наблюдаются высокие показатели среднего начисления и среднего баланса. 
--В то же время, видим большую разницу между средним и медианным балансом – есть выбросы (активные пользователи, у которых большие начисления коинов).
--Ввиду невысокой заинтересованности пользователей в трате коинов идея о смене модели монетизации на платформе является целесообразной. 
--Если оставлять модель без изменений, то необходимо предпринять действия, направленные на повышение заинтересованности пользователей в использовании коинов. 

--Если рассматривать варианты подписок на разные сроки, то, отталкиваясь от медианного баланса (цена должна быть комфортной для большинства пользователей), 
--можно рассмотреть для краткосрочной подписки (до 3 дней) стоимость в районе 50 коинов в пересчете на рубли, в районе 150 коинов – на 14 дней, 200 коинов – на месяц и т.д. 
--Также можно рассмотреть варианты различных цен для ограниченного и полного доступа к платформе. Кроме того, не будет лишним ознакомиться с предложениями конкурентов, 
--чтобы цена была «в рынке». После установления цен необходимо наблюдать за динамикой активности пользователей платформы, чтобы отследить эффект от нововведений, 
--в зависимости от данного эффекта запланировать пересмотр цен на следующем этапе.

-- ЗАДАНИЕ 3

--Метрика 1: Сколько в среднем пользователь решает задач
with a1 as(  				  
	select 
		user_id, 
		problem_id as cnt
	from coderun
	union						 
	select 
		user_id, 
		problem_id as cnt   
	from codesubmit
),
a2 as (
	select count(*) as cnt     	
	from a1
	group by user_id
)
select round(avg(cnt), 2) as problems_avg
from a2
--Вывод: Пользователь решает в среднем примерно 9.18 задач

--Метрика 2: Сколько в среднем пользователь делает попыток для решения 1 задачи
with b as (
	select 
		user_id, 
		problem_id, 
		count(problem_id) as cnt1
	from codesubmit c
	group by user_id, problem_id
	union
	select 
		user_id, 
		problem_id, 
		count(problem_id) as cnt1
	from coderun c1
	group by user_id, problem_id
)
select round(avg(cnt1),2) as att_problem_avg
from b
--Вывод: Пользователь делает примерно 5.75 попыток для решения 1 задачи 

--Метрика 3: Сколько в среднем пользователь проходит тестов
with c as (
	select 
		user_id, 
		count(distinct test_id) as cnt3
	from teststart t 
	group by user_id
)
select round(avg(cnt3),2) as tests_avg
from c
--Вывод: Пользователь проходит в среднем 1,68 тестов

--Метрика 4: Сколько в среднем пользователь делает попыток для прохождения 1 теста
with d as (
	select 
		user_id, 
		test_id, 
		count(test_id) as cnt4
	from teststart t 
	group by user_id, test_id
)
select round(avg(cnt4),2) as att_tests_avg
from d
--Вывод: Пользователь в среднем делает 1.26 попыток для прохождения 1 теста

--Метрика 5: Какая доля от общего числа пользователей решала хотя бы одну задачу или начинала проходить хотя бы один тест
with e as (
	select distinct user_id
	from codesubmit
	union
	select distinct user_id
	from coderun
	union
	select distinct user_id
	from teststart
)
select round(count(*) *100.0 / (select count(*) from users),2) as active_users_share
from e
--Вывод: Доля пользователей, которые решали хотя бы одну задачу или начинали проходить хотя бы один тест от общего числа пользователей - 63.48%
