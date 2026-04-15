# AI-Assisted SQL Analysis

In this exercise, you'll be working with ChatGPT both to understand existing queries and to construct new queries.

### Part 1

For each query, first try to read it on your own to understand what it accomplishes and what each piece and CTE is doing. Then, have ChatGPT try to break it down for you.
Here are some prompts or prompting strategies you could try:
* Explain this query one CTE at a time.  
* For each CTE, describe:  
    * its purpose  
    * its input tables  
    * what rows it returns  
    * how it is used later
* Describe the flow of data through this query step-by-step.
* Rewrite this query in plain English as if explaining to a non-technical stakeholder.
Once you understand what the query is doing, have ChatGPT help you to make it more efficient, more concise, or more readable.

For each question, consider:
* What did ChatGPT explain clearly?  
* Where was its explanation confusing?  
* Did it make any incorrect claims?  
* Did it suggest a change that altered the results?
* How did the execute time vary between provided query and ChatGPT optimized query?
* If there was a significant difference in the execute times ask ChatGPT to explain where the differences come from.

1.
```
WITH traded AS(
		SELECT playerid, yearid
		FROM batting
		GROUP BY playerid, yearid
		HAVING COUNT(*) > 1
		ORDER BY playerid, yearid),

	league_switch AS(
		SELECT playerid, yearid, b1.teamid AS AL_team, b2.teamid AS NL_team
		FROM traded INNER JOIN batting AS b1 USING (playerid, yearid)
  		          INNER JOIN batting AS b2 USING (playerid, yearid)
		WHERE b1.lgid = 'AL' AND b2.lgid = 'NL'),

	ws_matchups AS(
		SELECT yearid, t1.teamid AS AL_team, t1.franchid AS AL_franch, t2.teamid AS           NL_team, t2.franchid AS NL_franch
		FROM teams AS t1 INNER JOIN teams AS t2 USING (yearid)
		WHERE t1.lgwin = 'Y' and t2.lgwin = 'Y' AND t1.lgid = 'AL' AND t2.lgid = 'NL'
		ORDER BY yearid DESC)

SELECT namefirst, namelast, ws.yearid, al.name AS al_team, nl.name AS nl_team
FROM league_switch AS ls INNER JOIN ws_matchups AS ws ON ls.al_team = ws.al_team AND ls.nl_team = ws.nl_team AND ls.yearid = ws.yearid
                         INNER JOIN people USING (playerid)
						 INNER JOIN teams AS al ON ws.yearid = al.yearid AND ws.al_team = al.teamid
						 INNER JOIN teams AS nl ON ws.yearid = nl.yearid AND ws.nl_team = nl.teamid
ORDER BY yearid;
```

2.
```
WITH months AS (
    SELECT generate_series(
        date_trunc('month', MIN(debut::date)),
		date_trunc('month', MAX(debut::date)),
        interval '1 month'
    ) AS month_start
    FROM people
),

	all_debuts AS(
		SELECT
   		 	date_part('month', month_start) AS month,
			date_part('year',  month_start) AS year,
    		COUNT(people.playerid) as total_debut,
			teams.name AS team
		FROM months LEFT JOIN people ON debut::date >= month_start AND debut::date <  month_start + interval '1 month'
		            LEFT JOIN batting ON people.playerid = batting.playerid and date_part('year', people.debut::date) = batting.yearid
					INNER JOIN teams USING(teamid, yearid)
		GROUP BY month_start, teams.name
		ORDER BY month_start)

SELECT team, CONCAT(month, '-', year)as date, ROUND(AVG(total_debut) OVER(PARTITION BY team ORDER BY year, month ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 1) AS avg_6_month_debut
FROM all_debuts;
```

3.
```
WITH career_details AS(
	SELECT DISTINCT CASE WHEN pitching.g IS NOT NULL THEN 'pitcher'
	            	WHEN batting.g IS NOT NULL THEN 'batter' END AS position
				, date_part('year', debut::date) AS debut_year,(finalgame::date - debut::date) AS career_length
	FROM people LEFT JOIN batting USING (playerid)
	            LEFT JOIN pitching USING (playerid)
	WHERE finalgame IS NOT NULL),

	avg_career_yrs AS(
		SELECT DISTINCT position, debut_year, ROUND(AVG(career_length/365.25) OVER(PARTITION BY debut_year, career_details.position), 2) AS avg_career_yrs
		FROM career_details
		ORDER BY debut_year, position)

SELECT *
FROM avg_career_yrs;
```


### Part 2

In this part, you'll be given a series of questions that need to be answered using [window functions](https://www.thoughtspot.com/sql-tutorial/sql-window-functions). Have ChatGPT help you construct these queries. Recall that if you open the PSQL Tool, you can use \d <tablename> to get a list of each column and datatype in a table, which can help ChatGPT more easily construct a query. As you are doing them, have it break down and explain each part and add comments to explain and complicated steps. After you have gotten a query that works, have ChatGPT review it and offer any suggestions for improvements.

1. Find the team that had the longest streak of playoff appearances. There were no playoffs in 1994, so can you have your query account for that? Make sure that when you count the streak length that you don’t count 1994.

2. Which team has finished with the fewest wins or tied for the fewest wins in their division?

3. Find the most dominant 5 year performances by a team; that is, find the teams that had the highest winning period over a 5 year span. Only consider the modern era (1900 onward). In your output, make sure that you don’t repeat the same team in multiple rows if the 5-year spans overlap. Only show the span with the highest winning percentage. For example, don’t include the Chicago Cubs 1906-1910 and also Chicago Cubs 1905-1909.


### Part 3

LLMs can be helpful at creating regular expressions. Have ChatGPT help you write queries to answer the following questions. After your query has been written check it for any errors or inconstancies. Which edge cases will it not identify?  Can ChatGPT offer any suggestions for how this can be rectified?

1. Find all MLB players whose first and last initials match the first letters of their team's city and mascot name. Example: A player named Aaron Brown playing for the Atlanta Braves would qualify (A.B. → A.B.). Note that you want the first initial of first name to match the first letter of team city and the first initial of last name to match the first letter of team nickname.

2. For this question, use the movies database. Your goal on this question is to capture any movie title that contains "man" referring to a person. For example, "Superman", "Woman", and "Human" are all matches. It would also be a match when Man is standing alone, like in "Dead Man Walking".  However, we don't want "man" when it is part of a word like "Jumanji".
