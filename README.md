# postgreSQL_course
Документация https://www.postgresql.org/docs/current

1. Скачивается дистрибутив PostgreSQL с официального сайта.
2. Настроить в файлах postgresql.conf и pg_hba.conf listener_addresses «*» и 0.0.0.0/0
3. Убрать firewall: sudo uff allow from any to any.
4. Конфигурационный файл postgresql.conf

*max_parralel_workers_per_gather parameter* - навесить на пользователя и БДБ чтобы ограничить его воздействие на инфузорным для других пользователей

*alter system set enable_seqscan = ‘off’* (добавляет настройку в postgresql.auto.conf)  explain (analyze, settings) - показывает нестандартные параметры

explain analyze - показывает все настройки, + select - показывает параметры и производительность

analyze - показывает все настройки

```
Select * from pg_stats  размеры таблиц и другая инфа  
Select * from pg_database; - список ДБ
```

pg_wal - журнал транзакций - очень полезный, но один на все БД. 

В проде имеет смысл иметь одну БД на одном инстанс сервере, чтобы логи писались без закодирования (файл с логами искать data - current_logfile) postgresql.conf

*lc_messages=en_US.UTF_8* настройка уровня логирования 

*log_statement = ‘none’* - по умолчанию

Все БД создаются потопу БД template1, поэтому если необходимо задать какие-то шаблонные таблицы, то их можно занести в БД template1.

Можно создать свою шаблонную БД, но тогда при создании ее надо специально указать. template0 используется для восстановления дефолтной версии template1

pg_stat_user_tables - статистика об индексах, обновлений по всем таблицам

Event Triggers для аудита

Extensions можно добавить интерпретатор для написание запросов на другом языке, но он должен быть установлен на сервере

Domain = constraint в самой БД, использовать как тип данных (pg_admin)

Функции возвращают значения, а процедуры вызываются call

Materialized Views закэшированное вычисление Данных по расписанию


Селекты
```
select concat(first_name, null, last_name) from employees;
or
select first_name || coalesce(middle name as a test, ‘ ’ ) || last_name) from employees;
```
```
like  (~~)- ‘a%’ - case sensitive , любое количество букв % like - ‘_a_’ - один символ _  ilike (~~*) - case insensitive 

similar to ‘%(b|d)%’ - | b or d  Поддержка регулярок
~ regex  ~* regex
```

Копирование таблицы
```
Select * into new_table from old_table; 
```
Копирование структуры таблицы
```
Select * into new_table from old_table limit 0;
```

Полное копирование всего с constraints, indexes и все остальное
```
Create table new_table (like old_table including all);
```

**CLUSTER**

Кластерные индексы - упорядочивание по одному  или нескольким столбцам, занимает дополнительное пространство вне таблицы.
```
CLUSTER table_name USING index_name;
```
Поэтому является не особо рациональным в использовании.

Индексы

Access method: gin - json, xml, gist - координаты, btree - default, hash - длинные строки

Fill factor - перебалансировка при добавлении новых элементов
Если селект с like, то нужно создавать индекс с Operator class - test_pattern_ops
```
Select * from pg_stat_user_indexes; - использование индексов
 
Select * from pg_stat_statements; - интенсивность используемых запросов
```
В postgresql.conf shared_preload_libraries = ‘pg_stat_statements’ 

Добавить extensions - добавить pg_stat_statements на уровне базы


Планы

PREPARE prepare_name (int) AS 	Select * from employees u where u.employee.id=$1;

Execute prepare_name(107);


Группировки

Rollup - показывает группировку лесенкой
```
Select department_id, job_id, count(*) from employees group by rollup(department_id, job_id) order by department_id, job_id;
```
Cube группирует 2 значения с одной и потом с другой стороны
```
Select department_id, job_id, count(*) from employees group by cube(department_id, job_id) order by department_id, job_id;
```
Grouping sets группирует все значения по отдельности
```
Select department_id, job_id, count(*) from employees group by grouping sets(department_id, job_id) order by department_id, job_id;
```
Grouping
```
Select department_id, job_id, count(*), grouping(department_id) from employees group by rollup(department_id, job_id) order by department_id, job_id;
```

Fetch first 10 rows only; - вернуть первые 10 значений
Offset 5 Rows - пропустить 5 рядов


JOIN

Natural join - совпадающие столбцы  select from 2 tables with join between

Подзапросы

Тратят очень много времени, непроизводительны.
```
explain analyze
select employee_id, first_name, last_name from employees 
where employee_id IN (Select distinct manager_id from employees);

explain analyze
select distinct e1.employee_id, e1.first_name, e1.last_name from employees e1 
join employees e2 on e1.employee_id  = e2.manager_id;

explain analyze
select employee_id, first_name, last_name from employees 
where employee_id = any(Select distinct manager_id from employees);
```

Построение иерархии 

Postgres recursive cte

Автоматическое sequence - использования типа данных bigserial, serial, smallserial.

**Типы данных**

String
Text производительней чем varying character (int) с заданной длиной

Real и double precision не советуют использовать из-за возможной потери данных

Large Objects (lob)
Использовать надо bytea с параметром в external storage

Выделить общее в двух таблицах - intercept
Выделить различное - except
Объединить - union all (union == distinct)

Есть возможность вернуть значение insert, используя returning.

Чтобы перекопировать данные таблицы в файл csv: COPY (select * from employees) TO ‘D:\folder\fail.csv’ DELIMETER ‘,’ CSV HEADER;

Чтобы перекопировать наоборот из файла в таблицу:
```
Select * into emp_imp from employees limit 0
```
Copy emp_imp from  ‘D:\folder\fail.csv’ DELIMETER ‘,’ CSV HEADER;

```
PLPGSQL
do
$$
begin
	raise info 'Hello world';	
	
end;
$$
```
Raise + DEBUG/INFO/LOG/NOTICE/ERROR

% - вывод параметра 

Функции и процедурки проверяются во время выполнения, а не компиляции.

 Создание функции
 ```
create or replace function f1()
returns void
language plpgsql
as 
$$
begin
	raise info 'Hello world %', 'Master';
end;
$$

create or replace function f1()
returns text
language plpgsql
as 
$$
begin
	raise info 'Hello world %', 'Master';
return 'hw';
end;
$$

create or replace function f1(param1 text)
returns text
language plpgsql
as 
$$
declare 
v bigint = 10;
begin
	raise info '%', v;
return 'hw';
end;
$$

create or replace function f1(param1 bigint)
returns bigint
language plpgsql
as 
$$
begin
return param1 * 2;
end;
$$

Select f1 (‘q’);
```
Если брать значения из существующей таблицы
```
create or replace function get_names_by_id(emp_id employees.employee_id%type)
returns text
language plpgsql
as 
$$
declare 
f_name employees.first_name%type;
l_name employees.last_name%type;
v_result text;
begin
	select first_name, last_name into f_name, l_name from employees
	where employee_id = emp_id;
	v_result := concat(f_name, ' ', l_name);
return v_result;
end;
$$

select get_names_by_id(107);
```

Использовать row type, чтобы не прописывать все поля
```
create or replace function get_names_by_id(emp_id employees.employee_id%type)
returns text
language plpgsql
as 
$$
declare 
emp employees%rowtype;
v_result text;
begin
	select * into emp from employees
	where employee_id = emp_id;
	v_result := concat(emp.first_name, ' ', emp.last_name);
return v_result;
end;
$$

select get_names_by_id(107);
```
А лучше ‘emp record’ - универсальный тип данных

Информация о функции:
```
select * from information_schema."routines" r 
where routine_type = 'FUNCTION'
and routine_name = 'f1';
```
Создание процедур
```
create or replace procedure p1(param1 text)
language plpgsql
as 
$$
begin
	raise info 'Hello world %', param1;
end;
$$

Call p1('Master');
```
Внутренние блоки
Перезатирание переменной
```
do
$$
declare
var1 int := 100;
begin
	declare
	var1 text := ‘test’;
	begin
	raise info ‘%’, var1;	
	end;
end;
$$

Info: test
```
Нужно объявлять блок
```
do
$$
<<first_block>>
declare
var1 int := 100;
begin
	declare
	var1 text := ‘test’;
	begin
	raise info ‘%’, first_block.var1;	
	end;
end first_block;
$$

Info: 100
```
Константа

constant_name constant data_type := expression;

Следить за вводом типа данных
```
create or replace function score_decode(score smallint)
returns void
language plpgsql
as 
$$
begin
	raise info '%', score;
end;
$$

select score_decode(5::smallint);
```
Поэтому лучше сразу писать int, чтобы не вводить явную конвертацию.

**Условия**

if then if then else
if then elsif then
end if

```
create or replace function score_decode1(score int)
returns text
language plpgsql
as 
$$
declare 
decode_result text;
begin
	if score = 5 then
	decode_result := 'Отлично';
	elsif score = 4 then
	decode_result := 'Хорошо';
	end if;
	return decode_result;
end;
$$


select score_decode1(5);
```
Если больше 2 значений вместо if лучше использовать
case when then 
end case 
```
create or replace function score_decode1(score int)
returns text
language plpgsql
as 
$$
declare 
decode_result text;
begin
	case score 
	when 5 then
	decode_result := 'Отлично';
	when 4 then
	decode_result := 'Хорошо';
	end case;
	return decode_result;
end;
$$

select score_decode1(5);

create or replace function salary_correct(emp_id employees.employee_id%type)
returns text
language plpgsql
as 
$$
declare 
emp_dep text;
emp_salary float;
decode_result text;
begin
	select d.department_name into emp_dep 
	from employees e join departments d 
on e.department_id = d.department_id 
where e.employee_id = emp_id;
	if  emp_dep = 'IT' then
	emp_salary = 1.5;
decode_result = 'increase';
	else
	emp_salary = 0.9;
decode_result = 'decrease';
	end if;
	update employees set salary = salary * emp_salary;
	return decode_result;
end;
$$

select * from departments;

select salary, d.department_name  from employees e join departments d 
on e.department_id = d.department_id 
where employee_id = 100;
select salary_correct(100);
```

**Циклы**

Для вызова функции в loop нужно использовать perform. Но он не отлавливает результат 
работы функции перебрать в цикле всех сотрудников на каждого напустить функцию, созданную на предыдущей лаб. (которая изменяет зарплату
-в зависимости от отдела)
```
do
$$
declare 
rec employees.employee_id%type;
begin
	for rec IN (select employee_id from employees e)
	loop
	perform salary_correct(rec);
	end loop;
end;
$$
```
Для отладки функций, Процедурой в pg_admin
В файле postgresql.conf задать shared_preload_libraries 
plugin_debugger.so - linux
plugin_debugger.dll - windows

Перезагрузить
Добавить extension - pldbgapi

Дальше идти в фунцию и вызвать Debugging - debug правой кнопкой мыши

Обработка ошибок

Выделяется в собственный блок begin - end
```
exception when text_of_error then
raise error ‘your text’;

exception when others then (все ошибки)
GET stacked diagnostics err_context = PG_EXCEPTION_CONTEXT;
raise info ‘Error name:%’, SQLERRM;
raise info ‘Error State: %’, SQLSTATE;
raise info ‘Error context:%’, err_context;
return null;
```
Cursor для перебора данных
```
create or replace function enumerate()
returns text
language plpgsql
as 
$$
declare 
rec record;
cur_emp cursor for select first_name, last_name 
from employees;
	begin
		open cur_emp;
			loop
				fetch cur_emp into rec;
				exit when not found;
				raise info '%', rec.first_name || ' ' || rec.last_name;
			end loop;
			close cur_emp;
	return 'Done';
end;
$$

select enumerate();
```
Советуют не использовать курсор, а отдать предпочтение for.

Создание таблиц
```
create or replace function get_table(
sal employees.salary%type)
	returns table (		
		f_name employees.first_name%type,
		l_name employees.last_name%type
		)
language plpgsql
as 
$$
	begin
	return query
	select first_name, last_name
	from employees
	where salary >= sal;
end;
$$

select * from get_table(10000);
```
Запросы в функциях

По умолчанию выполняются в режиме volatile, но можно установить immutable, что в 10 раз работает быстрее.
```
create or replace function avg_salary()
	returns decimal(10,2)
language plpgsql
immutable
as 
$$
declare
res decimal(10,2);
	begin
		select avg(salary) into res from employees;
	return  res;
end;
$$

explain analyze
select * from avg_salary();
```
Security invoker - по умолчанию привилегия у того, кто вызвал

security definer - привилегия у того, кто владелец

leakproof - отсутствие возможных побочных эффектов.

PLPGSQL можно совмещать с другими языками: python, java, go и др.

Нужно подключить себе движок языка и добавить extension

В create function прописываем language соответствующий и после $$ используем уже другой язык.

**Unit тестирование**

Можно осуществлять с помощью платформы pgTAP.

Чтобы запускать job нужно использовать pgAgent (нужно установить дополнительно и появится в pgAdmin).
```
create table job_test (c1 serial, c2 timestamp);
insert into job_test (c2) values(current_timestamp);
```
И далее в pgAgent Job  создать job. Заполнить general (db, SQL, Local), steps (enabled, code - insert), 
schedules.

На job выбрать run now для запуска.

Postgres->Catalog->Table->

pga_jobsteplog - посмотреть результаты работы или ошибки

pga_job - информаиця о job

pga_jobstep - определение step

pga_schedule - расписание job

pga_joblog и pga_jobsteplog нужно периодически вычищать, поэтому на это нужно тоже создать job

**ПРОИЗВОДИТЕЛЬНОСТЬ**

Должна быть не одна БД, а несколько с перераспределением данных.

*oltp dw olap staging*

oltp - для оперативной работы, не должны содержать архивные данные (недели 2).
Остальное скидывается ночными пакетами в dw (data warehouse)
olap - для поддержки аналитики

Нужно мониторить

1. Процессор
2. Память
3. Диск
4. Сеть
5. Процессор 

https://pgtune.leopard.in.ua - помогает подкорректировать дефолтные настройки postgresgl.conf 

DB Type ( oltp, dw olap - разные настройки), Total memory 32 GB, CPU 16, Connections 300

max_worker_processes = 16 (по одному на ядро) - процессы posgresql.exe
max_parallel_workers_per_gather = 8 или 4 (выполнение запросов  - select параллельно)

Перед выполнением тяжелого запроса можно сделать: 

set max_parallel_workers_per_gather = 0 (будет работать на своем ядре и не мешать другим)

max_parallel_workers =  16 (для остальных операций)
max_parallel_maintenance_workers = 4 (vacuuming, analyze)

pg_stat_statements - чтобы посмотреть на какие запросы ушло больше всего ресурсов
```
Select * from pg_stat_statements order by total_exec_time desc

Select pg_stat_reset()  или Select pg_stat_statements_reset() - очистить таблицы
```
Память
```
Select * from pg_stat_database; 
```
bkls_read - из диска, blks_hit - из кэша буферов
blks_hit  к bkls_read  + blks_hit не менее 90%

shared_buffers = RAM/4 GB (от total memory зависит, раза в 4 меньше) но корректировать по рабочей ситуации

work_mem = 1764kB(сортировка и группировка)

effective_cash_size = 24GB(ограничение для памяти на диске)

maintenance_work_mem = 2GB

checkpoint_completion_target = 0.9 (событие, когда переставляется метка по переносу данных в БД)

wal_buffers = 14MB(количество памяти под буфер wal, зависит от shared_buffers)

default_statistics_target = 500 (количество записей)

random_page_cost = 4 или 1.1 (ед. стоимости оптимизатора) зависит от data storage: hd or ssd effective_io_concurency = 2
```
Select * from pg_statio_user_tables; 
```
blks_hit  к bkls_read  + blks_hit не менее 90%
```
Select * from pg_buffercache; (надо extension добавить)
```
Показывает, какой запрос тянет не из оперативной, а дисковой памяти:
```
Explain (analyze, buffers)
Select * from employees; 
```
SHOW ALL  - показывает все настройки

Диск

iostat -xmt 

checkpoint_completion_target = 0.9 - снижение нагрузки на диск

Сеть

Linux - ip a
tc -s -d qdisc ls dev ens33

Windows
Сетевой адаптер - длина очереди вывода должна быть 0

Мониторинг

PgAdmin dashboard - аналитика по подключениям и запросам
Grafana dashboard Postgres

Статистика
```
Select * from pg_stat_user_functions; - статистика вызова функций
```
В conf поменять на track_functions = all 

pg_stat_user_indexes - активно использующиеся индексы мониторить

pg_stat_user_tables - статистика по запросам, апдейтам

pg_stat_archiver - архивирование файлов

pg_stat_database - статистика по БД, numbackend - количество активных подключений в БД,наличие взаимных блокировок

pgbench - запуск тестов производительности
pgbench -U postgres -p 5432 -i -s 10 database_name
pgbench -U postgres -p 5432 -c -j 2  -t 500 -h ip(number). database_name

**Оптимизация**
1. Connections Проверить драйвер —jdbc или —npgsql (не должно быть —odbc)  Установить pipeline mode true - асинхронные запросы (с 14 версии) Connection pooling - pgBouncer (поскольку установка подключения очень дорогостоящая)
2. Индексы  pg_stat_user_indexes, pg_stat_user_tables
3. Секционирование/шардинг
4. Дефрагментация/vacuum/analyze/reindex
5. Настройки (pgtune)
6. Архитектура (oltp, dw, staging/reporting, olap)
7. Оптимизация запросов 

Sharding - плавное увеличение дополнительной мощности

Partition -  секционирование данных
```
CREATE TABLE temperature (
  id BIGSERIAL NOT NULL,
  city_id INT NOT NULL,
  timestamp TIMESTAMP NOT NULL,
  temp DECIMAL(5,2) NOT NULL
) PARTITION BY RANGE (timestamp);

CREATE TABLE temperature_201901 PARTITION OF temperature 
FOR VALUES FROM ('2019-01-01') TO ('2019-02-01');

CREATE TABLE temperature_201902 PARTITION OF temperature 
FOR VALUES FROM ('2019-02-01') TO ('2019-03-01');

CREATE TABLE temperature_201903 PARTITION OF temperature FOR VALUES FROM ('2019-03-01') 
TO ('2019-04-01');

insert into temperature
(city_id, "timestamp", temp)
values
(1, to_timestamp('2019-01-10', 'YYYY-MM-DD'), -62);

explain analyze
select * from temperature 
where "timestamp" <= to_timestamp('2019-01-10', 'YYYY-MM-DD');
```
Если не хватает мощностей на своем сервере

Создать новую БД на дополнтельном сервере
```
CREATE TABLE temperature_201905 (
  id BIGSERIAL NOT NULL,
  city_id INT NOT NULL,
  timestamp TIMESTAMP NOT NULL,
  temp DECIMAL(5,2) NOT NULL,
c1 biging
) 
```
Extension postgres_fdw
Foreign Data Wrappers добавляем foreign servers
Options - host и dbname


Вернуться в свой сервер БД
```
CREATE FOREIGN TABLE temperature_201905 PARTITION OF temperature FOR VALUES FROM
 ('2019-05-01') TO ('2019-06-01') SERVER server_name;
```
*Проблемы sharding:*
- Не реализована поддержка распределенных транзакций;
- Нет параллельного выполнения foreign scan;
- Global snapshot manager - есть вероятность не увидеть незафиксированные данные,
- Shard management;
- Dml operations не поддерживаются, только инсерты


Vacuum удаляет мертвые данные

Принудительный vacuum - vacuum full table_name; (без пользователей)

Analyze пересчитывает статистику
```
Explain analyze
Select * from table_name;
```
Extension pgstattuple - для первичных ключей
```
Select * from pgstatindex(‘index_name’); - статистика первичного ключа
```
reindex table table_name; или reindex index index_name; - профилактическое средство от ошибок в том числе

Оптимизация настроек
```
Explain analyze
Select * from employees;
```
Set enable_seqscan = ‘off’ для сеанса - принудительное использование индекса

Genetic query optimizer

Пробует разные планы выполнения запросов (перебор табличек для join)
geqo = ‘on’ - можно выключать, если нет большого количества join

Мастер создания диаграмм можно использовать для нахождения связей в таблицах

Для написания множества join также можно использовать query design :
Выбрать таблицы, выбрать поля.

Система аутентификации и предоставление разрешений
Настраивается с помощью pg_hba.conf. Можно подключить к ldap (AD).
pg_ident прописываются учетки
```
Select * from pg_authid;
```
Настройка проверки пароля - extension passwordcheck

Create Login/Group Roles
Пользователь от группы отличается одним параметром can login
Полные права - superuser
membership - добавить в группу пользователя

Чтобы выдать права к конкретной таблице, нужно зайти в properties таблицы - security.
Пользователи на уровне сервера.

Grant Wizard (вызывается на уровне БД) - используется, чтобы массово выдавать права 
пользователю на все существующие объекты бд (отобрать права не может, 
поэтому лучше давать права группе, а не роли, чтобы выгнав из группы, 
отнять права у пользователя) object selection  выбрать все
Privilege selection -выбрать пользователя и привилегию
Сформируется скрипт
```
CREATE ROLE report_users WITH
NOLOGIN
NOSUPERUSER
NOCREATEDB
NOCREATEROLE
INHERIT
NOREPLICATION
CONNECTION LIMIT -1
PASSWORD 'xxxxxx';

CREATE ROLE ivanov WITH
LOGIN
NOSUPERUSER
NOCREATEDB
NOCREATEROLE
INHERIT
NOREPLICATION
CONNECTION LIMIT -1
PASSWORD 'xxxxxx';

GRANT report_users TO ivanov;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO report_users;
```
Чтобы выдать права на новые объекты в БД, нужно установить default privileges 
на уровне БД или схемы

В 14 версии можно добавить в группу pg_read_all_data, pg_write_all_data

*Backup и Restore*

backup - резервная копия
archiving - перенос данных в архив restore - восстановить из backup
recovery - восстановление работоспособности
failure - сбой в данных
disaster - упал весь сервер

**Способы:**
- File System Level Backup: сервер остановили, перекопировали папку data и запустили сервер (если с 15.1 на 15.2 - минор, для мажора не подойдёт, не знает про wal файлы)
- Dump: для мажора выгружать в dump (выгрузка информации в текстовый файл) использовать утилиту pg_dump (выгрузка данных) и pg_dumpall (выгрузка объектов)

pg_dump -f d:\psb\dump.txt -v —column-inserts -d hr -h localhost -p 5432 -U postgres для каждодневного резервного копирования очень долго. Использовать, чтобы воспроизвести работу продовской бд (нельзя накатить wal)

- Continuous archiving (base packup): на открытой базе без отключения пользователей, делает метку о резервном копировании в wal, можно напустить применение файлов wal.

Создать папку на C:\pg01\archive_wal и предоставить всем все права
В postgresql.conf: archive_mode = on
archive_command = ‘copy «%p» «c:\\pg01\\archive_wal\\%f»’ archive_timeout = 60

Select * from pg_stat_archiver; - проверить запись wal

Создать папку на C:\pg01\ base_backup и предоставить всем все права

pg_basebackup -D C:\pg01\ base_backup -h host -p port -U postgres -P -v

Select * from pg_database;

*Для восстановления*
https://www.postgresql.org/docs/15/continuous-archiving.html#BACKUP-PITR-RECOVERY
1. Остановить Postgres
2. Идем в data, все копируем в data_old.
3. Взять файлы из ночного backup и копировать в папку data
4. Удаляем из папки pg_wal.
5. Скопировать файлы pg_wal из data_old
6. В postgresql.conf restore_command обратную archive_command (recovery блок  подкорректировать), recovery_target_time = ’2022-11-18 12:49:52.000 +03:00’ (время, когда все было правильно). Создать файл recovery.signal. Рестарт posgres.
7. Проверить, что все восстановилось (проверить логи)
8. Select pg_wal_restore_resume(); 
9. В postgresql.conf убираем recovery_target_time.   
Json, jsonb
Добавить и настроить postgrest
https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-json/
```
create table orders(
	id serial not null primary key,
	info jsonb not null
);

INSERT INTO orders (info)
VALUES('{ "customer": "John Doe", "items": {"product": "Beer","qty": 6}}');

INSERT INTO orders (info)
VALUES('{ "customer": "Lily Bush", "items": {"product": "Diaper","qty": 24}}'),
      ('{ "customer": "Josh William", "items": {"product": "Toy Car","qty": 1}}'),
      ('{ "customer": "Mary Clark", "items": {"product": "Toy Train","qty": 2}}');

    
SELECT info -> 'customer' AS customer
FROM orders;
```
Добавляем index типа gin для jsonb
```
SELECT info ->> 'customer' AS customer
FROM orders
WHERE info -> 'items' ->> 'product' = 'Diaper';

update orders 
set info = jsonb_set(info, '{customer}', '"Mr Hyde"', false)
where info ->> 'customer' = 'Mary Clark';
```
*Миграция на PostgreSQL*

Можно использовать бесплатные утилиты pgloader  Проблема с identity, мигрирует таблицы и индексы только, varchar -> text
Для MS SQL
/etc/feetds/freetds.conf
tds version = 8.0
client charset = UTF-8

Можно настроить файл для маппинга 

pgloader mssql://sa:password@host/db pgsql://postgres:password@host/db

Ispirer toolkit - платный
Умеет переносить функции и программный код

Full convert - платный
Очень быстрый перенос данных

Ora2pg

Проблемы:
Не переносит секционирование, заглавные буквы редактируются,
Sequence надо задавать, комментарии, роли и разрешения не переносятся

Отказоустойчивый кластер
High availability
Built-in streaming replication

Можно использовать postgres в связке с patroni etcd haproxy

Failover cluster
https://www.vultr.com/docs/set-up-highly-available-postgresql-replication-cluster-on-ubuntu-20-04-server/

Для создании 2-ой ноды
Поднять на 2 сервере postgres

На 1 сервере в postgresql.conf прописать:
wal_level = logical
wal_log_hints = on

 В pg_hba.conf в replication разрешить всем или определенному 0.0.0.0/0

Перезапустить на 1 сервере

На 2 сервере остановить Postgres и очистить каталог data

pg_basebackup -h from_host -U postgres -X stream -C -S 
replica_1 -v -R -W -D /catalog_data_of_to_host

На 2 сервере править настройки  postgresql.conf и 
pg_hba.conf руками, как на 1

На 2 сервере появится фал standby.signal

Выдать права учётке Postgres на data

Запускаем Postgres на 2 сервере (он будет read-only)

Диагностика 
Select * from pg_stat_wal_receiver; 

Select pg_promote();  - чтобы перевести на полноценный режим 2 сервер

Автоматического переключения тут нет.

