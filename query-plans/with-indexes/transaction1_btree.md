# Main

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
	EXPLAIN ANALYZE
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
	Insert on passengers_schedule  (cost=45.17..306825.16 rows=24500000 width=12) (actual time=566.081..573.783 rows=1813 loops=1)
	   CTE con
	     ->  Function Scan on get_connection  (cost=0.25..10.25 rows=1000 width=4) (actual time=24.633..24.634 rows=1 loops=1)
	   CTE free_seats
	     ->  Function Scan on get_seats_from_connection train_seats  (cost=20.25..30.25 rows=500 width=4) (actual time=541.284..541.308 rows=37 loops=1)
		   Filter: (NOT is_taken)
		   Rows Removed by Filter: 2
		   InitPlan 2 (returns $1)
		     ->  CTE Scan on con con_1  (cost=0.00..20.00 rows=1000 width=4) (actual time=0.001..0.002 rows=1 loops=1)
	   ->  Nested Loop  (cost=4.67..306784.66 rows=24500000 width=12) (actual time=566.026..568.697 rows=1813 loops=1)
		 ->  CTE Scan on con  (cost=0.00..20.00 rows=1000 width=4) (actual time=24.637..24.637 rows=1 loops=1)
		 ->  Materialize  (cost=4.67..575.91 rows=24500 width=8) (actual time=541.386..543.517 rows=1813 loops=1)
		       ->  Nested Loop  (cost=4.67..453.41 rows=24500 width=8) (actual time=541.378..542.578 rows=1813 loops=1)
			     ->  CTE Scan on free_seats  (cost=0.00..10.00 rows=500 width=4) (actual time=541.287..541.344 rows=37 loops=1)
			     ->  Materialize  (cost=4.67..137.28 rows=49 width=4) (actual time=0.002..0.018 rows=49 loops=37)
				   ->  Bitmap Heap Scan on passengers  (cost=4.67..137.04 rows=49 width=4) (actual time=0.082..0.341 rows=49 loops=1)
					 Recheck Cond: ((surname)::text = 'Cooper'::text)
					 Heap Blocks: exact=45
					 ->  Bitmap Index Scan on index_passengers_surname  (cost=0.00..4.65 rows=49 width=0) (actual time=0.052..0.052 rows=49 loops=1)
					       Index Cond: ((surname)::text = 'Cooper'::text)
	 Planning Time: 1.187 ms
	 Trigger for constraint passengers_schedule_connection_fkey: time=229.725 calls=1813
	 Trigger for constraint passengers_schedule_passenger_fkey: time=231.568 calls=1813
	 Trigger for constraint passengers_schedule_seat_fkey: time=229.603 calls=1813
	 JIT:
	   Functions: 20
	   Options: Inlining false, Optimization false, Expressions true, Deforming true
   
# Connexion

## Query Text: 
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
## Plan :
	Index Scan using index_schedule_start_time on schedule  (cost=435.34..443.37 rows=1 width=4) (actual time=3.787..3.790 rows=1 loops=1)
	  Index Cond: (start_time = $3)
	  Filter: ((route = $2) AND ((train)::text = ($4)::text))
	  InitPlan 3 (returns $2)
	    ->  Seq Scan on routes  (cost=16.61..435.06 rows=1 width=4) (actual time=1.953..3.744 rows=1 loops=1)
	          Filter: ((start_station = $0) AND (end_station = $1))
	          Rows Removed by Filter: 19562
	          InitPlan 1 (returns $0)
	            ->  Index Scan using index_stations_name on stations  (cost=0.29..8.30 rows=1 width=4) (actual time=0.047..0.049 rows=1 loops=1)
	                  Index Cond: ((name)::text = ($1)::text)
	          InitPlan 2 (returns $1)
	            ->  Index Scan using index_stations_name on stations stations_1  (cost=0.29..8.30 rows=1 width=4) (actual time=0.035..0.036 rows=1 loops=1)
	                  Index Cond: ((name)::text = ($2)::text)
   
   
