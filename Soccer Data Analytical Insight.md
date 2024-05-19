# Soccer Data Analytical Insight

## Analyze nested soccer event data

In this section, you will run some queries that use JOINs with BigQuery's array functionality to enable better control over the soccer event data.

```
SELECT
 Events.playerId,
 (Players.firstName || ' ' || Players.lastName) AS playerName,
 SUM(IF(Tags2Name.Label = 'assist', 1, 0)) AS numAssists

FROM
 `soccer.events` Events,
 Events.tags Tags

LEFT JOIN
 `soccer.tags2name` Tags2Name ON
   Tags.id = Tags2Name.Tag

LEFT JOIN
 `soccer.players` Players ON
   Events.playerId = Players.wyId

GROUP BY
 playerId, playerName

ORDER BY
 numAssists DESC
```

## Calculate the average pass distance by team

In this section, you will run some queries that use the nested fields in the soccer event data and BigQuery's array functionality and STRUCT data type to answer some interesting questions.

```
WITH
Passes AS
(
 SELECT
   *,
   /* 1801 is known Tag for 'accurate' from tags2name table */
   (1801 IN UNNEST(tags.id)) AS accuratePass,

   (CASE
     WHEN ARRAY_LENGTH(positions) != 2 THEN NULL
     ELSE
  /* Translate 0-100 (x,y) coordinate-based distances to absolute positions
  using "average" field dimensions of 105x68 before combining in 2D dist calc */
       SQRT(
         POW(
           (positions[ORDINAL(2)].x - positions[ORDINAL(1)].x) * 105/100,
           2) +
         POW(
           (positions[ORDINAL(2)].y - positions[ORDINAL(1)].y) * 68/100,
           2)
         )
     END) AS passDistance

 FROM
   `soccer.events`

 WHERE
   eventName = 'Pass'
)

SELECT
 Passes.teamId,
 Teams.name AS team,
 Teams.area.name AS teamArea,

 COUNT(Passes.Id) AS numPasses,
 AVG(Passes.passDistance) AS avgPassDistance,
 SAFE_DIVIDE(
   SUM(IF(Passes.accuratePass, Passes.passDistance, 0)),
   SUM(IF(Passes.accuratePass, 1, 0))
   ) AS avgAccuratePassDistance

FROM
 Passes

LEFT JOIN
 `soccer.teams` Teams ON
   Passes.teamId = Teams.wyId

WHERE
 Teams.type = 'club'

GROUP BY
 teamId, team, teamArea

ORDER BY
 avgPassDistance
```


## Analyze shot distance

In this section, you will create a new query to analyze shot distance.

### What impact does the distance of a shot have on the likelihood of a goal being scored?

To answer this question use a process similar to the previous section. For shots, use (x, y) values from the positions field in the events table.

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
      (100 - positions[ORDINAL(1)].x) * 105/100,
      2) +
    POW(
      (50 - positions[ORDINAL(1)].y) * 68/100,
      2)
     ) AS shotDistance

 FROM
  `soccer.events`

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


# Analyze shot angle

In this section, modify the previous query to look at the impact of the angles on shots.

In this case, the angle calculated is the one made by the location of the shot and the goal line, as shown below (image credit to Ian Dragulet).

Image displaying four different goal shot angle examples

![alt text](https://cdn.qwiklabs.com/OTXlpdshO87DlJLb%2F5TMB5S3AGImP4Y39kf8pBMqWWM%3Dg)

Larger angles arise from being close to the goal and in the center, so this is somewhat correlated with the distance calculation performed above. The shot angle calculations involve using BigQuery's trigonometric functions on the (x, y) data.

```
WITH
Shots AS
(
 SELECT
  *,

  /* 101 is known Tag for 'goals' from goals table */
  (101 IN UNNEST(tags.id)) AS isGoal,

  /* Translate 0-100 (x,y) coordinates to absolute positions using "average"
  field dimensions of 105x68 before using in various distance calcs;
  LEAST used to cap shot locations to on-field (x, y) (i.e. no exact 100s) */
  LEAST(positions[ORDINAL(1)].x, 99.99999) * 105/100 AS shotXAbs,
  LEAST(positions[ORDINAL(1)].y, 99.99999) * 68/100 AS shotYAbs

 FROM
   `soccer.events`

 WHERE
   /* Includes both "open play" & free kick shots (including penalties) */
   eventName = 'Shot' OR
   (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
),

ShotsWithAngle AS
(
 SELECT
   Shots.*,

   /* Law of cosines to get 'open' angle from shot location to goal, given
    that goal opening is 7.32m, placed midway up at field end of (105, 34) */
   SAFE.ACOS(
     SAFE_DIVIDE(
       ( /* Squared distance between shot and 1 post, in meters */
         (POW(105 - shotXAbs, 2) + POW(34 + (7.32/2) - shotYAbs, 2)) +
         /* Squared distance between shot and other post, in meters */
         (POW(105 - shotXAbs, 2) + POW(34 - (7.32/2) - shotYAbs, 2)) -
         /* Squared length of goal opening, in meters */
         POW(7.32, 2)
       ),
       (2 *
         /* Distance between shot and 1 post, in meters */
         SQRT(POW(105 - shotXAbs, 2) + POW(34 + 7.32/2 - shotYAbs, 2)) *
         /* Distance between shot and other post, in meters */
         SQRT(POW(105 - shotXAbs, 2) + POW(34 - 7.32/2 - shotYAbs, 2))
       )
     )
   /* Translate radians to degrees */
   ) * 180 / ACOS(-1)
   AS shotAngle

 FROM
   Shots
)

SELECT
 ROUND(shotAngle, 0) AS ShotAngleRound0,

 COUNT(*) AS numShots,
 SUM(IF(isGoal, 1, 0)) AS numGoals,
 AVG(IF(isGoal, 1, 0)) AS goalPct

FROM
 ShotsWithAngle

GROUP BY
 ShotAngleRound0

ORDER BY
 ShotAngleRound0
```