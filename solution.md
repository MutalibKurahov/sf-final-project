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



## Дополнительное задание: Дополнительные метрики

### Метрика 1: Сегментация пользователей по уровню активности и их ценность (LTV)

Мне кажется, что для принятия решения о подписке важно понять, какие пользователи наиболее ценны для платформы и готовы платить. Для этого я предлагаю посчитать сегментацию пользователей по уровню активности и их пожизненную ценность (LTV), потому что:

- Это поможет определить целевую аудиторию для подписки - какие пользователи уже активно используют платформу и могут быть готовы платить
- Покажет разницу в поведении между "power users" и "casual users", что важно для формирования тарифов
- Поможет оценить потенциальный доход от разных сегментов пользователей

Код для расчета:

```sql
WITH user_days_active AS (
    -- Считаем дни активности отдельно
    SELECT 
        user_id,
        COUNT(DISTINCT entry_at::date) AS days_active
    FROM userentry
    GROUP BY user_id
),
user_tasks_stats AS (
    -- Считаем статистику по задачам отдельно
    SELECT 
        user_id,
        COUNT(DISTINCT problem_id) AS tasks_solved,
        COUNT(id) AS total_task_attempts
    FROM codesubmit
    GROUP BY user_id
),
user_tests_stats AS (
    -- Считаем статистику по тестам отдельно
    SELECT 
        user_id,
        COUNT(DISTINCT test_id) AS tests_taken,
        COUNT(id) AS total_test_attempts
    FROM teststart
    GROUP BY user_id
),
user_spending AS (
    -- Считаем траты отдельно
    SELECT 
        user_id,
        SUM(value) AS total_spent
    FROM transaction
    WHERE type_id IN (1, 23, 24, 25, 26, 27, 28, 30)
    GROUP BY user_id
),
user_activity_stats AS (
    -- Объединяем все метрики
    SELECT 
        u.id AS user_id,
        u.date_joined,
        COALESCE(uda.days_active, 0) AS days_active,
        COALESCE(uts.tasks_solved, 0) AS tasks_solved,
        COALESCE(utest.tests_taken, 0) AS tests_taken,
        COALESCE(usp.total_spent, 0) AS total_spent
    FROM users u
    LEFT JOIN user_days_active uda ON u.id = uda.user_id
    LEFT JOIN user_tasks_stats uts ON u.id = uts.user_id
    LEFT JOIN user_tests_stats utest ON u.id = utest.user_id
    LEFT JOIN user_spending usp ON u.id = usp.user_id
),
user_segments AS (
    -- Сегментируем пользователей по активности
    SELECT 
        user_id,
        date_joined,
        days_active,
        tasks_solved,
        tests_taken,
        total_spent,
        CASE 
            WHEN tasks_solved >= 20 OR tests_taken >= 5 OR days_active >= 30 THEN 'Power User'
            WHEN tasks_solved >= 5 OR tests_taken >= 2 OR days_active >= 7 THEN 'Active User'
            WHEN tasks_solved > 0 OR tests_taken > 0 THEN 'Casual User'
            ELSE 'Inactive User'
        END AS user_segment
    FROM user_activity_stats
)
SELECT 
    user_segment,
    COUNT(*) AS user_count,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM user_segments), 2) AS pct_of_total,
    ROUND(AVG(days_active), 2) AS avg_days_active,
    ROUND(AVG(tasks_solved), 2) AS avg_tasks_solved,
    ROUND(AVG(tests_taken), 2) AS avg_tests_taken,
    ROUND(AVG(total_spent), 2) AS avg_spent,
    ROUND(SUM(total_spent), 2) AS total_spent_by_segment,
    ROUND(100.0 * COUNT(CASE WHEN total_spent > 0 THEN 1 END) / COUNT(*), 2) AS pct_paying_users
FROM user_segments
GROUP BY user_segment
ORDER BY 
    CASE user_segment
        WHEN 'Power User' THEN 1
        WHEN 'Active User' THEN 2
        WHEN 'Casual User' THEN 3
        WHEN 'Inactive User' THEN 4
    END;
```

**Выводы:**

Анализ сегментации пользователей показывает четкое разделение на группы:

1. **Power Users (5.91% пользователей)** - наиболее ценная группа:
   - В среднем активны 16 дней, решают 35 задач, проходят 3 теста
   - Средние траты: 229.66 кодкоинов на пользователя
   - 67% из них платят за контент
   - Генерируют 52% всех трат (37,665 из 75,170 кодкоинов)
   - **Рекомендация**: Это основная целевая аудитория для премиум-подписки. Готовы платить за качественный контент.

2. **Active Users (17.63% пользователей)** - стабильная группа:
   - В среднем активны 4 дня, решают 4 задачи, проходят 2 теста
   - Средние траты: 42.57 кодкоинов
   - 69% платят (самый высокий процент конверсии!)
   - **Рекомендация**: Хорошая целевая аудитория для месячной подписки. Высокая готовность платить.

3. **Casual Users (38.86% пользователей)** - самая большая группа:
   - В среднем активны 1.5 дня, решают менее 1 задачи
   - Средние траты: 13.61 кодкоинов
   - Только 44% платят
   - **Рекомендация**: Нужны стимулы для конверсии. Возможно, бесплатный пробный период подписки.

4. **Inactive Users (37.60% пользователей)** - неактивная группа:
   - Почти не используют платформу
   - Только 7% платят
   - **Рекомендация**: Фокус на удержание и онбординг, а не на монетизацию.

**Итог**: 23.54% пользователей (Power + Active) генерируют основную часть дохода и являются основной целевой аудиторией для подписки.

### Метрика 2: Время до первой покупки и конверсия в платящих пользователей

Мне кажется, что важно понять, как быстро пользователи начинают тратить деньги после регистрации и какой процент пользователей вообще когда-либо платил. Для этого я предлагаю посчитать время до первой покупки и конверсию в платящих, потому что:

- Это покажет оптимальный момент для предложения подписки - когда пользователь уже готов платить
- Поможет понять, нужно ли стимулировать первых покупок для увеличения конверсии
- Покажет разницу между пользователями, которые платят сразу, и теми, кто долго "присматривается"

Код для расчета:

```sql
WITH user_registration AS (
    SELECT 
        id AS user_id,
        date_joined
    FROM users
),
first_purchase AS (
    SELECT 
        t.user_id,
        MIN(t.created_at) AS first_purchase_date
    FROM transaction t
    WHERE t.user_id IS NOT NULL 
      AND t.type_id IN (1, 23, 24, 25, 26, 27, 28, 30)  -- только списания
    GROUP BY t.user_id
),
users_with_purchase_info AS (
    SELECT 
        ur.user_id,
        ur.date_joined,
        fp.first_purchase_date,
        CASE 
            WHEN fp.first_purchase_date IS NOT NULL THEN 
                EXTRACT(EPOCH FROM (fp.first_purchase_date - ur.date_joined)) / 86400.0
            ELSE NULL
        END AS days_to_first_purchase
    FROM user_registration ur
    LEFT JOIN first_purchase fp ON ur.user_id = fp.user_id
),
sorted_purchases AS (
    SELECT 
        days_to_first_purchase,
        ROW_NUMBER() OVER (ORDER BY days_to_first_purchase) AS rn,
        COUNT(*) OVER () AS cnt
    FROM users_with_purchase_info
    WHERE days_to_first_purchase IS NOT NULL
)
SELECT 
    (SELECT COUNT(*) FROM users_with_purchase_info) AS total_users,
    (SELECT COUNT(*) FROM users_with_purchase_info WHERE first_purchase_date IS NOT NULL) AS users_who_paid,
    ROUND(100.0 * (SELECT COUNT(*) FROM users_with_purchase_info WHERE first_purchase_date IS NOT NULL) / 
          (SELECT COUNT(*) FROM users_with_purchase_info), 2) AS conversion_to_paying_pct,
    (SELECT ROUND(AVG(days_to_first_purchase)::numeric, 2) FROM users_with_purchase_info WHERE days_to_first_purchase IS NOT NULL) AS avg_days_to_first_purchase,
    (SELECT ROUND(AVG(days_to_first_purchase)::numeric, 2) 
     FROM sorted_purchases 
     WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)) AS median_days_to_first_purchase,
    (SELECT ROUND(MIN(days_to_first_purchase)::numeric, 2) FROM users_with_purchase_info WHERE days_to_first_purchase IS NOT NULL) AS min_days_to_first_purchase,
    (SELECT ROUND(MAX(days_to_first_purchase)::numeric, 2) FROM users_with_purchase_info WHERE days_to_first_purchase IS NOT NULL) AS max_days_to_first_purchase,
    (SELECT COUNT(*) FROM users_with_purchase_info WHERE days_to_first_purchase <= 1) AS paid_within_1_day,
    (SELECT COUNT(*) FROM users_with_purchase_info WHERE days_to_first_purchase <= 7) AS paid_within_7_days,
    (SELECT COUNT(*) FROM users_with_purchase_info WHERE days_to_first_purchase <= 30) AS paid_within_30_days;
```

**Выводы:**

Анализ конверсии в платящих пользователей показывает важные паттерны:

1. **Общая конверсия**: 41.10% пользователей когда-либо платили за контент - это хороший показатель готовности платить.

2. **Скорость конверсии**:
   - **977 пользователей (86% платящих)** платят в первый же день после регистрации!
   - **1049 пользователей (92%)** платят в течение первой недели
   - **1094 пользователей (96%)** платят в течение первого месяца
   - Медианное время до первой покупки: **0 дней** (большинство платят сразу)

3. **Выводы**:
   - Пользователи готовы платить сразу после регистрации - это говорит о высоком доверии к платформе
   - Предложение подписки нужно делать **сразу при регистрации** или в первые дни использования
   - Бесплатный пробный период может быть не так эффективен, как скидка на первую подписку
   - **Рекомендация**: Предлагать подписку в онбординге, сразу после регистрации, с привлекательным предложением для новых пользователей

4. **Среднее время до покупки**: 3.75 дня, но это искажено небольшим количеством пользователей, которые платят позже. Большинство платят сразу.

### Метрика 3: Частота использования платформы (DAU/MAU ratio и паттерны активности)

Мне кажется, что важно понять, как часто пользователи возвращаются на платформу и какие у них паттерны активности. Для этого я предлагаю посчитать соотношение DAU/MAU и частоту использования, потому что:

- Это покажет, насколько "липким" является продукт - возвращаются ли пользователи регулярно
- Поможет определить оптимальные сроки подписки - если пользователи заходят каждый день, им подойдет месячная подписка, если раз в неделю - возможно, лучше годовая
- Покажет, есть ли сезонность или другие паттерны в использовании

Код для расчета:

```sql
WITH monthly_active_users AS (
    -- Пользователи, активные в каждом месяце
    SELECT 
        DATE_TRUNC('month', ue.entry_at) AS activity_month,
        COUNT(DISTINCT ue.user_id) AS mau
    FROM userentry ue
    GROUP BY DATE_TRUNC('month', ue.entry_at)
),
daily_active_users AS (
    -- Пользователи, активные в каждый день
    SELECT 
        DATE_TRUNC('month', ue.entry_at) AS activity_month,
        ue.entry_at::date AS activity_date,
        COUNT(DISTINCT ue.user_id) AS dau
    FROM userentry ue
    GROUP BY DATE_TRUNC('month', ue.entry_at), ue.entry_at::date
),
dau_mau_ratio AS (
    -- Соотношение DAU/MAU по месяцам
    SELECT 
        mau.activity_month,
        mau.mau,
        ROUND(AVG(dau.dau), 2) AS avg_dau,
        ROUND(100.0 * AVG(dau.dau) / mau.mau, 2) AS dau_mau_ratio_pct
    FROM monthly_active_users mau
    LEFT JOIN daily_active_users dau ON mau.activity_month = dau.activity_month
    GROUP BY mau.activity_month, mau.mau
),
user_frequency AS (
    -- Частота использования для каждого пользователя
    SELECT 
        ue.user_id,
        COUNT(DISTINCT ue.entry_at::date) AS total_active_days,
        MIN(ue.entry_at::date) AS first_activity,
        MAX(ue.entry_at::date) AS last_activity,
        (MAX(ue.entry_at::date) - MIN(ue.entry_at::date) + 1) AS days_between_first_last,
        CASE 
            WHEN (MAX(ue.entry_at::date) - MIN(ue.entry_at::date) + 1) > 0 
            THEN COUNT(DISTINCT ue.entry_at::date)::numeric / (MAX(ue.entry_at::date) - MIN(ue.entry_at::date) + 1)
            ELSE 0
        END AS activity_frequency
    FROM userentry ue
    GROUP BY ue.user_id
)
SELECT 
    'DAU/MAU Ratio by Month' AS metric_type,
    TO_CHAR(activity_month, 'YYYY-MM') AS period,
    mau,
    avg_dau,
    dau_mau_ratio_pct
FROM dau_mau_ratio
UNION ALL
SELECT 
    'User Activity Frequency' AS metric_type,
    'Overall' AS period,
    NULL AS mau,
    ROUND(AVG(total_active_days), 2) AS avg_dau,
    ROUND(AVG(activity_frequency) * 100, 2) AS dau_mau_ratio_pct
FROM user_frequency
WHERE days_between_first_last > 0
ORDER BY metric_type, period;
```

**Выводы:**

Анализ частоты использования показывает "липкость" продукта:

1. **DAU/MAU Ratio**: 
   - В среднем составляет 6-12% в зависимости от месяца
   - Это означает, что из всех пользователей, активных в месяце, только 6-12% активны каждый день
   - Для образовательной платформы это нормальный показатель (пользователи не заходят каждый день)

2. **Паттерны по месяцам**:
   - Наиболее активные месяцы: ноябрь-декабрь 2021, февраль-май 2022
   - DAU/MAU ratio растет со временем (с 6.4% в феврале до 11.7% в мае), что говорит об улучшении удержания

3. **Частота активности пользователей**:
   - Средняя частота активности: **77.26%** - это очень высокий показатель!
   - Это означает, что активные пользователи используют платформу в 77% дней между первым и последним визитом
   - Среднее количество активных дней: 2.90

4. **Выводы для подписки**:
   - Пользователи не заходят каждый день, но когда заходят - используют платформу интенсивно
   - **Месячная подписка** оптимальна, так как покрывает период активного использования
   - **Годовая подписка** с большой скидкой привлекательна для тех, кто планирует заниматься долгосрочно
   - Недельная подписка может быть слишком короткой для большинства пользователей

**Рекомендация**: Основной фокус на месячную подписку, годовая - как опция со скидкой для долгосрочных пользователей.
