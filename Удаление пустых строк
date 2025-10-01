# Удаление пустых строк

1.  Отобразить строки (на примере пустого значения поля `ClientLogin`):

``` sql
SELECT
  *
FROM
  <DB>.general_full
WHERE
  ClientLogin = ''        -- пустая строка
  OR ClientLogin IS NULL; -- либо NULL
```

2.  Проверяем/удаляем

``` sql
-- 1. Сначала проверьте, что фильтр отбирает именно те строки,
--    которые хотите удалить
SELECT
  count(*) AS rows_to_delete
FROM
  <DB>.general_full
WHERE
  ClientLogin = ''      -- пустая строка
  OR ClientLogin IS NULL;
```

``` sql
-- 2. Удаляем отобранные записи
ALTER TABLE <DB>.general_full
  DELETE WHERE ClientLogin = '' OR ClientLogin IS NULL;
```
