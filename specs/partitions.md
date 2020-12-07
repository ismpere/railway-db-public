# Partition Specification

## Partition 1

**What:** Partition table cities by country.
**Type:** value list
**Used in:**

- In query 3 when we query data from the cities table for insertion

```sql
	INSERT INTO stations (name, nr_platforms, city, date_created) VALUES (
		'Görlitz HBF', 12, (
			SELECT city_id FROM cities WHERE name = 'Görlitz'
		), now()::date
	);
	Insert on stations  (cost=373.00..373.02 rows=1 width=48)
	  InitPlan 1 (returns $0)
	    ->  Seq Scan on cities  (cost=0.00..373.00 rows=100 width=4)
	          Filter: ((name)::text = 'Görlitz'::text)
	  ->  Result  (cost=0.00..0.02 rows=1 width=48)
2020-11-23 11:45:01.031 CET [3416] LOG:  duration: 0.124 ms  plan:
	Query Text: SELECT 1 FROM ONLY "public"."cities" x WHERE "city_id" OPERATOR(pg_catalog.=) $1 FOR KEY SHARE OF x
	LockRows  (cost=0.29..8.31 rows=1 width=10)
	  ->  Index Scan using cities_pkey on cities x  (cost=0.29..8.30 rows=1 width=10)
	        Index Cond: (city_id = 10544)
2020-11-23 11:45:01.031 CET [3416] CONTEXT:  SQL statement "SELECT 1 FROM ONLY "public"."cities" x WHERE "city_id" OPERATOR(pg_catalog.=) $1 FOR KEY SHARE OF x"
2020-11-23 11:45:01.031 CET [3416] LOG:  duration: 2.742 ms  plan:
	Query Text: INSERT INTO stations (name, nr_platforms, city, date_created) VALUES (
		'Berlin Görlitzer Bhf', 6, (
			SELECT city_id FROM cities WHERE name LIKE '%Berlin%'
		), '1940-03-14'
	);
	Insert on stations  (cost=373.00..373.01 rows=1 width=48)
	  InitPlan 1 (returns $0)
	    ->  Seq Scan on cities  (cost=0.00..373.00 rows=6 width=4)
	          Filter: ((name)::text ~~ '%Berlin%'::text)
	  ->  Result  (cost=0.00..0.01 rows=1 width=48)
```  

- in query 7 when deleting connections by train producer's country


```sql
Query Text: DELETE FROM ONLY "public"."passengers_schedule" WHERE $1 OPERATOR(pg_catalog.=) "connection"
	Delete on passengers_schedule  (cost=0.00..350.38 rows=2 width=6)
	  ->  Seq Scan on passengers_schedule  (cost=0.00..350.38 rows=2 width=6)
	        Filter: (72 = connection)

Query Text: DELETE FROM schedule
	WHERE train IN (
		SELECT train_id FROM trains
		JOIN producers ON producer = producer_id
		JOIN cities ON city = city_id
		JOIN countries ON country = country_id
		WHERE train_state = 'out_of_order'
		AND countries.name = 'Germany'
	);
	Delete on schedule  (cost=1460.71..1909.74 rows=29 width=30)
	  ->  Hash Semi Join  (cost=1460.71..1909.74 rows=29 width=30)
	        Hash Cond: ((schedule.train)::text = (trains.train_id)::text)
	        ->  Seq Scan on schedule  (cost=0.00..397.41 rows=19541 width=37)
	        ->  Hash  (cost=1460.31..1460.31 rows=32 width=55)
	              Buckets: 2048 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 141kB
	              ->  Hash Join  (cost=963.45..1460.31 rows=32 width=55)
	                    Hash Cond: (trains.producer = producers.producer_id)
	                    ->  Seq Scan on trains  (cost=0.00..469.71 rows=7154 width=41)
	                          Filter: (train_state = 'out_of_order'::train_state)
	                    ->  Hash  (cost=962.33..962.33 rows=90 width=22)
	                          Buckets: 2048 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 95kB
	                          ->  Hash Join  (cost=382.43..962.33 rows=90 width=22)
	                                Hash Cond: (producers.city = cities.city_id)
	                                ->  Seq Scan on producers  (cost=0.00..504.00 rows=20000 width=14)
	                                ->  Hash  (cost=381.30..381.30 rows=90 width=16)
	                                      Buckets: 2048 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 85kB
	                                      ->  Hash Join  (cost=4.80..381.30 rows=90 width=16)
	                                            Hash Cond: (cities.country = countries.country_id)
	                                            ->  Seq Scan on cities  (cost=0.00..323.00 rows=20000 width=14)
	                                            ->  Hash  (cost=4.79..4.79 rows=1 width=10)
	                                                  Buckets: 1024  Batches: 1  Memory Usage: 9kB
	                                                  ->  Seq Scan on countries  (cost=0.00..4.79 rows=1 width=10)
	                                                        Filter: ((name)::text = 'Germany'::text)
```

- in backup transaction 2 when we are deleting by country (because for most tables, they find out the country via cities)

```sql
Query Text: UPDATE ONLY "public"."passengers" SET "city" = NULL WHERE $1 OPERATOR(pg_catalog.=) "city"
	Update on passengers  (cost=0.00..571.00 rows=100 width=174)
	  ->  Seq Scan on passengers  (cost=0.00..571.00 rows=100 width=174)
	        Filter: ($1 = city)
```

```sql
Query Text: UPDATE ONLY "public"."producers" SET "city" = NULL WHERE $1 OPERATOR(pg_catalog.=) "city"
Update on producers  (cost=0.00..554.00 rows=100 width=142)
  ->  Seq Scan on producers  (cost=0.00..554.00 rows=100 width=142)
        Filter: ($1 = city)
```

```sql
Query Text: DELETE FROM ONLY "public"."stations" WHERE $1 OPERATOR(pg_catalog.=) "city"
	Delete on stations  (cost=0.00..402.59 rows=99 width=6)
	  ->  Seq Scan on stations  (cost=0.00..402.59 rows=99 width=6)
		    Filter: ($1 = city)
```

- in backup transaction 5 we join with cities and also filter by country which again needs city

```sql
Query Text: /*
	* Select stations with random criteria.
	*
	* EXEC SCRIPT:
	psql [DB-NAME] < transactions/backup_transaction5.sql
	*/
	SELECT stations.name, date_created, nr_platforms, distance
	FROM stations JOIN cities ON city = city_id
	JOIN countries ON country = country_id
	CROSS JOIN routes
	WHERE (
		start_station = station_id
		OR end_station = station_id
	)
	AND country = (
		SELECT country_id
		FROM countries
		WHERE name = 'United States'
	)
	AND nr_platforms > 5
	AND (EXTRACT(year FROM date_created)) > 1950
	AND distance > 1000;
	Nested Loop  (cost=378.96..5888.21 rows=26 width=28)
	  Join Filter: ((routes.start_station = stations.station_id) OR (routes.end_station = stations.station_id))
	  InitPlan 1 (returns $0)
	    ->  Seq Scan on countries countries_1  (cost=0.00..4.79 rows=1 width=4)
	          Filter: ((name)::text = 'United States'::text)
	  ->  Seq Scan on routes  (cost=0.00..369.54 rows=10042 width=12)
	        Filter: (distance > 1000)
	  ->  Materialize  (cost=374.18..944.84 rows=26 width=28)
	        ->  Nested Loop  (cost=374.18..944.71 rows=26 width=28)
	              ->  Seq Scan on countries  (cost=0.00..4.79 rows=1 width=4)
	                    Filter: (country_id = $0)
	              ->  Hash Join  (cost=374.18..939.66 rows=26 width=32)
	                    Hash Cond: (stations.city = cities.city_id)
	                    ->  Seq Scan on stations  (cost=0.00..551.14 rows=5466 width=32)
	                          Filter: ((nr_platforms > 5) AND (date_part('year'::text, (date_created)::timestamp without time zone) > '1950'::double precision))
	                    ->  Hash  (cost=373.00..373.00 rows=94 width=8)
	                          Buckets: 8192 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 226kB
	                          ->  Seq Scan on cities  (cost=0.00..373.00 rows=94 width=8)
	                                Filter: (country = $0)
```

### Performance improvements
****


## Partition 2

**What:** Partition table trains by train_type.
**Type:** value list
**Used in:**

- in transaction 1 where available seats are fetched by train

```sql
Query Text: (
					SELECT seat_id
					FROM seats
					JOIN wagons ON wagon = wagon_id
					JOIN trains ON wagons.train = train_id
					JOIN schedule ON schedule.train = train_id
					WHERE connection_id = $1
				)
	Hash Join  (cost=849.00..18098.94 rows=45 width=4)
	  Hash Cond: (seats.wagon = wagons.wagon_id)
	  ->  Seq Scan on seats  (cost=0.00..13872.72 rows=900472 width=8)
	  ->  Hash  (cost=848.97..848.97 rows=2 width=4)
	        Buckets: 1024  Batches: 1  Memory Usage: 9kB
	        ->  Nested Loop  (cost=8.60..848.97 rows=2 width=4)
	              ->  Hash Join  (cost=8.32..848.30 rows=2 width=66)
	                    Hash Cond: ((wagons.train)::text = (schedule.train)::text)
	                    ->  Seq Scan on wagons  (cost=0.00..734.76 rows=40076 width=35)
	                    ->  Hash  (cost=8.30..8.30 rows=1 width=31)
	                          Buckets: 1024  Batches: 1  Memory Usage: 9kB
	                          ->  Index Scan using schedule_pkey on schedule  (cost=0.29..8.30 rows=1 width=31)
	                                Index Cond: (connection_id = 1)
	              ->  Index Only Scan using trains_pkey on trains  (cost=0.29..0.34 rows=1 width=31)
	                    Index Cond: (train_id = (wagons.train)::text)
```

- in transaction 3 in join statement and inserting new connections

```sql
2020-11-23 11:45:03.057 CET [3416] LOG:  duration: 0.013 ms  plan:
	Query Text: SELECT 1 FROM ONLY "public"."routes" x WHERE "route_id" OPERATOR(pg_catalog.=) $1 FOR KEY SHARE OF x
	LockRows  (cost=0.29..8.31 rows=1 width=10)
	  ->  Index Scan using routes_pkey on routes x  (cost=0.29..8.30 rows=1 width=10)
	        Index Cond: (route_id = 20001)
2020-11-23 11:45:03.057 CET [3416] CONTEXT:  SQL statement "SELECT 1 FROM ONLY "public"."routes" x WHERE "route_id" OPERATOR(pg_catalog.=) $1 FOR KEY SHARE OF x"
2020-11-23 11:45:03.058 CET [3416] LOG:  duration: 0.012 ms  plan:
	Query Text: SELECT 1 FROM ONLY "public"."trains" x WHERE "train_id"::pg_catalog.text OPERATOR(pg_catalog.=) $1::pg_catalog.text FOR KEY SHARE OF x
	LockRows  (cost=0.29..8.31 rows=1 width=10)
	  ->  Index Scan using trains_pkey on trains x  (cost=0.29..8.30 rows=1 width=10)
	        Index Cond: ((train_id)::text = 'bywdfevctetzjdwvofemzctyerwshr'::text)
2020-11-23 11:45:03.058 CET [3416] CONTEXT:  SQL statement "SELECT 1 FROM ONLY "public"."trains" x WHERE "train_id"::pg_catalog.text OPERATOR(pg_catalog.=) $1::pg_catalog.text FOR KEY SHARE OF x"
```

```sql
Query Text: WITH selected_routes AS (
		SELECT * FROM routes
		WHERE start_station = (
			SELECT station_id FROM stations WHERE name = 'Görlitz HBF'
		) OR start_station = (
			SELECT station_id FROM stations WHERE name = 'Berlin Görlitzer Bhf'
		) OR end_station = (
			SELECT station_id FROM stations WHERE name = 'Görlitz HBF'
		) OR end_station = (
			SELECT station_id FROM stations WHERE name = 'Berlin Görlitzer Bhf'
		)
	),
	train AS(
		SELECT * FROM trains
		WHERE train_id = 'bywdfevctetzjdwvofemzctyerwshr'
	),
	starting_times AS (
		SELECT start_time
		FROM  generate_series(
			now(), now() + interval '1 year', interval '1 hour'
		) AS start_time
	)
	INSERT INTO schedule (route, train, start_time, end_time)
	(
		SELECT route_id, train_id, start_time, ending_times.calc
		FROM selected_routes AS sr1
		CROSS JOIN train
		CROSS JOIN starting_times
		CROSS JOIN LATERAL (
			SELECT calc_end_time(start_time::timestamp, (
				SELECT max_speed FROM train_types JOIN train ON train_type = train_type_id
			), (
				SELECT distance FROM selected_routes AS sr2 WHERE sr1.route_id = sr2.route_id
			), (
				SELECT max_speed FROM selected_routes AS sr3 WHERE sr1.route_id = sr3.route_id
			)) AS calc
		) AS ending_times
	);
	
		Insert on schedule  (cost=2154.80..7831997.56 rows=388000 width=56)
		  CTE selected_routes
		    ->  Seq Scan on routes  (cost=1610.35..2126.61 rows=388 width=20)
			  Filter: ((start_station = $0) OR (start_station = $1) OR (end_station = $2) OR (end_station = $3))
			  InitPlan 1 (returns $0)
			    ->  Seq Scan on stations  (cost=0.00..402.59 rows=99 width=4)
				  Filter: ((name)::text = 'Görlitz HBF'::text)
			  InitPlan 2 (returns $1)
			    ->  Seq Scan on stations stations_1  (cost=0.00..402.59 rows=99 width=4)
				  Filter: ((name)::text = 'Berlin Görlitzer Bhf'::text)
			  InitPlan 3 (returns $2)
			    ->  Seq Scan on stations stations_2  (cost=0.00..402.59 rows=99 width=4)
				  Filter: ((name)::text = 'Görlitz HBF'::text)
			  InitPlan 4 (returns $3)
			    ->  Seq Scan on stations stations_3  (cost=0.00..402.59 rows=99 width=4)
				  Filter: ((name)::text = 'Berlin Görlitzer Bhf'::text)
		  CTE train
		    ->  Index Scan using trains_pkey on trains  (cost=0.29..8.30 rows=1 width=48)
			  Index Cond: ((train_id)::text = 'bywdfevctetzjdwvofemzctyerwshr'::text)
		  ->  Nested Loop  (cost=19.89..7829862.64 rows=388000 width=56)
			->  Nested Loop  (cost=0.01..4872.64 rows=388000 width=44)
			      ->  Function Scan on generate_series start_time  (cost=0.01..10.01 rows=1000 width=8)
			      ->  Materialize  (cost=0.00..13.60 rows=388 width=36)
				    ->  Nested Loop  (cost=0.00..11.66 rows=388 width=36)
					  ->  CTE Scan on train  (cost=0.00..0.02 rows=1 width=32)
					  ->  CTE Scan on selected_routes sr1  (cost=0.00..7.76 rows=388 width=4)
			->  Result  (cost=19.88..20.14 rows=1 width=8)
			      InitPlan 7 (returns $6)
				->  Hash Join  (cost=0.03..2.42 rows=1 width=4)
				      Hash Cond: (train_types.train_type_id = train_1.train_type)
				      ->  Seq Scan on train_types  (cost=0.00..2.00 rows=100 width=8)
				      ->  Hash  (cost=0.02..0.02 rows=1 width=4)
					    Buckets: 1024  Batches: 1  Memory Usage: 9kB
					    ->  CTE Scan on train train_1  (cost=0.00..0.02 rows=1 width=4)
			      InitPlan 8 (returns $8)
				->  CTE Scan on selected_routes sr2  (cost=0.00..8.73 rows=2 width=4)
				      Filter: ($7 = route_id)
			      InitPlan 9 (returns $9)
				->  CTE Scan on selected_routes sr3  (cost=0.00..8.73 rows=2 width=4)
				      Filter: ($7 = route_id)
		JIT:
		  Functions: 49
		  Options: Inlining true, Optimization true, Expressions true, Deforming true
```

## Partition 3

**What:** Partition table seats by seat_id
**Type:** range
**Used in:**

- In query backup 3 when we query data from the seats table for show number of seats per wagon

```sql
Query Text: /*
* Show number of seats per wagon in each train.
* → Count nr of seats, group by wagon and train.
*
* EXEC SCRIPT:
psql [DB-NAME] < transactions/backup_transaction4.sql
*/
SELECT COUNT(seat_id), wagon_nr, train_id
FROM seats JOIN wagons ON wagon = wagon_id
JOIN trains ON train = train_id
GROUP BY train_id, wagon_nr;
GroupAggregate  (cost=134154.28..146359.00 rows=320000 width=43)
  Group Key: trains.train_id, wagons.wagon_nr
  ->  Sort  (cost=134154.28..136405.46 rows=900472 width=39)
	Sort Key: trains.train_id, wagons.wagon_nr
	->  Hash Join  (cost=1872.71..20473.65 rows=900472 width=39)
	      Hash Cond: ((wagons.train)::text = (trains.train_id)::text)
	      ->  Hash Join  (cost=1235.71..17472.41 rows=900472 width=39)
		    Hash Cond: (seats.wagon = wagons.wagon_id)
		    ->  Seq Scan on seats  (cost=0.00..13872.72 rows=900472 width=8)
		    ->  Hash  (cost=734.76..734.76 rows=40076 width=39)
			  Buckets: 65536  Batches: 1  Memory Usage: 3291kB
			  ->  Seq Scan on wagons  (cost=0.00..734.76 rows=40076 width=39)
	      ->  Hash  (cost=387.00..387.00 rows=20000 width=31)
		    Buckets: 32768  Batches: 1  Memory Usage: 1487kB
		    ->  Seq Scan on trains  (cost=0.00..387.00 rows=20000 width=31)
JIT:
  Functions: 24
  Options: Inlining false, Optimization false, Expressions true, Deforming true
```

## Partition 4

**What:** Partition wagons by wagon_id
**Type:** range
**Used in:**

## Partition 5

**What:** Partition passengers_schedule by hash of all 3 foreign keys
**Type:** hash
**Used in:**

## Partition 6

**What:** Partition schedule by connection_id
**Type:** hash
**Used in:**

## Partition 7

**What:** Partition routes by route_id
**Type:** hash
**Used in:**

