# plSQL-YandexProject-3
Самостоятельный проект по итогу окончания курса "Серверное программирование на SQL"

## Ревью
Результат ревью и отзыв по проекту размещен в [Comments.txt]

## Описание проекта. Система управления персоналом «Всё записано».
Вы работаете в растущей IT-компании Dream Big. Специализация компании — разработка программного обеспечения на заказ. Десятки сотрудников каждый день без устали трудятся над самыми разнообразными проектами: от скромных приложений до грандиозных платформ. 

Вы попивали кофе во время перерыва, когда ворвался взъерошенный тимлид Арсений с новой горящей задачей — нужно автоматизировать некоторые процессы в Dream Big для внутренней системы управления персоналом под кодовым названием «Всё записано».

В компании есть база данных PostgreSQL с детализированной информацией о сотрудниках, проектах и логах времени. Сейчас структуре БД и механизмам запросов не хватает гибкости и производительности. Кроме того, политика безопасности проекта гласит: все запросы выполняются строго через функции и процедуры, сырые запросы к таблицам — запрещены.

### Цель проекта
Cоздать ряд хранимых процедур и функций, которые оптимизируют и автоматизируют процессы извлечения, анализа и изменения данных. Арсений предполагает, что это позволит менеджерам по персоналу и бухгалтерии Dream Big быстро получать отчёты, анализировать рабочие часы и корректировать данные в режиме реального времени. Протестировать гипотезу Арсения.

#### Предусловия
Перед началом выполнения заданий необходимо развернуть базу данных из дампа 

#### Задание №1
В Dream Big ежемесячно оценивают производительность сотрудников. В результате бывает, кому-то повышают, а изредка понижают почасовую ставку. 
Напишите хранимую процедуру ```update_employees_rate```, которая обновляет почасовую ставку сотрудников на определённый процент. При понижении ставка не может быть ниже минимальной — 500 рублей в час. Если по расчётам выходит меньше, устанавливают минимальную ставку.

На вход процедура принимает строку в формате ```json```:
```sql
[
    -- uuid сотрудника                                      процент изменения ставки
    {"employee_id": "6bfa5e20-918c-46d0-ab18-54fc61086cba", "rate_change": 10}, 
    -- -- -- 
    {"employee_id": "5a6aed8f-8f53-4931-82f4-66673633f2a8", "rate_change": -5}
]
```

```sql
create or replace procedure update_employees_rate(rate_update json)
language plpgsql
as $$
DECLARE --Объявляем переменные _length, _min_rate, _employee_rate, _rate_changer, _new_rate
    _length integer := json_array_length(rate_update);
    _min_rate integer := 500;
    _employee_rate integer;
    _rate_changer integer;
    _new_rate integer;
BEGIN	
    IF _length <= 0 THEN 
		raise exception 'Запрос не может быть пустым'; --Выводим информационное сообщение (валидация) если rate_update <= 0
    END IF;

    FOR _i IN 0.._length-1 --Запускаем цикл и задаём условие
    LOOP
    	select into _employee_rate (select rate from employees where id = (select rate_update->_i->>'employee_id')::uuid);
    	select into _rate_changer (select rate_update->_i->>'rate_change');
    	select into _new_rate (select _employee_rate + (_employee_rate * _rate_changer / 100));

        IF _new_rate < _min_rate THEN
        	update employees set rate = _min_rate where id = (select rate_update->_i->>'employee_id')::uuid;
        ELSE
            update employees
            set rate = _employee_rate + (_employee_rate * _rate_changer / 100)
            where id = (select rate_update->_i->>'employee_id')::uuid;
        END IF;
     END LOOP;
END;
$$;
	
--Тест задания №1
CALL update_employees_rate(
    '[
        {"employee_id": "80718590-e2bf-492b-8c83-6f8c11d007b1", "rate_change": 10}, 
        {"employee_id": "80718590-e2bf-492b-8c83-6f8c11d007b1", "rate_change": -5}
    ]'::json
); 

--Проверка задания №1
select * from employees where id = '80718590-e2bf-492b-8c83-6f8c11d007b1';
```

#### Задание №2
С ростом доходов компании и учётом ежегодной инфляции Dream Big индексирует зарплату всем сотрудникам.

Напишите хранимую процедуру ```indexing_salary```, которая повышает зарплаты всех сотрудников на определённый процент. 

Процедура принимает один целочисленный параметр — процент индексации ```p```. Сотрудникам, которые получают зарплату по ставке ниже средней относительно всех сотрудников до индексации, начисляют дополнительные 2% (```p + 2```). Ставка остальных сотрудников увеличивается на ```p```%.

Зарплата хранится в БД в типе данных ```integer```, поэтому если в результате повышения зарплаты образуется дробное число, его нужно округлить до целого.

```sql
create or replace procedure indexing_salary(p integer)
language plpgsql
as $$
DECLARE --Объявляем переменные _index, _average_rate, _value
    _index integer := 2;
    _average_rate integer;
    _value record;
BEGIN
    IF p <= 0 THEN
		raise exception 'Ставка не может быть меньше или равна нулю'; --Выводим информационное сообщение (валидация) если параметр p <= 0
    END IF;

    select into _average_rate (select avg(rate) from employees);

    FOR _value IN (select * from employees) -- Запускаем цикл и задаём условие
	LOOP
    	IF _value.rate < _average_rate THEN
        	update employees set rate = (_value.rate + (_value.rate * (p + _index) / 100)) where id = _value.id;
    	ELSE
        	update employees set rate = (_value.rate + (_value.rate * p / 100)) where id = _value.id;
    	END IF;
    END LOOP;
END;
$$;

--Тест задания №2
CALL indexing_salary(5);

--Проверка задания №2
select * from employees order by rate;
```

#### Задание №3
Завершая проект, нужно сделать два действия в системе учёта:
1. Изменить значение поля ```is_active``` в записи проекта на ```false``` — чтобы рабочее время по этому проекту больше не учитывалось.
2. Посчитать бонус, если он есть — то есть распределить неизрасходованное время между всеми членами команды проекта. Неизрасходованное время — это разница между временем, которое выделили на проект (```estimated_time```), и фактически потраченным. Если поле ```estimated_time``` не задано, бонусные часы не распределятся. Если отработанных часов нет — расчитывать бонус не нужно.

Разберёмся с бонусом. 

Если в момент закрытия проекта ```estimated_time```:
1. не ```NULL```,
2. больше суммы всех отработанных над проектом часов,

всем членам команды проекта начисляют бонусные часы.

Размер бонуса считают так: 75% от сэкономленных часов делят на количество участников проекта, но не более 16 бонусных часов на сотрудника. Дробные значения округляют в меньшую сторону (например, 3.7 часа округляют до 3). Рабочие часы заносят в логи с текущей датой. 

Например, если на проект запланировали 100 часов, а сделали его за 30 — 3/4 от сэкономленных 70 часов распределят бонусом между участниками проекта.

Создайте пользовательскую процедуру завершения проекта ```close_project```. Если проект уже закрыт, процедура должна вернуть ошибку без начисления бонусных часов.
```sql
create or replace procedure close_project(_id uuid)
language plpgsql
as $$
DECLARE --Объявляем переменные _max_share_bonus, _max_bonus_hours, _project_actual_time, _project_estimate_time, _participants, _possible_bonus_hours, _actual_bonus_hours, _value 
    _max_share_bonus integer := 0.75;
    _max_bonus_hours integer := 16;
    _project_actual_time integer;
    _project_estimate_time integer;
    _participants integer;
    _possible_bonus_hours integer;
    _actual_bonus_hours integer;
    _value record;
BEGIN
    IF (select is_active from projects where id = _id) = false then 
		raise exception 'Проект уже закрыт'; --Выводим информационное сообщение (валидация)
    END IF;

	IF (select exists(select 1 from projects where id = _id)) = false then 
		raise exception 'Проект не найден'; --Выводим информационное сообщение (валидация)
    END IF;

    select into _project_actual_time (select sum(work_hours) from logs where project_id = _id);
    select into _participants (select count(distinct employee_id) from logs where project_id = _id);
    select into _project_estimate_time (select estimated_time from projects where id = _id);

    IF --Задаём условия по заданию (не NULL и больше суммы всех отработанных над проектом часов) и запускаем цикл
        _project_estimate_time > _project_actual_time and _project_estimate_time is not null THEN
        select into _possible_bonus_hours (select div((_project_estimate_time - _project_actual_time), _participants) * _max_share_bonus);

        select into _actual_bonus_hours
            CASE
                WHEN _possible_bonus_hours > _max_bonus_hours THEN _max_bonus_hours
            ELSE _possible_bonus_hours
            END;

        FOR _value IN (
            select
                distinct employee_id,
                SUM(work_hours) OVER (PARTITION BY employee_id, project_id) as total_work_hours
            from logs where project_id = _id)
        LOOP
            insert into logs (employee_id, project_id, work_date, work_hours)
            values (_value.employee_id, _id, current_date, _actual_bonus_hours);
        END LOOP;

        update projects set is_active = false where id = _id;
    END IF;
END;
$$;

--Тест задания №3
CALL close_project('5f14f454-afbf-4f05-8d48-19db9237c8ff'); 

--Проверка задания №3
select * from projects where id = '5f14f454-afbf-4f05-8d48-19db9237c8ff';
```

#### Задание №4
Напишите процедуру ```log_work``` для внесения отработанных сотрудниками часов. Процедура добавляет новые записи о работе сотрудников над проектами.

Процедура принимает id сотрудника, id проекта, дату и отработанные часы и вносит данные в таблицу ```logs```. 

Если проект завершён, добавить логи нельзя — процедура должна вернуть ошибку ```Project closed```. Количество залогированных часов может быть в этом диапазоне: от 1 до 24 включительно — нельзя внести менее 1 часа или больше 24. Если количество часов выходит за эти пределы, необходимо вывести предупреждение о недопустимых данных и остановить выполнение процедуры.

Запись помечается флагом ```required_review```, если:
1. залогированно более 16 часов за один день — Dream Big заботится о здоровье сотрудников;
2. запись внесена будущим числом;
3. запись внесена более ранним числом, чем на неделю назад от текущего дня — например, если сегодня 10.04.2023, все записи старше 3.04.2023 получат флажок.
```sql
create or replace procedure log_work(_employee_id uuid, _project_id uuid, _day date, _work_hours integer)
language plpgsql
as $$
DECLARE -- Объявляем переменные _max_hour_check, _max_interval_check, _required_review
    _max_hour_check integer := 16;
    _max_interval_check interval := '7 days';
    _required_review bool;
BEGIN
    IF (select exists(select 1 from projects where id = _project_id)) = false THEN 
		raise exception 'Проект не найден'; --Выводим информационное сообщение (валидация)
    END IF;

    IF (select exists(select 1 from employees where id = _employee_id)) = false THEN 
		raise exception 'Сотрудник не найден'; --Выводим информационное сообщение (валидация)
    END IF;

    IF (select is_active from projects where id = _project_id) = false THEN 
		raise exception 'Проект уже закрыт'; --Выводим информационное сообщение (валидация)
    END IF;

    IF _work_hours <1 or _work_hours > 24 THEN 
		raise exception 'Рабочие часы должны быть в интервале с 1 до 8';
    END IF;

    select into _required_review
        CASE
            WHEN _work_hours > _max_hour_check
            	or _day > (select current_date)
                or _day < ((select current_date) - _max_interval_check)
            THEN true
            ELSE false
        END;

    insert into logs (employee_id, project_id, work_date, work_hours, required_review)
    values (_employee_id, _project_id, _day, _work_hours, _required_review);
END;
$$;

--Тест задания №4
CALL log_work(
    '6db4f4a3-239b-4085-a3f9-d1736040b38c', -- employee uuid
    '35647af3-2aac-45a0-8d76-94bc250598c2', -- project uuid
    '2023-10-22',                           -- work date
    4                                       -- worked hours
);

--Проверка задания №4
select * from logs where employee_id = '6db4f4a3-239b-4085-a3f9-d1736040b38c'
and project_id = '35647af3-2aac-45a0-8d76-94bc250598c2';
```

#### Задание №5
Чтобы бухгалтерия корректно начисляла зарплату, нужно хранить историю изменения почасовой ставки сотрудников. Создайте отдельную таблицу ```employee_rate_history``` с такими столбцами:
1. ```id``` — id записи,
2. ```employee_id``` — id сотрудника,
3. ```rate``` — почасовая ставка сотрудника,
4. ```from_date``` — дата назначения новой ставки.

Внесите в таблицу текущие данные всех сотрудников. В качестве ```from_date``` используйте дату основания компании: ```'2020-12-26'```.

Напишите триггерную функцию ```save_employee_rate_history``` и триггер ```change_employee_rate```. При добавлении сотрудника в таблицу ```employees``` и изменении ставки сотрудника триггер автоматически вносит запись в таблицу ```employee_rate_history``` из трёх полей: id сотрудника, его ставки и текущей даты.
```sql
--Создаём таблицу employee_rate_history исходя из условия и назначаем ограничения 
create table if not exists employee_rate_history(
    id serial primary key,
    employee_id uuid not null,
    rate integer not null,
    from_date date not null default '2020-12-26',
    constraint fk_employee_id foreign key (employee_id) references employees(id),
    constraint rate_must_be_positive check (rate > 0),
    constraint from_date_must_be_later_than_2020_12_25 check (from_date >= '2020-12-26')
);

--Создаём функцию save_employee_rate_history с типом возвращаемого значения trigger
create or replace function save_employee_rate_history()
returns trigger
language plpgsql
as $$
BEGIN
    insert into employee_rate_history (employee_id, rate, from_date)
    values (NEW.id, NEW.rate, CURRENT_DATE);
    RETURN NEW;
END;
$$;

--Создаём триггер 
create or replace trigger change_employee_rate
after insert or update on employees
for each row
execute function save_employee_rate_history();

--Тест задания №5
update employees set rate = rate * 1.05 
where id = '181b2307-9e97-4cde-8b9e-b2d9d3d79f09';

--Проверка задания №5
select * from employee_rate_history 
where employee_id = '181b2307-9e97-4cde-8b9e-b2d9d3d79f09';
```

#### Задание №6
После завершения каждого проекта Dream Big проводит корпоративную вечеринку, чтобы отпраздновать очередной успех и поощрить сотрудников. Тех, кто посвятил проекту больше всего часов, награждают премией «Айтиголик» — они получают почётные грамоты и ценные подарки от заказчика.

Чтобы вычислить айтиголиков проекта, напишите функцию ```best_project_workers```.

Функция принимает id проекта и возвращает таблицу с именами трёх сотрудников, которые залогировали максимальное количество часов в этом проекте. Результирующая таблица состоит из двух полей: имени сотрудника и количества часов, отработанных на проекте.
```sql
create or replace function best_project_workers(_project_id uuid)
returns table (employee text, work_hours bigint)
language plpgsql
as $$
BEGIN
    IF (select exists(select 1 from projects where id = _project_id)) = false THEN 
		raise exception 'Проект не найден'; --Выводим информационное сообщение (валидация)
    END IF;

    RETURN QUERY --Задаём условие
        select
            distinct el.name as name,
            SUM(l.work_hours) over (partition by l.employee_id, l.project_id) as work_hours
        from logs l
        left join employees el on l.employee_id = el.id
        where project_id = _project_id
        order by work_hours desc
        limit 3;
END;
$$;

--Тест задания №6
SELECT employee, work_hours FROM best_project_workers(
    '4abb5b99-3889-4c20-a575-e65886f266f9' -- Project UUID
);

--Проверка задания №6
select distinct 
	el.name as name,
	SUM(l.work_hours) over (partition by l.employee_id, l.project_id) as work_hours
from logs l
left join employees el on l.employee_id = el.id
where l.project_id = '4abb5b99-3889-4c20-a575-e65886f266f9'
order by work_hours desc
limit 3;
```

#### Задание №7 + не обязательное задание
К вам заглянул утомлённый главный бухгалтер Марк Захарович с лёгкой синевой под глазами и попросил как-то автоматизировать расчёт зарплаты, пока бухгалтерия не испустила дух.

Напишите для бухгалтерии функцию ```calculate_month_salary``` для расчёта зарплаты за месяц.

Функция принимает в качестве параметров даты начала и конца месяца и возвращает результат в виде таблицы с четырьмя полями: ```id``` (сотрудника), ```employee``` (имя сотрудника), ```worked_hours``` и ```salary```.

Процедура суммирует все залогированные часы за определённый месяц и умножает на актуальную почасовую ставку сотрудника. Исключения — записи с флажками ```required_review``` и ```is_paid```.

Если суммарно по всем проектам сотрудник отработал более 160 часов в месяц, все часы свыше 160 оплатят с коэффициентом 1.25.

**Не обязательно:**
Доработайте функцию ```calculate_month_salary```.

Если у сотрудника есть флажки ```required_review```, при выполнении функции появится предупреждение: ```Warning! Employee % hours must be reviewed!```.

На часы, не требующие проверки менеджера, это не повлияет — зарплату за них начислят в обычном режиме.
```sql
create or replace function calculate_month_salary(_month_begin date, _month_end date)
returns table (id uuid, worked_hours bigint, salary numeric(9,2))
language plpgsql
as $$
DECLARE --Объявляем переменные _standart_hours, _over_coef, _value
    _standart_hours integer := 160;
    _over_coef numeric := 1.25;
    _value record;
BEGIN
    IF extract(month from _month_begin) != extract(month from _month_end) THEN 
		raise exception 'Дата начала и окончания должны быть одного месяца'; --Выводим информационное сообщение (валидация)
    END IF;

    IF extract(year from _month_begin) != extract(year from _month_end) THEN 
		raise exception 'Дата начала и окончания должны быть одного года'; --Выводим информационное сообщение (валидация)
    END IF;

    IF _month_begin > _month_end THEN 
		raise exception 'Начало месяца должно быть меньше окончания'; --Выводим информационное сообщение (валидация)
    END IF;

    IF _month_begin != (date_trunc('month', (_month_begin- interval '1 month')) + interval '1 month')::date THEN 
		raise exception 'Дата начала месяца не является фактической'; --Выводим информационное сообщение (валидация)
    END IF;
	
    IF _month_end != (date_trunc('month', _month_end) + interval '1 month - 1 day')::date THEN
		raise exception 'Дата окончания месяца не является фактической'; --Выводим информационное сообщение (валидация)
    END IF;

    FOR _value IN (
    select
        distinct l.employee_id as id,
                 el.name as name,
                 l.required_review as required_review
                 from logs l
                 left join employees el on l.employee_id = el.id
                 where work_date >= _month_begin and work_date <= _month_end and is_paid = false and required_review = true) 
	LOOP --Запускаем цикл с сообщением
        raise notice 'Warning! Employee with id = %, name = % hours must be reviewed!', _value.id, _value.name; --Выводим информационное сообщение (валидация) по заданию №7*
    END LOOP;

    RETURN QUERY
        WITH worked_hours as (
            select
                employee_id, 
				project_id, 
				work_hours
            from logs 
			where work_date >= _month_begin 
				and work_date <= _month_end 
				and required_review = false 
				and is_paid = false
        )
        SELECT
            wh.employee_id as id,
            SUM(wh.work_hours) as worked_hours,
            CASE
                WHEN SUM(wh.work_hours) > _standart_hours
                THEN (_standart_hours * el.rate)::numeric(9,2) +
                     ((SUM(wh.work_hours) - _standart_hours) * el.rate * _over_coef)::numeric(9,2)
                ELSE (SUM(wh.work_hours) * el.rate)::numeric(9,2)
            END as employees_salary
        from worked_hours wh
        left join employees el on wh.employee_id = el.id
        group by wh.employee_id, el.rate
        order by worked_hours desc;
END;
$$;

--Тест и проверка задания №7
SELECT * FROM calculate_month_salary(
    '2023-10-01',  -- start of month
    '2023-10-31'   -- end of month
); 
```

