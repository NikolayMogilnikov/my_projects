1. Найдите количество вопросов, которые набрали больше 300 очков или как минимум 100 раз были добавлены в «Закладки».
```
SELECT COUNT(title)
FROM stackoverflow.posts
WHERE score > 300 OR favorites_count >= 100;
```
2. Сколько в среднем в день задавали вопросов с 1 по 18 ноября 2008 включительно? Результат округлите до целого числа.
```
SELECT ROUND(AVG(cnt)) FROM
(SELECT DATE_TRUNC('day', creation_date)::date, COUNT(post_type_id) as cnt
    FROM stackoverflow.posts
    WHERE creation_date::date BETWEEN '2008-11-1' AND '2008-11-18' AND post_type_id = 1
GROUP BY DATE_TRUNC('day', creation_date)::date) AS A;
```
3. Сколько пользователей получили значки сразу в день регистрации? Выведите количество уникальных пользователей.
```
SELECT COUNT(DISTINCT u.id)
FROM stackoverflow.users AS u
LEFT JOIN stackoverflow.badges AS b ON b.user_id = u.id
WHERE DATE(b.creation_date)  = DATE(u.creation_date);
```
4. Сколько уникальных постов пользователя с именем Joel Coehoorn получили хотя бы один голос?
```
SELECT COUNT (DISTINCT P.id)
FROM stackoverflow.users as u
JOIN stackoverflow.POSTS AS P ON 
u.id=p.user_id
JOIN stackoverflow.votes as v
on p.id = v.post_id
WHERE display_name LIKE '%Joel Coehoorn%';
```
5. Выгрузите все поля таблицы vote_types. Добавьте к таблице поле rank, в которое войдут номера записей в обратном порядке. Таблица должна быть отсортирована по полю id.
```
SELECT *,
  rank()  OVER (ORDER BY id DESC)
FROM stackoverflow.vote_types
ORDER BY id;
```
6. Отберите 10 пользователей, которые поставили больше всего голосов типа Close. Отобразите таблицу из двух полей: идентификатором пользователя и количеством голосов. Отсортируйте данные сначала по убыванию количества голосов, потом по убыванию значения идентификатора пользователя.
```
WITH ctr AS (
SELECT u.id AS user_id,
    COUNT(*) OVER (PARTITION BY u.id) AS cnt
FROM stackoverflow.users as u
JOIN  stackoverflow.votes as v ON u.id = v.user_id
JOIN  stackoverflow.vote_types as vt ON vt.id = v.vote_type_id
WHERE vt.name = 'Close'
)
SELECT user_id, cnt
FROM ctr
GROUP BY user_id, cnt
ORDER BY cnt DESC, user_id DESC
LIMIT 10;
```
7. Отберите 10 пользователей по количеству значков, полученных в период с 15 ноября по 15 декабря 2008 года включительно.
- Отобразите несколько полей:
- идентификатор пользователя;
- число значков;
- место в рейтинге — чем больше значков, тем выше рейтинг.
Пользователям, которые набрали одинаковое количество значков, присвойте одно и то же место в рейтинге.
Отсортируйте записи по количеству значков по убыванию, а затем по возрастанию значения идентификатора пользователя.
```
SELECT user_id, COUNT(id) AS amount, DENSE_RANK() OVER (ORDER BY COUNT(id)desc)
FROM stackoverflow.badges
WHERE creation_date::date BETWEEN  '20081115' AND '20081215'
GROUP BY user_id
ORDER BY amount DESC, user_id
limit 10;
```
8. Сколько в среднем очков получает пост каждого пользователя?
Сформируйте таблицу из следующих полей:
- заголовок поста;
- идентификатор пользователя;
- число очков поста;
- среднее число очков пользователя за пост, округлённое до целого числа.
Не учитывайте посты без заголовка, а также те, что набрали ноль очков.
```
SELECT title,
user_id,
score,
ROUND (AVG(score) OVER (partition by user_id),0)
FROM stackoverflow.posts
WHERE score <> 0 AND title IS NOT NULL
```
9. Отобразите заголовки постов, которые были написаны пользователями, получившими более 1000 значков. Посты без заголовков не должны попасть в список.
```
WITH ctr AS (
SELECT user_id, COUNT(id) AS cnt
FROM stackoverflow.badges
GROUP BY user_id
)
SELECT DISTINCT p.title
FROM ctr as c
JOIN  stackoverflow.posts as p ON c.user_id = p.user_id
WHERE c.cnt > 1000 AND p.title IS NOT NULL AND p.title <> '';
```
10. Напишите запрос, который выгрузит данные о пользователях из США (англ. United States). Разделите пользователей на три группы в зависимости от количества просмотров их профилей:
- пользователям с числом просмотров больше либо равным 350 присвойте группу 1;
- пользователям с числом просмотров меньше 350, но больше либо равно 100 — группу 2;
- пользователям с числом просмотров меньше 100 — группу 3.
Отобразите в итоговой таблице идентификатор пользователя, количество просмотров профиля и группу. Пользователи с нулевым количеством просмотров не должны войти в итоговую таблицу.
```
SELECT id,
       views,
       CASE
       WHEN views >=350 THEN 1
          WHEN views <350 AND views >=100 THEN 2
          WHEN views <100 THEN 3   
       END       
FROM stackoverflow.users 
WHERE location LIKE '%United States%' AND views !=0
GROUP BY id, views
```
11. Дополните предыдущий запрос. Отобразите лидеров каждой группы — пользователей, которые набрали максимальное число просмотров в своей группе. Выведите поля с идентификатором пользователя, группой и количеством просмотров. Отсортируйте таблицу по убыванию просмотров, а затем по возрастанию значения идентификатора.
```
WITH tab_1 as (
SELECT id,
       views,
       CASE
       WHEN views >=350 THEN 1
          WHEN views <350 AND views >=100 THEN 2
          WHEN views <100 THEN 3   
       END as category      
FROM stackoverflow.users 
WHERE location LIKE '%United States%' AND views !=0
GROUP BY id, views
)
SELECT id,
       category,
       views
FROM
  (SELECT *,
          rank() over(PARTITION BY category
                      ORDER BY views DESC) AS ranks
   FROM tab_1) AS tab_2
WHERE ranks = 1
ORDER BY views DESC, id
```
12. Посчитайте ежедневный прирост новых пользователей в ноябре 2008 года. Сформируйте таблицу с полями:
- номер дня;
- число пользователей, зарегистрированных в этот день;
сумму пользователей с накоплением.
```
SELECT day, cnt,
    sum(cnt) over (order by day)
FROM
(SELECT EXTRACT(DAY FROM DATE_TRUNC('day', creation_date)::date) as day, COUNT(id) as cnt
    FROM stackoverflow.users
    WHERE creation_date::date BETWEEN '2008-11-1' AND '2008-11-30'
GROUP BY EXTRACT(DAY FROM DATE_TRUNC('day', creation_date)::date)) as a;
```
13. Для каждого пользователя, который написал хотя бы один пост, найдите интервал между регистрацией и временем создания первого поста. Отобразите:
 - идентификатор пользователя;
 - разницу во времени между регистрацией и первым постом.
```
WITH pre AS (SELECT u.id AS u_id,
       u.creation_date AS u_creation,
       FIRST_VALUE(p.creation_date) OVER (PARTITION BY p.user_id ORDER BY p.creation_date) AS first_post
FROM stackoverflow.users u
JOIN stackoverflow.posts p ON u.id=p.user_id)
 
SELECT DISTINCT u_id,
       first_post - u_creation
FROM pre
```
14. Выведите общую сумму просмотров постов за каждый месяц 2008 года. Если данных за какой-либо месяц в базе нет, такой месяц можно пропустить. Результат отсортируйте по убыванию общего количества просмотров.
```
SELECT DATE_TRUNC('month', creation_date)::date, sum(views_count) as cnt
FROM stackoverflow.posts
GROUP BY DATE_TRUNC('month', creation_date)::date
ORDER BY cnt DESC
```
15. Выведите имена самых активных пользователей, которые в первый месяц после регистрации (включая день регистрации) дали больше 100 ответов. Вопросы, которые задавали пользователи, не учитывайте. Для каждого имени пользователя выведите количество уникальных значений user_id. Отсортируйте результат по полю с именами в лексикографическом порядке.
```
    SELECT u.display_name, COUNT(DISTINCT p.user_id)
FROM stackoverflow.users u
JOIN stackoverflow.posts p ON u.id = p.user_id
JOIN stackoverflow.post_types pt ON p.post_type_id = pt.id
WHERE pt.type = 'Answer'
  AND p.creation_date::date BETWEEN u.creation_date::date AND (u.creation_date + INTERVAL '1 month')::date
GROUP BY u.display_name
HAVING COUNT(DISTINCT p.id) > 100
ORDER BY u.display_name;
```
16. Выведите количество постов за 2008 год по месяцам. Отберите посты от пользователей, которые зарегистрировались в сентябре 2008 года и сделали хотя бы один пост в декабре того же года. Отсортируйте таблицу по значению месяца по убыванию.
```
WITH i as (
select DISTINCT u.id as user_id, 
        p.id as post_id,
        u.creation_date::date as reg_date,
        p.creation_date::date as post_date
from stackoverflow.users u
join stackoverflow.posts p ON p.user_id=u.id
where u.creation_date::date between '2008-09-01' and '2008-09-30'
and p.creation_date::date between '2008-12-01' and '2008-12-31' )

select  DATE_TRUNC('month', p2.creation_date)::date as month,
        COUNT(DISTINCT p2.id) 
from i
JOIN stackoverflow.posts p2 ON i.user_id=p2.user_id
group by month
order by month desc;
```
17. Используя данные о постах, выведите несколько полей:
- идентификатор пользователя, который написал пост;
- дата создания поста;
- количество просмотров у текущего поста;
сумму просмотров постов автора с накоплением.
Данные в таблице должны быть отсортированы по возрастанию идентификаторов пользователей, а данные об одном и том же пользователе — по возрастанию даты создания поста.
```
SELECT user_id, creation_date, views_count,
    SUM(views_count) over (partition by user_id order by creation_date)
FROM stackoverflow.posts
ORDER BY user_id, creation_date
```
18. Сколько в среднем дней в период с 1 по 7 декабря 2008 года включительно пользователи взаимодействовали с платформой? Для каждого пользователя отберите дни, в которые он или она опубликовали хотя бы один пост. Нужно получить одно целое число — не забудьте округлить результат.
```
WITH a AS
(SELECT DISTINCT u.id AS uid,
      COUNT(DISTINCT p.creation_date::DATE) AS cnt 
FROM stackoverflow.users u
JOIN stackoverflow.posts p ON u.id = p.user_id

 WHERE DATE_TRUNC('day', p.creation_date)::DATE BETWEEN '2008-12-01' AND '2008-12-07' GROUP BY u.id )

SELECT ROUND(AVG(a.cnt))
FROM a
```
19. На сколько процентов менялось количество постов ежемесячно с 1 сентября по 31 декабря 2008 года? Отобразите таблицу со следующими полями:
- номер месяца;
- количество постов за месяц;
- процент, который показывает, насколько изменилось количество постов в текущем месяце по сравнению с предыдущим.
Если постов стало меньше, значение процента должно быть отрицательным, если больше — положительным. Округлите значение процента до двух знаков после запятой.
Напомним, что при делении одного целого числа на другое в PostgreSQL в результате получится целое число, округлённое до ближайшего целого вниз. Чтобы этого избежать, переведите делимое в тип numeric.
```
with ctr as (
SELECT EXTRACT(MONTH FROM DATE_TRUNC('day', creation_date)::date) as month, COUNT(id) as cnt
    FROM stackoverflow.posts
    WHERE creation_date::date BETWEEN '2008-09-01' AND '2008-12-31'
GROUP BY EXTRACT(MONTH FROM DATE_TRUNC('day', creation_date)::date)
)
select month, cnt,
    round((cnt - lag(cnt, 1) over (order by month))::numeric / lag(cnt, 1) over (order by month) * 100, 2)
from ctr;
```
20. Выгрузите данные активности пользователя, который опубликовал больше всего постов за всё время. Выведите данные за октябрь 2008 года в таком виде:
- номер недели;
- дата и время последнего поста, опубликованного на этой неделе.
```
WITH top_user as (
SELECT user_id, count(id) over (partition by user_id) as cnt
FROM stackoverflow.posts
ORDER BY cnt DESC
LIMIT 1
)
SELECT EXTRACT(week from p.creation_date) as week, max(creation_date) as max_dt
    FROM stackoverflow.posts as p
    JOIN top_user as u ON u.user_id = p.user_id
    WHERE p.creation_date::date BETWEEN '2008-10-01' AND '2008-10-31'
GROUP BY EXTRACT(week from p.creation_date)
ORDER BY week;
```
