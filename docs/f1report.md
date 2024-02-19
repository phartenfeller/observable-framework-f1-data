# F1 Report

This will show _data_ about season **2023**.

## Consume SQLite file

Import database:

```js
import {FileAttachment} from "npm:@observablehq/stdlib";
import {SQLiteDatabaseClient} from "npm:@observablehq/sqlite";

const file = await FileAttachment("./data/f1data.sqlite");
display(file)
const db = await SQLiteDatabaseClient.open(file);
display(db)

const constructorResults = await db.sql`
select position, ct.name, points, c.wins
  from constructorstandings c
  join constructor_teams ct
    on c.constructorid = ct.constructorid
 where raceid = (select max(raceid) from races where year = 2023)
 order by position;`;
display(constructorResults);
```

## Display as table

```js
Inputs.table(constructorResults)
```

## Display as bar chart

```js
import * as Plot from "npm:@observablehq/plot";

const chart = Plot.plot({
  marginLeft: 120,
  x: {label: "Points"},
  y: {label: "Team"},
  color: {type: "log", scheme: "turbo"},
  marks: [
    Plot.barX(constructorResults, {x: "points", y: "name", fill: "points", sort: {y: "x", reverse: true}})
  ],
})

display(chart)
```

### Result per Race

As a heatmap:

```js
const raceResults = await db.sql`
select case when ra.round < 10 then '0' else '' end || ra.round
      || ' ' || replace(ra.name, ' Grand Prix', '') as race
     , d.code as driver
     , r.points
     , ra.round
  from results r
  join races ra
    on r.raceid = ra.raceid
   and ra.year = 2023
  join drivers d
    on r.driverid = d.driverid
order by ra.round;
`;

const raceResultsChart = Plot.plot({
  padding: 0,
  width: 1200,
  marginLeft: 120,
  grid: true,
  y: {label: "Race"},
  color: {type: "linear", scheme: "PuRd"},
  marks: [
    Plot.axisX({ label: "Driver", lineWidth: 3, tickRotate: 90, marginBottom: 60}),
    Plot.cell(raceResults, {x: "driver", y: "race", fill: "points", inset: 0.5, padding: 5}),
    Plot.text(raceResults, {x: "driver", y: "race", text: (d) => d.points, fill: "white", stroke: "#2d3336", title: "title"})
  ]
})

display(raceResultsChart)
```

As a boxplot:

```js
const raceResultsBox = Plot.plot({
  marginLeft: 60,
  color: {type: "linear", scheme: "PuRd"},
  x: {
    grid: true,
    label: "Points"
  },
  y: {
    label: "Driver"
  },
  marks: [
    Plot.boxX(raceResults, {x: "points", y: "driver"})
  ]
})

display(raceResultsBox)
```

Facetting:

```js
const constrPointsPerRace = await db.sql`
select ra.round
     , sum(r.points) as points
     , c.name as team
  from results r
  join races ra
    on r.raceid = ra.raceid
   and ra.year = 2023
  join constructor_teams c
    on r.constructorid = c.constructorid
group by ra.round, c.name
;
`;

const constrPointsRaceFacet = Plot.plot({
  grid: true,
  height: 600,
  marginLeft: 120,
  title: 'Title',
  subtitle: 'Subtitle',
  caption: 'Caption',
  color: {
    scheme: "BuRd",
    legend: true
  },
  fy: {
    label: "Team"
  },
  marks: [
    Plot.frame(),
    Plot.dot(constrPointsPerRace, {
      x: "round",
      stroke: "points",
      fy: "team",
      r: (d) => 2 + Math.sqrt(d.points),
     //fx: "driver"
    })
  ]
})

display(constrPointsRaceFacet)
```

## Interactivity

```js
const raceTime = await db.sql`
select case when ra.round < 10 then '0' else '' end || ra.round
      || ' ' || replace(ra.name, ' Grand Prix', '') as race
     , d.code as driver
     , r.milliseconds / 1000 as seconds
     , ra.round
     , c.name as team
     , row_number() over (partition by c.name) as driverNo
  from results r
  join races ra
    on r.raceid = ra.raceid
   and ra.year = 2023
  join drivers d
    on r.driverid = d.driverid
  join constructor_teams c
    on r.constructorid = c.constructorid
order by ra.round, driverNo;
`;

const teams = Array.from(new Set(raceTime.map((d) => d.team)));
const selectedTeam = view(Inputs.select(teams, {value: "Ferrari", label: "Select Team"}));
```

You selected: ${selectedTeam} - Position: ${constructorResults.find((d) => d.name === selectedTeam).position}

```js
const filteredRaceTime = raceTime.filter((d) => d.team === selectedTeam);
const [driver1, driver2] = filteredRaceTime.map((d) => d.driver).filter((d, i, a) => a.indexOf(d) === i);
```

Driver 1: ${driver1} - Driver 2: ${driver2}

```js
const pivotRaceTime = [];
for (let i = 0; i < filteredRaceTime.length; i += 2) {
  const [row1, row2] = [filteredRaceTime[i], filteredRaceTime[i + 1]].sort((a, b) => a.driver === driver1 ? -1 : 1);

  pivotRaceTime.push({
    race: row1.race,
    driver1: row1.driver,
    driver2: row2.driver,
    driver1Seconds: row1.seconds,
    driver2Seconds: row2.seconds,
    diff: row1.seconds - row2.seconds,
    diffText: `${Math.abs(row1.seconds - row2.seconds)} s`,
    driver1Factor: row1.seconds / row2.seconds,
    driver2Factor: row2.seconds / row1.seconds,
  });
}

const teamRaceTime = Plot.plot({
  marginTop: 0,
  marginLeft: 100,
  width: 800,
  y: {grid: true, label: "Race"},
  x: {label: "Seconds", domain: [0.98, 1.02]},
  color: {
    type: "categorical",
    domain: [1, 2],
    unknown: "#aaa",
    // transform: Math.sign,
    tickFormat: (d) => d === 1 ? driver1 : driver2,
    legend: true
  },
  marks: [
    Plot.ruleX([0]),
    Plot.link(
      pivotRaceTime,
      {
        y: "race",
        x1: "driver1Factor",
        x2: "driver2Factor",
        markerStart: "dot",
        markerEnd: "arrow",
        // sort: {y: "x2"},
        stroke: (d) => {
          if (d.diff > 0) {
            return 2;
          } else if (d.diff < 0) {
            return 1;
          } else {
            return null;
          }
        },
        strokeWidth: 2
      }
    ),
    Plot.text(pivotRaceTime, {
      x: "driver2Factor",
      y: "race",
      // filter: "highlight",
      text: "diffText",
      // fill: "currentColor",
      //stroke: "black",
      dy: -12
    })
  ]
});

display(teamRaceTime)
```
