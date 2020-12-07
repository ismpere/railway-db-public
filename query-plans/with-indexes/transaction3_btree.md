# Query Text: 
	WITH selected_routes AS (
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
# Plan
	Insert on schedule  (cost=560.48..18508.01 rows=6000 width=56) (actual time=6511.935..6511.946 rows=0 loops=1)
	  CTE selected_routes
	    ->  Seq Scan on routes  (cost=33.22..549.48 rows=6 width=20) (actual time=9.655..9.660 rows=2 loops=1)
	          Filter: ((start_station = $0) OR (start_station = $1) OR (end_station = $2) OR (end_station = $3))
	          Rows Removed by Filter: 19563
	          InitPlan 1 (returns $0)
	            ->  Index Scan using index_stations_name on stations  (cost=0.29..8.30 rows=1 width=4) (actual time=0.022..0.024 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Görlitz HBF'::text)
	          InitPlan 2 (returns $1)
	            ->  Index Scan using index_stations_name on stations stations_1  (cost=0.29..8.30 rows=1 width=4) (actual time=0.014..0.015 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Berlin Görlitzer Bhf'::text)
	          InitPlan 3 (returns $2)
	            ->  Index Scan using index_stations_name on stations stations_2  (cost=0.29..8.30 rows=1 width=4) (actual time=0.010..0.011 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Görlitz HBF'::text)
	          InitPlan 4 (returns $3)
	            ->  Index Scan using index_stations_name on stations stations_3  (cost=0.29..8.30 rows=1 width=4) (actual time=0.009..0.010 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Berlin Görlitzer Bhf'::text)
	  CTE train
	    ->  Index Scan using trains_pkey on trains  (cost=0.29..8.30 rows=1 width=47) (actual time=0.141..0.145 rows=1 loops=1)
	          Index Cond: ((train_id)::text = 'bywdfevctetzjdwvofemzctyerwshr'::text)
	  ->  Nested Loop  (cost=2.70..17950.22 rows=6000 width=56) (actual time=19.821..6086.029 rows=17522 loops=1)
	        ->  Nested Loop  (cost=0.01..85.23 rows=6000 width=44) (actual time=12.813..60.659 rows=17522 loops=1)
	              ->  Function Scan on generate_series start_time  (cost=0.01..10.01 rows=1000 width=8) (actual time=2.989..9.901 rows=8761 loops=1)
	              ->  Materialize  (cost=0.00..0.23 rows=6 width=36) (actual time=0.001..0.003 rows=2 loops=8761)
	                    ->  Nested Loop  (cost=0.00..0.20 rows=6 width=36) (actual time=9.813..9.830 rows=2 loops=1)
	                          ->  CTE Scan on train  (cost=0.00..0.02 rows=1 width=32) (actual time=0.144..0.145 rows=1 loops=1)
	                          ->  CTE Scan on selected_routes sr1  (cost=0.00..0.12 rows=6 width=4) (actual time=9.662..9.671 rows=2 loops=1)
	        ->  Result  (cost=2.69..2.95 rows=1 width=8) (actual time=0.333..0.333 rows=1 loops=17522)
	              InitPlan 7 (returns $6)
	                ->  Hash Join  (cost=0.03..2.42 rows=1 width=4) (actual time=0.196..0.207 rows=1 loops=1)
	                      Hash Cond: (train_types.train_type_id = train_1.train_type)
	                      ->  Seq Scan on train_types  (cost=0.00..2.00 rows=100 width=8) (actual time=0.020..0.039 rows=100 loops=1)
	                      ->  Hash  (cost=0.02..0.02 rows=1 width=4) (actual time=0.032..0.033 rows=1 loops=1)
	                            Buckets: 1024  Batches: 1  Memory Usage: 9kB
	                            ->  CTE Scan on train train_1  (cost=0.00..0.02 rows=1 width=4) (actual time=0.004..0.010 rows=1 loops=1)
	              InitPlan 8 (returns $8)
	                ->  CTE Scan on selected_routes sr2  (cost=0.00..0.14 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=17522)
	                      Filter: ($7 = route_id)
	                      Rows Removed by Filter: 1
	              InitPlan 9 (returns $9)
	                ->  CTE Scan on selected_routes sr3  (cost=0.00..0.14 rows=1 width=4) (actual time=0.001..0.001 rows=1 loops=17522)
	                      Filter: ($7 = route_id)
	                      Rows Removed by Filter: 1
