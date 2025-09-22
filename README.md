# Инструкция по созданию клиентского дашборда

## 1. Приложение ([ссылка](http://77.91.122.214))

- Получить токен ([ссылка](https://oauth.yandex.ru/authorize?response_type=token&client_id=db0084b785964e89908f2b32e246f1de))
*важно быть залогиненым под тем логином, на котором Директ*

- Создать подключение в приложении (выбрав нужные номера целей)
- Грузануть данные единоразовой выгрузкой (в задачах)
- Добавить к задачам CLIENTS (balance_clients, general_full, poisk_zapros, poisk_zapros)

## 2. DBeaver

- Создать представление `general_full_with_rank_and_meta_clean`

В коде `<DB>` заменить на название БД и заменить цели.

Найти все конверсии (`<DB>` заменить):

```sql
SELECT name
FROM system.columns
WHERE database = '<DB>' AND
table = 'general_full' AND name LIKE 'Conversions%';
```

Таблица преобразования целей: [ссылка](https://docs.google.com/spreadsheets/d/1SvNoSatRwc5pO7gxRiU-9mfSsuGpwC3molNbAR7xlYA/edit?usp=sharing)

### Основной код:

```sql

CREATE OR REPLACE VIEW <DB>.general_full_with_rank_and_meta_clean AS
WITH base AS (
    SELECT DISTINCT CampaignId
    FROM <DB>.general_full
    WHERE CampaignId IS NOT NULL
),
weekly AS (
    SELECT CampaignId, toStartOfWeek(Date) AS Week, sum(Cost) AS WeeklyCost
    FROM <DB>.general_full
    WHERE Date >= toStartOfWeek(now() - INTERVAL 10 WEEK)
    GROUP BY CampaignId, toStartOfWeek(Date)
),
week_lagged AS (
    SELECT CampaignId, dateDiff('week', Week, toStartOfWeek(now())) AS week_lag, WeeklyCost
    FROM weekly
    WHERE dateDiff('week', Week, toStartOfWeek(now())) BETWEEN 0 AND 9
),
aggregated AS (
    SELECT
        CampaignId,
        maxIf(WeeklyCost, week_lag = 0) AS week_0_cost,
        maxIf(WeeklyCost, week_lag = 1) AS week_1_cost,
        maxIf(WeeklyCost, week_lag = 2) AS week_2_cost,
        maxIf(WeeklyCost, week_lag = 3) AS week_3_cost,
        maxIf(WeeklyCost, week_lag = 4) AS week_4_cost,
        maxIf(WeeklyCost, week_lag = 5) AS week_5_cost,
        maxIf(WeeklyCost, week_lag = 6) AS week_6_cost,
        maxIf(WeeklyCost, week_lag = 7) AS week_7_cost,
        maxIf(WeeklyCost, week_lag = 8) AS week_8_cost,
        maxIf(WeeklyCost, week_lag = 9) AS week_9_cost
    FROM week_lagged
    GROUP BY CampaignId
),
filled AS (
    SELECT b.CampaignId,
           coalesce(a.week_0_cost, 0) AS week_0_cost,
           coalesce(a.week_1_cost, 0) AS week_1_cost,
           coalesce(a.week_2_cost, 0) AS week_2_cost,
           coalesce(a.week_3_cost, 0) AS week_3_cost,
           coalesce(a.week_4_cost, 0) AS week_4_cost,
           coalesce(a.week_5_cost, 0) AS week_5_cost,
           coalesce(a.week_6_cost, 0) AS week_6_cost,
           coalesce(a.week_7_cost, 0) AS week_7_cost,
           coalesce(a.week_8_cost, 0) AS week_8_cost,
           coalesce(a.week_9_cost, 0) AS week_9_cost
    FROM base b
    LEFT JOIN aggregated a USING (CampaignId)
),

ranked AS (
    SELECT *,
           row_number() OVER (
               ORDER BY
                   if(week_0_cost > 0, 0,
                   if(week_1_cost > 0, 1,
                   if(week_2_cost > 0, 2,
                   if(week_3_cost > 0, 3,
                   if(week_4_cost > 0, 4,
                   if(week_5_cost > 0, 5,
                   if(week_6_cost > 0, 6,
                   if(week_7_cost > 0, 7,
                   if(week_8_cost > 0, 8,
                   if(week_9_cost > 0, 9, 10)))))))))) ASC,
                   week_0_cost DESC, week_1_cost DESC, week_2_cost DESC,
                   week_3_cost DESC, week_4_cost DESC, week_5_cost DESC,
                   week_6_cost DESC, week_7_cost DESC, week_8_cost DESC, week_9_cost DESC
           ) + 9 AS activity_rank
    FROM filled
)

SELECT
    CAST(gf.AccountName AS Nullable(String)) AS AccountName,
    CAST(gf.AdFormat AS Nullable(String)) AS AdFormat,
    CAST(gf.AdGroupId AS Nullable(UInt64)) AS AdGroupId,
    CAST(gf.AdGroupName AS Nullable(String)) AS AdGroupName,
    CAST(gf.AdId AS Nullable(UInt64)) AS AdId,
    CAST(gf.AdNetworkType AS Nullable(String)) AS AdNetworkType,
    CAST(gf.Age AS Nullable(String)) AS Age,
    CAST(gf.AvgClickPosition AS Nullable(Float64)) AS AvgClickPosition,
    CAST(gf.AvgCpc AS Nullable(Float64)) AS AvgCpc,
    CAST(gf.AvgEffectiveBid AS Nullable(Float64)) AS AvgEffectiveBid,
    CAST(gf.AvgImpressionPosition AS Nullable(Float64)) AS AvgImpressionPosition,
    CAST(gf.AvgPageviews AS Nullable(Float64)) AS AvgPageviews,
    CAST(gf.AvgTrafficVolume AS Nullable(Float64)) AS AvgTrafficVolume,
    CAST(gf.BounceRate AS Nullable(String)) AS BounceRate,
    CAST(gf.Bounces AS Nullable(Int32)) AS Bounces,
    CAST(gf.CampaignId AS Nullable(UInt64)) AS CampaignId,
    CAST(gf.CampaignName AS Nullable(String)) AS CampaignName,
    CAST(gf.CampaignType AS Nullable(String)) AS CampaignType,
    CAST(gf.CampaignUrlPath AS Nullable(String)) AS CampaignUrlPath,
    CAST(gf.CarrierType AS Nullable(String)) AS CarrierType,
    CAST(gf.Clicks AS Nullable(Int32)) AS Clicks,
    CAST(gf.ClientLogin AS Nullable(String)) AS ClientLogin,
    CAST(gf.Cost AS Nullable(Float64)) AS Cost,
    CAST(gf.Criteria AS Nullable(String)) AS Criteria,
    CAST(gf.CriteriaId AS Nullable(UInt64)) AS CriteriaId,
    CAST(gf.CriteriaType AS Nullable(String)) AS CriteriaType,
    CAST(gf.Ctr AS Nullable(String)) AS Ctr,
    CAST(gf.Date AS Nullable(Date)) AS Date,
    CAST(gf.Device AS Nullable(String)) AS Device,
    CAST(gf.ExternalNetworkName AS Nullable(String)) AS ExternalNetworkName,
    CAST(gf.Gender AS Nullable(String)) AS Gender,
    CAST(gf.Impressions AS Nullable(Int32)) AS Impressions,
    CAST(gf.IncomeGrade AS Nullable(String)) AS IncomeGrade,
    CAST(gf.LocationOfPresenceId AS Nullable(UInt64)) AS LocationOfPresenceId,
    CAST(gf.LocationOfPresenceName AS Nullable(String)) AS LocationOfPresenceName,
    CAST(gf.MatchType AS Nullable(String)) AS MatchType,
    CAST(gf.MobilePlatform AS Nullable(String)) AS MobilePlatform,
    (
        ifNull(gf.Conversions_310375038_AUTO, 0) + --заменить
        ifNull(gf.Conversions_338617666_AUTO, 0) + --заменить
        ifNull(gf.Conversions_344168214_AUTO, 0) --заменить
    ) AS sum_conversions,
    CAST(r.activity_rank AS Nullable(UInt32)) AS activity_rank
FROM <DB>.general_full gf
LEFT JOIN ranked r ON gf.CampaignId = r.CampaignId;
```

- Создать представление `rk_clients_href ()`

### Код:

```sql
-- Создаём / переопределяем представление с нужными полями
CREATE OR REPLACE VIEW <DB>.rk_clients_href AS
SELECT
  Id,
  TextAd_Href,
  TextAd_Image_OriginalUrl,
  TextAd_Image_PreviewUrl,
  TextAd_Title,
  TextAd_Title2,
  TextAd_Text,
  TextAd_VideoExtension_PreviewUrl,
  TextAd_VideoExtension_ThumbnailUrl
FROM <DB>.rk_CLIENTS;
```

## 3. DataLens

- скопировать работающий воркбук
- скорректировать Датасет (заменить таблицы)
  *важно сначала удалить все таблицы, потом выставить новые в таком же виде. Главное в последнюю очередь добавлють ту что мэтчится по LEFT JOIN*
  
