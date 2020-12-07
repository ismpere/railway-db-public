# Prerequisites

## Getting the connection


	Query Text: 
			SELECT connection_id FROM schedule WHERE route = (
				SELECT route_id FROM routes
				WHERE start_station = (
					SELECT station_id
					FROM stations
					WHERE name = $1
				)
				AND end_station = (
					SELECT station_id
					FROM stations
					WHERE name = $2
				)
			)
			AND start_time = $3
			AND train = $4
		
	Index Scan using index_schedule_start_time on schedule  (cost=434.48..442.50 rows=1 width=4) (actual time=5.059..5.062 rows=1 loops=1)
	  Index Cond: (start_time = '2003-08-13 05:01:10.879157'::timestamp without time zone)
	  Filter: ((route = $2) AND ((train)::text = 'cglytjrbckkgdnfegmsjpirvqpkpsh'::text))
	  InitPlan 3 (returns $2)
	    ->  Seq Scan on routes  (cost=16.04..434.48 rows=1 width=4) (actual time=2.545..4.992 rows=1 loops=1)
	          Filter: ((start_station = $0) AND (end_station = $1))
	          Rows Removed by Filter: 19562
	          InitPlan 1 (returns $0)
	            ->  Index Scan using index_stations_name on stations  (cost=0.00..8.02 rows=1 width=4) (actual time=0.053..0.055 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Carnegie Central'::text)
	          InitPlan 2 (returns $1)
	            ->  Index Scan using index_stations_name on stations stations_1  (cost=0.00..8.02 rows=1 width=4) (actual time=0.034..0.036 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Bytča West'::text)

	                  


# Main Query

## Query Text: 
	* A family buys a ticket.
	* → A new row for every person with a given surname is inserted in passengers_schedule.
	*
	* SCRIPT PARAMS:
	* surname: family name, varchar
	* start_station: name of start station of connection, varchar
	* end_station: name of destination station of connection, varchar
	* train_id: code of train, varchar
	* start_time: start time and date of connection, timestamp
	*
	* EXEC SCRIPT:
	* psql -v surname="'SURNAME'" -v name="'NAME'" -v phone_number="'PHONE_NR'" -v start_station="'START'" -v end_station="'END'" -v train_id="'TRAIN'" -v start_time="'DATE & TIME'" [DB-NAME] < transactions/transaction1.sql
	*
	* TEST VALUES:
	psql -v surname="'Cooper'" -v start_station="'Carnegie Central'" -v end_station="'Bytča West'" -v train_id="'cglytjrbckkgdnfegmsjpirvqpkpsh'" -v start_time="'2003-08-13 05:01:10.879157'" [DB-NAME] < transactions/transaction1.sql
	*/
	WITH con AS (
		SELECT connection_id FROM get_connection(
			'Carnegie Central'::varchar,
			'Bytča West'::varchar,
			'2003-08-13 05:01:10.879157'::timestamp,
			'cglytjrbckkgdnfegmsjpirvqpkpsh'::varchar
		)
	),
	free_seats AS (
		SELECT seat FROM (
			SELECT seat FROM get_seats_from_connection((SELECT * FROM con))
			AS train_seats
			WHERE is_taken = FALSE
		) AS seats
	),
	family AS (
		SELECT passenger_id, connection_id, seat FROM passengers, con, free_seats
		WHERE surname = 'Cooper'
	)
	INSERT INTO passengers_schedule (passenger, connection, seat)
	(SELECT passenger_id, connection_id, seat FROM family)
	RETURNING passenger, connection, seat;
	
## Plan

	Insert on passengers_schedule  (cost=44.88..306824.87 rows=24500000 width=12) (actual time=633.772..641.818 rows=1813 loops=1)
	  CTE con
	    ->  Function Scan on get_connection  (cost=0.25..10.25 rows=1000 width=4) (actual time=91.959..91.960 rows=1 loops=1)
	  CTE free_seats
	    ->  Function Scan on get_seats_from_connection train_seats  (cost=20.25..30.25 rows=500 width=4) (actual time=541.620..541.645 rows=37 loops=1)
	          Filter: (NOT is_taken)
	          Rows Removed by Filter: 2
	          InitPlan 2 (returns $1)
	            ->  CTE Scan on con con_1  (cost=0.00..20.00 rows=1000 width=4) (actual time=0.006..0.007 rows=1 loops=1)
	  ->  Nested Loop  (cost=4.38..306784.37 rows=24500000 width=12) (actual time=633.712..636.549 rows=1813 loops=1)
	        ->  CTE Scan on con  (cost=0.00..20.00 rows=1000 width=4) (actual time=91.964..91.965 rows=1 loops=1)
	        ->  Materialize  (cost=4.38..575.62 rows=24500 width=8) (actual time=541.743..544.026 rows=1813 loops=1)
	              ->  Nested Loop  (cost=4.38..453.12 rows=24500 width=8) (actual time=541.740..543.153 rows=1813 loops=1)
	                    ->  CTE Scan on free_seats  (cost=0.00..10.00 rows=500 width=4) (actual time=541.624..541.679 rows=37 loops=1)
	                    ->  Materialize  (cost=4.38..137.00 rows=49 width=4) (actual time=0.003..0.025 rows=49 loops=37)
	                          ->  Bitmap Heap Scan on passengers  (cost=4.38..136.75 rows=49 width=4) (actual time=0.105..0.536 rows=49 loops=1)
	                                Recheck Cond: ((surname)::text = 'Cooper'::text)
	                                Heap Blocks: exact=45
	                                ->  Bitmap Index Scan on index_passengers_surname  (cost=0.00..4.37 rows=49 width=0) (actual time=0.062..0.062 rows=49 loops=1)
	                                      Index Cond: ((surname)::text = 'Cooper'::text)
	JIT:
	  Functions: 20
	  Options: Inlining false, Optimization false, Expressions true, Deforming true
	  Options: Inlining false, Optimization false, Expressions true, Deforming true
