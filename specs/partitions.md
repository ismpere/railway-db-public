# Partition Specification

## Partition 1

**What:** Partition table cities by country.
**Type:** value list
**Used in:**

- [in transaction 3 when we query data from the cities table for insertion](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#create-stations)

- [in transaction 7 when deleting connections by train producer's country](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction7.md#delete-from-passengers_schedule-for-every-connection-with-this-train)



- [in backup transaction 2 when we are deleting by country (because for most tables, they find out the country via cities](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction2.md#dependencies)

- [in backup transaction 5 we join with cities and also filter by country which again needs city](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction5.md)

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

