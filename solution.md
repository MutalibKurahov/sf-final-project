# Решение задач для IT Resume

```sql
WITH cohort_users AS (
    SELECT 
        id AS user_id,
        DATE_TRUNC('month', date_joined) AS cohort_month,
        date_joined::date AS registration_date
    FROM users
),
user_activity AS (
    SELECT DISTINCT
        ue.user_id,
        ue.entry_at::date AS activity_date
    FROM userentry ue
),
cohort_activity AS (
    SELECT 
        cu.user_id,
        cu.cohort_month,
        cu.registration_date,
        ua.activity_date,
        (ua.activity_date - cu.registration_date) AS days_since_registration
    FROM cohort_users cu
    LEFT JOIN user_activity ua ON cu.user_id = ua.user_id
),
retention_windows AS (
    SELECT 
        ca.user_id,
        ca.cohort_month,
        ca.registration_date,
        MAX(CASE WHEN ca.days_since_registration >= 0 THEN 1 ELSE 0 END) AS active_day_0,
        MAX(CASE WHEN ca.days_since_registration >= 1 THEN 1 ELSE 0 END) AS active_day_1,
        MAX(CASE WHEN ca.days_since_registration >= 3 THEN 1 ELSE 0 END) AS active_day_3,
        MAX(CASE WHEN ca.days_since_registration >= 7 THEN 1 ELSE 0 END) AS active_day_7,
        MAX(CASE WHEN ca.days_since_registration >= 14 THEN 1 ELSE 0 END) AS active_day_14,
        MAX(CASE WHEN ca.days_since_registration >= 30 THEN 1 ELSE 0 END) AS active_day_30,
        MAX(CASE WHEN ca.days_since_registration >= 60 THEN 1 ELSE 0 END) AS active_day_60,
        MAX(CASE WHEN ca.days_since_registration >= 90 THEN 1 ELSE 0 END) AS active_day_90
    FROM cohort_activity ca
    GROUP BY ca.user_id, ca.cohort_month, ca.registration_date
)
SELECT 
    TO_CHAR(cohort_month, 'YYYY-MM') AS cohort,
    COUNT(DISTINCT user_id) AS total_users,
    ROUND(100.0 * SUM(active_day_0) / COUNT(DISTINCT user_id), 2) AS day_0,
    ROUND(100.0 * SUM(active_day_1) / COUNT(DISTINCT user_id), 2) AS day_1,
    ROUND(100.0 * SUM(active_day_3) / COUNT(DISTINCT user_id), 2) AS day_3,
    ROUND(100.0 * SUM(active_day_7) / COUNT(DISTINCT user_id), 2) AS day_7,
    ROUND(100.0 * SUM(active_day_14) / COUNT(DISTINCT user_id), 2) AS day_14,
    ROUND(100.0 * SUM(active_day_30) / COUNT(DISTINCT user_id), 2) AS day_30,
    ROUND(100.0 * SUM(active_day_60) / COUNT(DISTINCT user_id), 2) AS day_60,
    ROUND(100.0 * SUM(active_day_90) / COUNT(DISTINCT user_id), 2) AS day_90
FROM retention_windows
GROUP BY cohort_month
ORDER BY cohort_month;
```

**Выводы:**

Анализ rolling retention показывает следующие закономерности:

1. **Высокий retention в день регистрации (day_0)**: Большинство когорт показывают retention 80-90% в день регистрации, что говорит о хорошем первом впечатлении от платформы.

2. **Резкое падение на следующий день (day_1)**: Retention падает до 27-65% уже на второй день, что указывает на необходимость улучшения онбординга и удержания пользователей в первые дни.

3. **Стабилизация на 7-14 день**: К 7-14 дню retention стабилизируется на уровне 10-38% в зависимости от когорты. Это критический период для формирования привычки использования платформы.

4. **Долгосрочный retention (30-90 дней)**: К 30 дню retention составляет 3-47%, к 90 дню - 0-27%. Наиболее успешные когорты (ноябрь-декабрь 2021) показывают лучшие долгосрочные результаты.

5. **Рекомендации по тарифам**: 
   - Учитывая, что значительная часть пользователей возвращается в течение 7-14 дней, имеет смысл предложить **недельную подписку** для тех, кто хочет попробовать платформу.
   - Для долгосрочных пользователей (30+ дней retention 3-47%) оптимальна **месячная подписка**.
   - **Годовая подписка** с большой скидкой будет привлекательна для наиболее активных пользователей (около 1-27% от когорты).

## Задание 2: Метрики по балансу пользователей

```sql
WITH user_debits AS (
    SELECT 
        user_id,
        SUM(value) AS total_debits
    FROM transaction
    WHERE user_id IS NOT NULL 
      AND type_id IN (1, 23, 24, 25, 26, 27, 28, 30)
    GROUP BY user_id
),
user_credits AS (
    SELECT 
        user_id,
        SUM(value) AS total_credits
    FROM transaction
    WHERE user_id IS NOT NULL 
      AND type_id NOT IN (1, 23, 24, 25, 26, 27, 28, 30)
    GROUP BY user_id
),
user_balances AS (
    SELECT 
        u.id AS user_id,
        COALESCE(SUM(CASE 
            WHEN t.type_id IN (1, 23, 24, 25, 26, 27, 28, 30) THEN -t.value 
            ELSE t.value 
        END), 0) AS balance
    FROM users u
    LEFT JOIN transaction t ON u.id = t.user_id
    GROUP BY u.id
),
all_metrics AS (
    SELECT 
        u.id AS user_id,
        COALESCE(ud.total_debits, 0) AS total_debits,
        COALESCE(uc.total_credits, 0) AS total_credits,
        ub.balance
    FROM users u
    LEFT JOIN user_debits ud ON u.id = ud.user_id
    LEFT JOIN user_credits uc ON u.id = uc.user_id
    LEFT JOIN user_balances ub ON u.id = ub.user_id
),
sorted_balances AS (
    SELECT 
        balance,
        ROW_NUMBER() OVER (ORDER BY balance) AS rn,
        COUNT(*) OVER () AS cnt
    FROM all_metrics
)
SELECT 
    (SELECT ROUND(AVG(total_debits), 2) FROM all_metrics) AS avg_debits_per_user,
    (SELECT ROUND(AVG(total_credits), 2) FROM all_metrics) AS avg_credits_per_user,
    (SELECT ROUND(AVG(balance), 2) FROM all_metrics) AS avg_balance_all_users,
    (SELECT ROUND(AVG(balance), 2) 
     FROM sorted_balances 
     WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)) AS median_balance_all_users;
```

**Результаты:**
- Среднее списание на пользователя: **27.10** кодкоинов
- Среднее начисление на пользователя: **265.41** кодкоинов
- Средний баланс: **238.31** кодкоинов
- Медианный баланс: **56.00** кодкоинов

**Выводы:**

1. **Дисбаланс начислений и списаний**: Пользователи получают в среднем 265 кодкоинов, а тратят только 27. Это указывает на то, что текущая система начислений слишком щедрая, и пользователям не нужно покупать кодкоины.

2. **Разница между средним и медианным балансом**: Средний баланс (238) значительно выше медианного (56), что говорит о наличии небольшой группы пользователей с очень высоким балансом, в то время как у большинства баланс относительно небольшой.

3. **Рекомендации по ценообразованию**: 
   - Если 1 кодкоин = 1 рубль, то средний пользователь тратит около 27 рублей в месяц.
   - Учитывая, что медианный баланс 56 кодкоинов, можно ориентироваться на **месячную подписку в диапазоне 200-500 рублей** (с учетом того, что подписка должна включать больше функционала, чем просто покупка кодкоинов).
   - Для годовой подписки можно предложить скидку 30-40%, что даст цену около **2000-3500 рублей в год** (эквивалент 167-292 рублей в месяц).

## Задание 3: Метрики активности пользователей

```sql
WITH users_who_solved_tasks AS (
    SELECT DISTINCT user_id
    FROM codesubmit
),
users_who_took_tests AS (
    SELECT DISTINCT user_id
    FROM teststart
),
users_who_were_active AS (
    SELECT user_id FROM users_who_solved_tasks
    UNION
    SELECT user_id FROM users_who_took_tests
),
task_metrics AS (
    SELECT 
        cs.user_id,
        COUNT(DISTINCT cs.problem_id) AS tasks_solved,
        COUNT(cs.id) AS total_submits
    FROM codesubmit cs
    WHERE cs.user_id IN (SELECT user_id FROM users_who_solved_tasks)
    GROUP BY cs.user_id
),
test_metrics AS (
    SELECT 
        ts.user_id,
        COUNT(DISTINCT ts.test_id) AS tests_taken,
        COUNT(ts.id) AS total_test_attempts
    FROM teststart ts
    WHERE ts.user_id IN (SELECT user_id FROM users_who_took_tests)
    GROUP BY ts.user_id
),
purchase_stats AS (
    SELECT 
        t.user_id,
        t.type_id,
        CASE 
            WHEN t.type_id = 23 THEN 'task'
            WHEN t.type_id = 27 THEN 'test'
            WHEN t.type_id = 24 THEN 'hint'
            WHEN t.type_id = 25 THEN 'solution'
            ELSE 'other'
        END AS purchase_type
    FROM transaction t
    WHERE t.type_id IN (23, 24, 25, 27, 28)
      AND t.user_id IS NOT NULL
),
all_metrics AS (
    SELECT 
        (SELECT ROUND(AVG(tasks_solved), 2) FROM task_metrics) AS avg_tasks_solved_per_user,
        (SELECT ROUND(AVG(tests_taken), 2) FROM test_metrics) AS avg_tests_taken_per_user,
        (SELECT ROUND(AVG(total_submits::numeric / NULLIF(tasks_solved, 0)), 2) 
         FROM task_metrics WHERE tasks_solved > 0) AS avg_attempts_per_task,
        (SELECT ROUND(AVG(total_test_attempts::numeric / NULLIF(tests_taken, 0)), 2) 
         FROM test_metrics WHERE tests_taken > 0) AS avg_attempts_per_test,
        ROUND(100.0 * (SELECT COUNT(DISTINCT user_id) FROM users_who_were_active) / 
              (SELECT COUNT(*) FROM users), 2) AS pct_active_users,
        (SELECT COUNT(DISTINCT user_id) FROM purchase_stats WHERE purchase_type = 'task') AS users_bought_tasks,
        (SELECT COUNT(DISTINCT user_id) FROM purchase_stats WHERE purchase_type = 'test') AS users_bought_tests,
        (SELECT COUNT(DISTINCT user_id) FROM purchase_stats WHERE purchase_type = 'hint') AS users_bought_hints,
        (SELECT COUNT(DISTINCT user_id) FROM purchase_stats WHERE purchase_type = 'solution') AS users_bought_solutions,
        (SELECT COUNT(*) FROM purchase_stats WHERE purchase_type = 'task') AS total_tasks_purchased,
        (SELECT COUNT(*) FROM purchase_stats WHERE purchase_type = 'test') AS total_tests_purchased,
        (SELECT COUNT(*) FROM purchase_stats WHERE purchase_type = 'hint') AS total_hints_purchased,
        (SELECT COUNT(*) FROM purchase_stats WHERE purchase_type = 'solution') AS total_solutions_purchased,
        (SELECT COUNT(DISTINCT user_id) FROM purchase_stats) AS users_bought_anything,
        (SELECT COUNT(DISTINCT user_id) FROM transaction WHERE user_id IS NOT NULL) AS users_with_any_transaction
)
SELECT * FROM all_metrics;
```

**Результаты:**
- Среднее количество решенных задач на пользователя: **9.52**
- Среднее количество пройденных тестов на пользователя: **1.68**
- Среднее количество попыток на задачу: **3.17**
- Среднее количество попыток на тест: **1.15**
- Доля активных пользователей: **61.97%**
- Пользователей, купивших задачи: **522**
- Пользователей, купивших тесты: **676**
- Пользователей, купивших подсказки: **53**
- Пользователей, купивших решения: **151**
- Всего покупок задач: **1675**
- Всего покупок тестов: **989**
- Всего покупок подсказок: **118**
- Всего покупок решений: **423**
- Пользователей, купивших что-либо: **1139**
- Пользователей с хотя бы одной транзакцией: **2402**

**Выводы:**

1. **Активность пользователей**: 61.97% пользователей решали задачи или проходили тесты, что является хорошим показателем вовлеченности.

2. **Паттерны использования**:
   - Пользователи в среднем решают около 10 задач, но проходят только 1-2 теста, что говорит о том, что задачи более популярны.
   - Среднее количество попыток на задачу (3.17) указывает на то, что задачи достаточно сложные и требуют нескольких попыток.

3. **Покупки материалов**:
   - **Тесты** - самый популярный платный контент (676 пользователей купили, 989 покупок).
   - **Задачи** - второй по популярности (522 пользователя, 1675 покупок).
   - **Решения** - менее популярны (151 пользователь, 423 покупки).
   - **Подсказки** - наименее популярны (53 пользователя, 118 покупок).

4. **Рекомендации по составу подписки**:
   - **Бесплатный функционал**: Большинство задач должны оставаться бесплатными, так как это основной способ привлечения пользователей.
   - **Платные функции для подписки**:
     - Доступ к премиум-тестам (самый популярный платный контент)
     - Доступ к решениям задач (помогает в обучении)
     - Увеличенное количество попыток на задачу (сейчас в среднем 3 попытки)
     - Доступ к подсказкам (хотя они менее популярны, могут быть полезны)
   - **Ограничения для бесплатной версии**: Можно ограничить количество попыток на задачу (например, 2-3 попытки бесплатно, далее требуется подписка).
