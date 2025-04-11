# SEEKHO-RETENTION-RATE
Q1.The number of those users who returned exactly on the next day (Day 1).
 Code: 
returns_next_day AS (
    SELECT 
        s.signup_on,
        COUNT(DISTINCT a.user_id) AS came_back
    FROM signup_dates s
    JOIN user_activity a
      ON s.user_id = a.user_id
     AND a.activity_date = DATE(s.signup_on, '+1 day')
    GROUP BY s.signup_on
),
The number of those users who returned exactly on the next day (Day 1).
daily_signups AS (
    SELECT 
        signup_on,
        COUNT(user_id) AS total_signups
    FROM signup_dates
    GROUP BY signup_on
)
The Day 1 retention rate, which is the ratio of users who returned on the next day to
the number of new users on the activity_date.
SELECT 
    d.signup_on AS signup_date,
    d.total_signups,
    IFNULL(r.came_back, 0) AS returned_users,
    ROUND(IFNULL(r.came_back, 0) * 1.0 / d.total_signups, 2) AS day1_retention
FROM daily_signups d
LEFT JOIN returns_next_day r
  ON d.signup_on = r.signup_on 
ORDER BY d.signup_on;
Q2.Identify user sessions. A session is defined as a sequence of activities by a user
where the time difference between consecutive events is less than or equal to 30
minutes. If the time between two events exceeds 30 minutes, it's considered the start
of a new session.
 Code : 
WITH raw_events AS (
    SELECT
        user_id,
        event_type,
        event_time,
        LAG(event_time) OVER (
            PARTITION BY user_id 
            ORDER BY event_time
        ) AS last_event_time
    FROM user_events
),
flag_sessions AS (
    SELECT
        user_id,
        event_type,
        event_time,
        CASE 
            WHEN last_event_time IS NULL 
                OR (strftime('%s', event_time) - strftime('%s', last_event_time)) > 1800
            THEN 1
            ELSE 0
        END AS new_session
    FROM raw_events
),

assign_sessions AS (
 SELECT
        user_id,
        event_type,
        event_time,
        SUM(new_session) OVER (
            PARTITION BY user_id 
            ORDER BY event_time 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS session_num
    FROM flag_sessions
),
session_data AS (
    SELECT
        user_id,
        session_num,
        MIN(event_time) AS session_start,
        MAX(event_time) AS session_end,
        COUNT(*) AS total_events,
        printf('%02d:%02d:%02d',
            (julianday(MAX(event_time)) - julianday(MIN(event_time))) * 24,
            ((julianday(MAX(event_time)) - julianday(MIN(event_time))) * 24 * 60) % 60,
            ((julianday(MAX(event_time)) - julianday(MIN(event_time))) * 24 * 60 * 60) % 60
        ) AS session_length
    FROM assign_sessions
    GROUP BY user_id, session_num
)
SELECT
    user_id,
    session_num AS session_id,
    session_start,
    session_end,
    session_length,
    total_events AS event_count
FROM session_data
ORDER BY user_id, session_num;
