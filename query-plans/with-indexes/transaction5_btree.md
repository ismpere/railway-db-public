# Query Text: /*
	* Which trains arrive the fullest?
	* → Show all arriving trains for given station on a given day
	* with their occupancy rates
	* with highest occupancy first.
	*
	* SCRIPT PARAMS:
	* station: the end station for which this is shown, varchar
	* day: the date, date
	*
	* EXEC SCRIPT:
	* psql -v station="'STATION'" -v day="'2020-08-22'" railway-system < transactions/transaction5.sql
	*
	* TEST VALUES:
	psql -v station="'Weinsberg East'" [DB-NAME] < transactions/transaction5.sql
	
	* NOTE: We see as result= 0.00% due to of the huge number of seat in a seat => no, it was because of rounding of the division
	*/
	EXPLAIN ANALYZE
	WITH connections AS (
		SELECT * FROM schedule
		JOIN routes ON route = route_id
		WHERE end_station = (
			SELECT station_id FROM stations
			WHERE name = 'Weinsberg East'
		)
	)
	Select q1.nm as Origin, end_time::date AS arrival_date, end_time::time AS arrival_time, q2.rate from
	    (SELECT s.name as nm, sch.end_time, connections.connection_id as cnt
	        from stations as s, schedule as sch, routes as r, connections
	        where connections.connection_id =sch.connection_id and sch.route = r.route_id and r.start_station= s.station_id) as q1
	    Inner join
	        (SELECT cor.connection as cnt, format_to_percentage(cor.rate) as rate from connections ,calc_occupancy_rate(connections.connection_id)  as cor) as q2
	    ON q1.cnt=q2.cnt
	    ORDER BY q2.rate DESC;


# Plan 
	Sort  (cost=923.16..923.21 rows=20 width=60) (actual time=14797.423..14797.433 rows=13 loops=1)
	   Sort Key: ((to_char(('100'::double precision * cor.rate), '999D99%'::text))::character varying) DESC
	   Sort Method: quicksort  Memory: 26kB
	   CTE connections
	     ->  Hash Join  (cost=377.87..826.58 rows=2 width=75) (actual time=4.807..16.622 rows=13 loops=1)
		   Hash Cond: (schedule.route = routes.route_id)
		   InitPlan 1 (returns $0)
		     ->  Index Scan using index_stations_name on stations  (cost=0.29..8.30 rows=1 width=4) (actual time=0.064..0.066 rows=1 loops=1)
			   Index Cond: ((name)::text = 'Weinsberg East'::text)
		   ->  Seq Scan on schedule  (cost=0.00..397.41 rows=19541 width=55) (actual time=0.018..5.978 rows=19541 loops=1)
		   ->  Hash  (cost=369.54..369.54 rows=2 width=20) (actual time=3.757..3.757 rows=4 loops=1)
			 Buckets: 1024  Batches: 1  Memory Usage: 9kB
			 ->  Seq Scan on routes  (cost=0.00..369.54 rows=2 width=20) (actual time=0.682..3.743 rows=4 loops=1)
			       Filter: (end_station = $0)
			       Rows Removed by Filter: 19559
	   ->  Hash Join  (cost=18.31..96.15 rows=20 width=60) (actual time=1126.337..14797.305 rows=13 loops=1)
		 Hash Cond: (cor.connection = connections.connection_id)
		 ->  Nested Loop  (cost=0.25..50.29 rows=2000 width=36) (actual time=1113.517..14784.394 rows=13 loops=1)
		       ->  CTE Scan on connections connections_1  (cost=0.00..0.04 rows=2 width=4) (actual time=4.812..4.829 rows=13 loops=1)
		       ->  Function Scan on calc_occupancy_rate cor  (cost=0.25..10.25 rows=1000 width=8) (actual time=1136.823..1136.824 rows=1 loops=13)
		 ->  Hash  (cost=18.03..18.03 rows=2 width=32) (actual time=12.801..12.804 rows=13 loops=1)
		       Buckets: 1024  Batches: 1  Memory Usage: 9kB
		       ->  Nested Loop  (cost=0.86..18.03 rows=2 width=32) (actual time=0.159..12.775 rows=13 loops=1)
			     ->  Nested Loop  (cost=0.57..17.33 rows=2 width=20) (actual time=0.105..12.530 rows=13 loops=1)
				   ->  Nested Loop  (cost=0.29..16.65 rows=2 width=20) (actual time=0.041..12.305 rows=13 loops=1)
					 ->  CTE Scan on connections  (cost=0.00..0.04 rows=2 width=4) (actual time=0.002..11.850 rows=13 loops=1)
					 ->  Index Scan using schedule_pkey on schedule sch  (cost=0.29..8.30 rows=1 width=16) (actual time=0.031..0.031 rows=1 loops=13)
					       Index Cond: (connection_id = connections.connection_id)
				   ->  Index Scan using routes_pkey on routes r  (cost=0.29..0.34 rows=1 width=8) (actual time=0.014..0.014 rows=1 loops=13)
					 Index Cond: (route_id = sch.route)
			     ->  Index Scan using stations_pkey on stations s  (cost=0.29..0.35 rows=1 width=20) (actual time=0.016..0.016 rows=1 loops=13)
				   Index Cond: (station_id = r.start_station)
