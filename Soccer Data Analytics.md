# Soccer Data Analytics

## Matches with the most goals

SELECT
 date,
 label,
 (team1.score + team2.score) AS totalGoals
FROM
 `soccer.matches` Matches
LEFT JOIN
 `soccer.competitions` Competitions ON
   Matches.competitionId = Competitions.wyId
WHERE
 status = 'Played' AND
 Competitions.name = 'Spanish first division'
ORDER BY
 totalGoals DESC, date DESC


## Players with the most passes

SELECT
 playerId,
 (Players.firstName || ' ' || Players.lastName) AS playerName,
 COUNT(id) AS numPasses

FROM
 `soccer.events` Events

LEFT JOIN
 `soccer.players` Players ON
   Events.playerId = Players.wyId

WHERE
 eventName = 'Pass'

GROUP BY
 playerId, playerName

ORDER BY
 numPasses DESC

LIMIT 10


## Determine penalty kick success rate

SELECT
 playerId,
 (Players.firstName || ' ' || Players.lastName) AS playerName,
 COUNT(id) AS numPKAtt,
 SUM(IF(101 IN UNNEST(tags.id), 1, 0)) AS numPKGoals,

 SAFE_DIVIDE(
   SUM(IF(101 IN UNNEST(tags.id), 1, 0)),
   COUNT(id)
   ) AS PKSuccessRate

FROM
 `soccer.events` Events

LEFT JOIN
 `soccer.players` Players ON
   Events.playerId = Players.wyId

WHERE
 eventName = 'Free Kick' AND
 subEventName = 'Penalty'

GROUP BY
 playerId, playerName

HAVING
 numPkAtt >= 5

ORDER BY
 PKSuccessRate DESC, numPKAtt DESC