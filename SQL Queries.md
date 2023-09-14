**Use Dataset Database**

```sql
USE Dataset
Go
```
**Data Cleaning**

1. Rename Columns

```sql
EXEC sp_rename 'movies.title', 'Title', 'COLUMN';
EXEC sp_rename 'movies.year', 'Year', 'COLUMN';
EXEC sp_rename 'movies.runtime', 'Runtime', 'COLUMN';
EXEC sp_rename 'movies.certificate', 'Certificate', 'COLUMN';
EXEC sp_rename 'movies.genre', 'Genre', 'COLUMN';
EXEC sp_rename 'movies.director', 'Director', 'COLUMN';
EXEC sp_rename 'movies.stars', 'Stars', 'COLUMN';
EXEC sp_rename 'movies.rating', 'Rating', 'COLUMN';
EXEC sp_rename 'movies.metascore', 'Metascore', 'COLUMN';
EXEC sp_rename 'movies.votes', 'Votes', 'COLUMN';
EXEC sp_rename 'movies.gross', 'Gross', 'COLUMN';
```
2. Update some columns such as removing special characters

```sql
UPDATE movies
SET Rating = ROUND(Rating, 2);

UPDATE movies
SET Gross = ROUND(Gross, 2);

UPDATE movies
SET Year = REPLACE(Year, LEFT(Year, CHARINDEX('(', Year)), '')
WHERE CHARINDEX('(', Year) > 0;

UPDATE movies
SET Director = CASE
    WHEN Director IS NOT NULL AND CHARINDEX(',', Director) > 0 THEN LEFT(Director, CHARINDEX(',', Director))
    ELSE Director
    END

UPDATE movies
SET Director = LTRIM(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(Director, '"',''), ',',''), '''',''),']',''), '[', ''))

UPDATE movies
SET stars = LTRIM(REPLACE(REPLACE(REPLACE(REPLACE(stars, '"',''), '''',''),']',''), '[', ''))

```
3. Add new column called Title_Year

```sql
ALTER TABLE movies
ADD Title_Year VARCHAR(255) NOT NULL
DEFAULT 'None' WITH VALUES;

UPDATE movies
SET Title_Year = CONCAT(Title, ' (', Year, ')');
```

4. Check Duplicates

```sql
SELECT *
FROM movies
WHERE Title_Year in (
	SELECT 
		Title_Year
	FROM movies
	GROUP BY Title_Year
	HAVING COUNT(*) > 1)
```
Commentary: Even two movies has the same name and same realease year, they are different each other.


**Data Exploratory and answer some questions**

1. How many movies are in the dataset?

```sql
SELECT DISTINCT COUNT(*) DistinctCountMovies
FROM movies
```
2. How many genres has the dataset?

```sql
SELECT COUNT(DISTINCT TRIM(value)) DistinctCountGenres
FROM movies
CROSS APPLY STRING_SPLIT(genre, ',')
```
3. Which genres has more movies on it ?

```sql
SELECT 
	TRIM(value) Genre, 
	Count(TRIM(value)) CountGenres
FROM movies
CROSS APPLY STRING_SPLIT(genre, ',') 
GROUP BY TRIM(value)
ORDER BY 2 DESC
```

4. How many gross was earned by year and grand total?

```sql
SELECT 
	Year, 
	COALESCE(ROUND(SUM(gross),2),0) TotalGross
FROM movies
GROUP BY Year WITH ROLLUP
ORDER BY Year
```
5. How many directors are in the dataset?

```sql
SELECT COUNT(DISTINCT director) DistinctCountDirector
FROM movies
```

6. How many movies does not have gross in the dataset?

```sql
SELECT DISTINCT Count(*) DistinctCountMoviesWithoutGross
FROM movies
WHERE gross IS NULL;
```
7. Who are the top 10 Director that has more movies?

```sql
SELECT 
	Director, 
	CountDirector 
FROM (
	SELECT 
		Director, 
		count(director) CountDirector,
		DENSE_RANK() OVER(ORDER BY COUNT(Director) DESC) RankDirector
	FROM movies	
	GROUP BY Director) temp
WHERE RankDirector <= 10
ORDER BY CountDirector DESC
```

8. Who are the top 10 director that earn more gross?

```sql
SELECT TOP 10 
	Director, 
	SUM(gross) TotalGross
FROM movies
WHERE gross IS NOT NULL
GROUP BY Director
ORDER BY 2 DESC
```

9. Which combination of director and star has worked more than once?

```sql
SELECT DISTINCT  
	CONCAT(Director, ' - ', value), 
	COUNT(CONCAT(Director, ' - ', value)) CountCombination
from movies
CROSS APPLY STRING_SPLIT(stars, ',') 
GROUP BY CONCAT(Director, ' - ', value)
HAVING COUNT(CONCAT(Director, ' - ', value)) > 1
ORDER by CountCombination DESC
```

10.a. Which director has earn more than the others in each year?

```sql
SELECT 
	Year, 
	Director, 
	TotalGross 
FROM (
	SELECT 
		Year, 
		Director, 
		SUM(gross) TotalGross,
		RANK() OVER(PARTITION BY Year ORDER BY SUM(gross) DESC) RankingGross
	FROM movies
	WHERE gross IS NOT NULL
	GROUP BY Director, Year
	) temp
WHERE RankingGross = 1;
```

10.b. Which director has earn less than the others in each year?

```sql
SELECT Year, Director, TotalGross 
	FROM (
	SELECT 
		year, 
		Director, 
		SUM(gross) TotalGross,
		RANK() OVER(PARTITION BY Year ORDER BY SUM(gross)) RankingGross
	FROM movies
	WHERE gross IS NOT NULL
	GROUP BY Director, year
	) temp
WHERE RankingGross = 1;
```

11.a. Which title is ranking number one in each year?

```sql
SELECT year, title, rating 
	FROM (
		SELECT 
			year, 
			title, 
			rating,
			DENSE_RANK() OVER(PARTITION BY year ORDER BY rating DESC) DenseRankRanking
		FROM movies
		GROUP BY year, title, rating
		) temp
WHERE DenseRankRanking = 1;
```

11.b. Which title is the worst ranking in each year?

```sql
SELECT year, title, rating 
	FROM (
		SELECT 
			year, 
			title, 
			rating,
			DENSE_RANK() OVER(PARTITION BY year ORDER BY rating) DenseRankRanking
		FROM movies
		GROUP BY year, title, rating
		) temp
WHERE DenseRankRanking = 1;
```

12. What is the annual increase or growth in total gross earnings from year to year?

```sql
DECLARE @maxYear SMALLINT;
SET @maxYear = (SELECT MAX(year) FROM movies);

WITH gross AS (
	SELECT 
		year, 
		ROUND(SUM(gross),2) TotalGross, 
		LAG(ROUND(SUM(gross),2),1,0) OVER(ORDER BY year) PreviousGross
	FROM movies
	GROUP BY year
)

SELECT 
	Year, 
	CONCAT(
		FORMAT(
			CAST(
				COALESCE(
					ROUND((totalGross - PreviousGross) / NULLIF(PreviousGross, 0),5) * 100
					,0) 
			AS DECIMAL(10,2))
		,'N')
	, '%') as 'Growth Year'
FROM gross
WHERE year <> @maxYear
```
13. Over the past five years, analyze the number of stars that have total gross earnings above and below the average total gross for each respective year.

```sql
DECLARE @avgGrossLast5Years DECIMAL(10,2);

SET @avgGrossLast5Years = (SELECT AVG(gross) FROM movies WHERE gross IS NOT NULL and year >= (@maxYear - 5));

WITH AvgGrossActors as (
	SELECT 
		TRIM(value) Actors,
		SUM(gross) TotalGross,
		(CASE WHEN sum(gross) >  @avgGrossLast5Years THEN 'MoreThanAvg' 
		  WHEN sum(gross) < @avgGrossLast5Years  THEN 'LowerThanAvg'
		  ELSE 'SameAvg'
		  END) as CompareAvgGlobal
	FROM movies
	CROSS APPLY STRING_SPLIT(stars, ',')
	WHERE gross is NOT NULL AND stars IS NOT NULL AND year >= (@maxYear - 5)
	GROUP BY TRIM(value)
)

SELECT 
	CompareAvgGlobal, 
	COUNT(CompareAvgGlobal) as CountCompareAvgGlobal
FROM AvgGrossActors
GROUP BY CompareAvgGlobal
```

14. Which movie has maximium of runtime and minium of runtime?

```sql
DECLARE @maxRuntime SMALLINT;
DECLARE @minRuntime SMALLINT;

SET @maxRuntime = (SELECT MAX(runtime) FROM movies);
SET @minRuntime = (SELECT MIN(runtime) FROM movies);

SELECT 
	Title, 
	'Movie with max runtime' Description, 
	Runtime
FROM movies
WHERE runtime = @maxRuntime
UNION
SELECT 
	Title, 
	'Movie with max runtime' Description, 
	Runtime
FROM movies
WHERE runtime = @minRuntime
```
15. How many certificate are in the dataset?

```sql
SELECT 
	Certificate, 
	COUNT(certificate) CountCertificate
FROM movies
WHERE certificate IS NOT NULL
GROUP BY certificate
ORDER BY CountCertificate desc
```

16. Which move has metascore of 100?

```sql
SELECT title, metascore
FROM movies
WHERE EXISTS (
    SELECT *
    FROM movies AS sub
    WHERE metascore = 100
    AND sub.title = movies.title
	AND sub.year = movies.year
)
```

17. Can you provide a matrix that displays the genre counts for the top 10 directors with the most movies in a given dataset?

```sql
DECLARE @DirectorVsGenre NVARCHAR(max)
DECLARE @ListGenres NVARCHAR(max)

SELECT @ListGenres = ISNULL(@ListGenres + ',', '') + QUOTENAME(TRIM(value))
FROM (
	SELECT DISTINCT 
		TRIM(value) value 
	FROM movies  
	CROSS APPLY STRING_SPLIT(genre, ',') 
) as c ORDER BY value

SET @DirectorVsGenre = '
	SELECT Director, ' + @ListGenres + 
	'FROM (
		SELECT 
			director, 
			value
		FROM movies
		CROSS APPLY STRING_SPLIT(genre, '','') AS SplitGenres
		WHERE director IN (SELECT 
								director
						   FROM (
								SELECT 
              director, 
              COUNT(director) as cant_director,
              RANK() OVER(ORDER BY COUNT(director) DESC) ranking
								FROM movies	
								GROUP BY director) temp
							WHERE ranking <= 10)
	) AS SourceTable
PIVOT
	(
		COUNT(value)
		FOR value IN ('+ @ListGenres +')
	) as PivotTable'

EXEC sp_executesql @DirectorVsGenre
```

18. How can I generate a matrix that displays the total gross earnings for the top 10 stars in each of the last five years, showcasing the year and the top stars for each year?

```sql
DECLARE @YearVsStar NVARCHAR(max)
DECLARE @ListYears NVARCHAR(max)

SELECT @ListYears = ISNULL(@ListYears + ',', '') + QUOTENAME(year)
FROM (
	SELECT DISTINCT 
		year 
	FROM movies 
	WHERE year >= (@maxYear - 5)
) as c ORDER BY year

SET @YearVsStar = '
	SELECT value, ' + @ListYears + '
	FROM (
		SELECT 
			year, 
			gross, 
			value
		FROM movies
		CROSS APPLY STRING_SPLIT(stars, '','')
		WHERE TRIM(value) IN (
							SELECT TOP 10 
								TRIM(value)
							FROM movies 
							CROSS APPLY STRING_SPLIT(stars, '','') 
							WHERE year >= (@maxYear - 5) AND gross IS NOT NULL
							GROUP BY gross,TRIM(value) 
							ORDER BY SUM(gross) DESC)
	) AS SourceTable
PIVOT
	(
		SUM(gross)
		FOR year IN ('+ @ListYears +')
	) as PivotTable'

EXEC sp_executesql @YearVsStar
```












