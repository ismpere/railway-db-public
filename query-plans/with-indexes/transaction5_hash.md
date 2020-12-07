# Query Text:
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

	Sort  (cost=1406.53..1406.73 rows=80 width=60) (actual time=13883.769..13883.782 rows=12 loops=1)
	  Sort Key: ((to_char(('100'::double precision * cor.rate), '999D99%'::text))::character varying) DESC
	  Sort Method: quicksort  Memory: 26kB
	  CTE connections
	    ->  Hash Join  (cost=377.58..1211.42 rows=4 width=75) (actual time=5.528..24.062 rows=12 loops=1)
	          Hash Cond: (schedule.route = routes.route_id)
	          InitPlan 1 (returns $0)
	            ->  Index Scan using index_stations_name on stations  (cost=0.00..8.02 rows=1 width=4) (actual time=0.037..0.038 rows=1 loops=1)
	                  Index Cond: ((name)::text = 'Weinsberg East'::text)
	          ->  Seq Scan on schedule  (cost=0.00..740.08 rows=35708 width=55) (actual time=0.048..12.013 rows=35708 loops=1)
	          ->  Hash  (cost=369.54..369.54 rows=2 width=20) (actual time=4.003..4.004 rows=4 loops=1)
	                Buckets: 1024  Batches: 1  Memory Usage: 9kB
	                ->  Seq Scan on routes  (cost=0.00..369.54 rows=2 width=20) (actual time=0.676..3.990 rows=4 loops=1)
	                      Filter: (end_station = $0)
	                      Rows Removed by Filter: 19561
	  ->  Hash Join  (cost=36.30..192.58 rows=80 width=60) (actual time=1143.318..13883.613 rows=12 loops=1)
	        Hash Cond: (cor.connection = connections.connection_id)
	        ->  Nested Loop  (cost=0.25..100.33 rows=4000 width=36) (actual time=1118.650..13858.851 rows=12 loops=1)
	              ->  CTE Scan on connections connections_1  (cost=0.00..0.08 rows=4 width=4) (actual time=5.535..5.555 rows=12 loops=1)
	              ->  Function Scan on calc_occupancy_rate cor  (cost=0.25..10.25 rows=1000 width=8) (actual time=1154.267..1154.267 rows=1 loops=12)
	        ->  Hash  (cost=36.00..36.00 rows=4 width=32) (actual time=24.649..24.652 rows=12 loops=1)
	              Buckets: 1024  Batches: 1  Memory Usage: 9kB
	              ->  Nested Loop  (cost=0.86..36.00 rows=4 width=32) (actual time=0.776..24.616 rows=12 loops=1)
	                    ->  Nested Loop  (cost=0.58..34.61 rows=4 width=20) (actual time=0.390..23.428 rows=12 loops=1)
	                          ->  Nested Loop  (cost=0.29..33.31 rows=4 width=20) (actual time=0.033..22.439 rows=12 loops=1)
	                                ->  CTE Scan on connections  (cost=0.00..0.08 rows=4 width=4) (actual time=0.002..18.581 rows=12 loops=1)
	                                ->  Index Scan using schedule_pkey on schedule sch  (cost=0.29..8.31 rows=1 width=16) (actual time=0.316..0.316 rows=1 loops=12)
	                                      Index Cond: (connection_id = connections.connection_id)
	                          ->  Index Scan using routes_pkey on routes r  (cost=0.29..0.33 rows=1 width=8) (actual time=0.078..0.078 rows=1 loops=12)
	                                Index Cond: (route_id = sch.route)
	                    ->  Index Scan using stations_pkey on stations s  (cost=0.29..0.35 rows=1 width=20) (actual time=0.095..0.095 rows=1 loops=12)
	                          Index Cond: (station_id = r.start_station)
