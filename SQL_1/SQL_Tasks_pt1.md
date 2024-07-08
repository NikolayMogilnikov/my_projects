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
SELECT title
    , user_id
    , score
    , ROUND (AVG(score) OVER (partition by user_id),0)
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
