# Challange

## Task 3

The midpoint & field dimension here is different from the one in tutorial

```
WITH
Shots AS
(
 SELECT
  *,

  /* 101 is known Tag for 'goals' from goals table */
  (101 IN UNNEST(tags.id)) AS isGoal,

  /* Translate 0-100 (x,y) coordinate-based distances to absolute positions
  using "average" field dimensions of 105x68 before combining in 2D dist calc */
  SQRT(
    POW(
      (110 - positions[ORDINAL(1)].x) * 95/100,
      2) +
    POW(
      (60 - positions[ORDINAL(1)].y) * 76/100,
      2)
     ) AS shotDistance

 FROM
  `soccer.events195`

 WHERE
  /* Includes both "open play" & free kick shots (including penalties) */
  eventName = 'Shot' OR
  (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
)

SELECT
 ROUND(shotDistance, 0) AS ShotDistRound0,

 COUNT(*) AS numShots,
 SUM(IF(isGoal, 1, 0)) AS numGoals,
 AVG(IF(isGoal, 1, 0)) AS goalPct

FROM
 Shots

WHERE
 shotDistance <= 50

GROUP BY
 ShotDistRound0

ORDER BY
 ShotDistRound0
```

## Task 4

```
CREATE FUNCTION `soccer.GetShotDistanceToGoal195`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 /* Translate 0-100 (x,y) coordinate-based distances to absolute positions
 using "average" field dimensions of 95x76 before combining in 2D dist calc */
 SQRT(
   POW((110 - x) * 95/100, 2) +
   POW((60 - y) * 76/100, 2)
   )
 );
```


```
CREATE FUNCTION `soccer.GetShotAngleToGoal195`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 SAFE.ACOS(
   /* Have to translate 0-100 (x,y) coordinates to absolute positions using
   "average" field dimensions of 95x76 before using in various distance calcs */
   SAFE_DIVIDE(
     ( /* Squared distance between shot and 1 post, in meters */
       (POW(95 - (x * 95/100), 2) + POW(38 + (7.32/2) - (y * 76/100), 2)) +
       /* Squared distance between shot and other post, in meters */
       (POW(95 - (x * 95/100), 2) + POW(38 - (7.32/2) - (y * 76/100), 2)) -
       /* Squared length of goal opening, in meters */
       POW(7.32, 2)
     ),
     (2 *
       /* Distance between shot and 1 post, in meters */
       SQRT(POW(95 - (x * 95/100), 2) + POW(38 + 7.32/2 - (y * 76/100), 2)) *
       /* Distance between shot and other post, in meters */
       SQRT(POW(95 - (x * 95/100), 2) + POW(38 - 7.32/2 - (y * 76/100), 2))
     )
    )
  /* Translate radians to degrees */
  ) * 180 / ACOS(-1)
 )
;
```


```
CREATE MODEL `soccer.xg_logistic_reg_model_195`
OPTIONS(
 model_type = 'LOGISTIC_REG',
 input_label_cols = ['isGoal']
 ) AS

SELECT
 Events.subEventName AS shotType,

 /* 101 is known Tag for 'goals' from goals table */
 (101 IN UNNEST(Events.tags.id)) AS isGoal,

  `soccer.GetShotDistanceToGoal195`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotDistance,

 `soccer.GetShotAngleToGoal195`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotAngle

FROM
 `soccer.events195` Events

LEFT JOIN
 `soccer.matches` Matches ON
   Events.matchId = Matches.wyId

LEFT JOIN
 `soccer.competitions` Competitions ON
   Matches.competitionId = Competitions.wyId

WHERE
 /* Filter out World Cup matches for model fitting purposes */
 Competitions.name != 'World Cup' AND
 /* Includes both "open play" & free kick shots (including penalties) */
 (
   eventName = 'Shot' OR
   (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
 )
;
```