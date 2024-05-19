# Machine Learning using Soccer Data

## Calculate shot distance and shot angle

In this section, you will create some user-defined functions in BigQuery that help with shot distance and angle calculations, helping to prepare the soccer event data for eventual use in an ML model.

```
CREATE FUNCTION `soccer.GetShotDistanceToGoal`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 /* Translate 0-100 (x,y) coordinate-based distances to absolute positions
 using "average" field dimensions of 105x68 before combining in 2D dist calc */
 SQRT(
   POW((100 - x) * 105/100, 2) +
   POW((50 - y) * 68/100, 2)
   )
 );
```

```
CREATE FUNCTION `soccer.GetShotAngleToGoal`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 SAFE.ACOS(
   /* Have to translate 0-100 (x,y) coordinates to absolute positions using
   "average" field dimensions of 105x68 before using in various distance calcs */
   SAFE_DIVIDE(
     ( /* Squared distance between shot and 1 post, in meters */
       (POW(105 - (x * 105/100), 2) + POW(34 + (7.32/2) - (y * 68/100), 2)) +
       /* Squared distance between shot and other post, in meters */
       (POW(105 - (x * 105/100), 2) + POW(34 - (7.32/2) - (y * 68/100), 2)) -
       /* Squared length of goal opening, in meters */
       POW(7.32, 2)
     ),
     (2 *
       /* Distance between shot and 1 post, in meters */
       SQRT(POW(105 - (x * 105/100), 2) + POW(34 + 7.32/2 - (y * 68/100), 2)) *
       /* Distance between shot and other post, in meters */
       SQRT(POW(105 - (x * 105/100), 2) + POW(34 - 7.32/2 - (y * 68/100), 2))
     )
    )
  /* Translate radians to degrees */
  ) * 180 / ACOS(-1)
 )
;
```


## Create expected goals models using BigQuery ML

In this section, you will use BigQuery ML to create and execute machine learning models in BigQuery using standard SQL queries. In this case, you will build expected goals models on the soccer event data to predict the likelihood of a shot going in for a goal given its type, distance, and angle (these are known to be good predictors of the chance of a goal, as demonstrated in the BigQuery Soccer Data Analytical Insight lab).

Expected goals models are commonly used in soccer analytics to measure the quality of shots and finishing/saving ability given shot quality, and they have a variety of applications in both retrospective match analysis and making projections.

### Fitting a logistic regression model for expected goals

```
CREATE MODEL `soccer.xg_logistic_reg_model`
OPTIONS(
 model_type = 'LOGISTIC_REG',
 input_label_cols = ['isGoal']
 ) AS

SELECT
 Events.subEventName AS shotType,

 /* 101 is known Tag for 'goals' from goals table */
 (101 IN UNNEST(Events.tags.id)) AS isGoal,

  `soccer.GetShotDistanceToGoal`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotDistance,

 `soccer.GetShotAngleToGoal`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotAngle

FROM
 `soccer.events` Events

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

### Understanding the Logistic regression model fit

Now that you've fit the logistic regression model, you can use BigQuery ML's weights functionality to see the model coefficients for each predictor.

```
SELECT
 *
FROM
 ML.WEIGHTS(MODEL soccer.xg_logistic_reg_model)
;
```

### Create a boosted tree model for expected goals

Next, you will fit a boosted tree model to the data using BigQuery ML's implementation of XGBoost, and compare the results to the previously fit logistic regression model. In theory, boosted trees can be more accurate because of their ability to take into account nonlinear relationships between features and the outcome as well as interactions among features.

```
CREATE MODEL `soccer.xg_boosted_tree_model`
OPTIONS(
 model_type = 'BOOSTED_TREE_CLASSIFIER',
 input_label_cols = ['isGoal']
 ) AS

SELECT
 Events.subEventName AS shotType,

 /* 101 is known Tag for 'goals' from goals table */
 (101 IN UNNEST(Events.tags.id)) AS isGoal,
  `soccer.GetShotDistanceToGoal`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotDistance,

 `soccer.GetShotAngleToGoal`(Events.positions[ORDINAL(1)].x,
   Events.positions[ORDINAL(1)].y) AS shotAngle

FROM
 `soccer.events` Events

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

## Apply an expected goals model to new data

Now that you've fit an expected goals model of reasonable accuracy and explainability, you can apply it to "new" data - in this case, the 2018 World Cup (which was left out of the model fitting).

The logistic regression model created in the previous step will be used to assess the difficulty of each shot and goal in that competition, enabling identification of the most "impressive" goals in the tournament.

### Get probabilities for all shots in 2018 World Cup

You can use BigQuery ML's prediction functionality with the logistic regression model fit in the previous step (or even the boosted tree model, if you prefer) to look at the probability of each shot scoring or not in the World Cup.

```
SELECT
 *

FROM
 ML.PREDICT(
   MODEL `soccer.xg_logistic_reg_model`,
   (
     SELECT
       Events.subEventName AS shotType,

       /* 101 is known Tag for 'goals' from goals table */
       (101 IN UNNEST(Events.tags.id)) AS isGoal,

       `soccer.GetShotDistanceToGoal`(Events.positions[ORDINAL(1)].x,
           Events.positions[ORDINAL(1)].y) AS shotDistance,

       `soccer.GetShotAngleToGoal`(Events.positions[ORDINAL(1)].x,
           Events.positions[ORDINAL(1)].y) AS shotAngle

     FROM
       `soccer.events` Events

     LEFT JOIN
       `soccer.matches` Matches ON
           Events.matchId = Matches.wyId

     LEFT JOIN
       `soccer.competitions` Competitions ON
           Matches.competitionId = Competitions.wyId

     WHERE
       /* Look only at World Cup matches for model predictions */
       Competitions.name = 'World Cup' AND
       /* Includes both "open play" & free kick shots (including penalties) */
       (
           eventName = 'Shot' OR
           (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
       )
   )
 )
```


### Identify most unlikely goals using model probabilities

```
SELECT
 predicted_isGoal_probs[ORDINAL(1)].prob AS predictedGoalProb,
 * EXCEPT (predicted_isGoal, predicted_isGoal_probs),

FROM
 ML.PREDICT(
   MODEL `soccer.xg_logistic_reg_model`,
   (
     SELECT
       Events.playerId,
       (Players.firstName || ' ' || Players.lastName) AS playerName,

       Teams.name AS teamName,
       CAST(Matches.dateutc AS DATE) AS matchDate,
       Matches.label AS match,

     /* Convert match period and event seconds to minute of match */
       CAST((CASE
         WHEN Events.matchPeriod = '1H' THEN 0
         WHEN Events.matchPeriod = '2H' THEN 45
         WHEN Events.matchPeriod = 'E1' THEN 90
         WHEN Events.matchPeriod = 'E2' THEN 105
         ELSE 120
         END) +
         CEILING(Events.eventSec / 60) AS INT64)
         AS matchMinute,

       Events.subEventName AS shotType,

       /* 101 is known Tag for 'goals' from goals table */
       (101 IN UNNEST(Events.tags.id)) AS isGoal,

       `soccer.GetShotDistanceToGoal`(Events.positions[ORDINAL(1)].x,
           Events.positions[ORDINAL(1)].y) AS shotDistance,

       `soccer.GetShotAngleToGoal`(Events.positions[ORDINAL(1)].x,
           Events.positions[ORDINAL(1)].y) AS shotAngle

     FROM
       `soccer.events` Events

     LEFT JOIN
       `soccer.matches` Matches ON
           Events.matchId = Matches.wyId

     LEFT JOIN
       `soccer.competitions` Competitions ON
           Matches.competitionId = Competitions.wyId

     LEFT JOIN
       `soccer.players` Players ON
           Events.playerId = Players.wyId

     LEFT JOIN
       `soccer.teams` Teams ON
           Events.teamId = Teams.wyId

     WHERE
       /* Look only at World Cup matches to apply model */
       Competitions.name = 'World Cup' AND
       /* Includes both "open play" & free kick shots (but not penalties) */
       (
         eventName = 'Shot' OR
         (eventName = 'Free Kick' AND subEventName IN ('Free kick shot'))
       ) AND
       /* Filter only to goals scored */
       (101 IN UNNEST(Events.tags.id))
   )
 )

ORDER BY
  predictedgoalProb
```