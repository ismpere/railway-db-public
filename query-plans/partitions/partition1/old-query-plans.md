# Query Plan Transaction 1
## Main Query
> Insert new row into `passengers_schedule`

```sql
Insert on passengers_schedule  (cost=67.69..67.70 rows=1 width=12)

	  CTE con
	    ->  Function Scan on get_connection  
			(cost=0.25..10.25 rows=1000 width=4)

	  InitPlan 2 (returns $1)
	    ->  Seq Scan on passengers  (cost=0.00..17.18 rows=1 width=4)
	          Filter: (((surname)::text = '''Tim'''::text)
						AND ((name)::text = '''Schmidt'''::text) 
						AND ((phone_number)::text = '''4395872834'''::text))

	  InitPlan 3 (returns $2)
	    ->  CTE Scan on con  (cost=0.00..20.00 rows=1000 width=4)

	  InitPlan 5 (returns $4)
	    ->  Limit  (cost=20.25..20.27 rows=1 width=4)
	          InitPlan 4 (returns $3)
	            ->  CTE Scan on con con_1  (cost=0.00..20.00 rows=1000 width=4)
	          ->  Function Scan on get_seats_from_connection  
	          	  (cost=0.25..10.25 rows=500 width=4)
	                Filter: (NOT is_taken)

	  ->  Result  (cost=0.00..0.01 rows=1 width=12)
```

## Dependencies

> Select the connection 

```sql
Seq Scan on schedule  (cost=82.25..109.22 rows=1 width=4)

	  Filter: ((route = $2) AND (start_time = $3) AND ((train)::text = ($4)::text))

	  InitPlan 3 (returns $2)
	    ->  Seq Scan on routes  (cost=46.75..82.25 rows=1 width=4)
	          Filter: ((start_station = $0) AND (end_station = $1))

	          InitPlan 1 (returns $0)
	            ->  Seq Scan on stations  (cost=0.00..23.38 rows=5 width=4)
	                  Filter: ((name)::text = ($1)::text)

	          InitPlan 2 (returns $1)
	            ->  Seq Scan on stations stations_1  (cost=0.00..23.38 rows=5 width=4)
	                  Filter: ((name)::text = ($2)::text)
```

> Select the seat

```sql
Nested Loop  (cost=37.15..274.48 rows=11940 width=4)

	  ->  Nested Loop  (cost=37.15..100.22 rows=1990 width=36)

	        ->  Index Only Scan using trains_pkey on trains (cost=0.15..8.17 rows=1 width=32)
	              Index Cond: (train_id = ($1)::text)

	        ->  Hash Join  (cost=37.00..72.15 rows=1990 width=4)
	              Hash Cond: (seats.wagon = w1.wagon_id)
	              ->  Seq Scan on seats  (cost=0.00..29.90 rows=1990 width=8)
	              ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                    ->  Seq Scan on wagons w1  (cost=0.00..22.00 rows=1200 width=4)

	  ->  Materialize  (cost=0.00..25.03 rows=6 width=32)
	        ->  Seq Scan on wagons w2  (cost=0.00..25.00 rows=6 width=32)
	              Filter: ((train)::text = ($1)::text)
```

## Used Functions

> `get_seats_from_connection`

```sql
Result  (cost=10.25..10.26 rows=1 width=4)

	  InitPlan 2 (returns $1)
	    ->  Function Scan on get_seats_from_train  (cost=0.25..10.25 rows=1000 width=4)

	          InitPlan 1 (returns $0)
	            ->  Result  (cost=0.00..0.00 rows=0 width=32)
	                  One-Time Filter: false
```

# Query Plan Transaction 2

> Number of produced trains per producer's country

```sql
HashAggregate  (cost=10281233.30..10281235.30 rows=200 width=40)

	  Group Key: countries.name
	  ->  Hash Join  (cost=7042.76..7135433.30 rows=629160000 width=64)
	        Hash Cond: (trains.producer = p1.producer_id)
	        ->  Nested Loop  (cost=32.95..16131.49 rows=1284000 width=68)
	              ->  Hash Join  (cost=32.95..58.11 rows=1200 width=32)
	                    Hash Cond: (c2.country = countries.country_id)
	                    ->  Seq Scan on cities c2  (cost=0.00..22.00 rows=1200 width=4)
	                    ->  Hash  (cost=20.20..20.20 rows=1020 width=36)
	                          ->  Seq Scan on countries  
	                          	  (cost=0.00..20.20 rows=1020 width=36)
	                          
	              ->  Materialize  (cost=0.00..26.05 rows=1070 width=36)
	                    ->  Seq Scan on trains  (cost=0.00..20.70 rows=1070 width=36)
	                    
	        ->  Hash  (cost=3070.56..3070.56 rows=240100 width=4)
	              ->  Nested Loop  (cost=37.00..3070.56 rows=240100 width=4)
	                    ->  Hash Join  (cost=37.00..53.19 rows=490 width=0)
	                          Hash Cond: (p2.city = c1.city_id)
	                          ->  Seq Scan on producers p2  
	                          	  (cost=0.00..14.90 rows=490 width=4)
	                          ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                                ->  Seq Scan on cities c1  
	                                	(cost=0.00..22.00 rows=1200 width=4)
	                                
	                    ->  Materialize  (cost=0.00..17.35 rows=490 width=4)
	                          ->  Seq Scan on producers p1  
	                          	  (cost=0.00..14.90 rows=490 width=4)
	                          
	JIT:
	  Functions: 37
	  Options: Inlining true, Optimization true, Expressions true, Deforming true
```

# Query Plan Transaction 3

> Add connection

```sql
TODO
```

# Query Plan Transaction 4

> Create departure schedule for station

```sql
Sort  (cost=77.04..77.05 rows=5 width=44)
	  Sort Key: schedule.start_time
	  
	  InitPlan 1 (returns $0)
	    ->  Seq Scan on stations  (cost=0.00..23.38 rows=5 width=4)
	          Filter: ((name)::text = '''START'''::text)
	          
	  ->  Hash Join  (cost=31.35..53.61 rows=5 width=44)
	        Hash Cond: (schedule.route = routes.route_id)
	        ->  Seq Scan on schedule  (cost=0.00..19.70 rows=970 width=44)
	        ->  Hash  (cost=31.25..31.25 rows=8 width=8)
	              ->  Seq Scan on routes  (cost=0.00..31.25 rows=8 width=8)
	                    Filter: (start_station = $0)
```
# Query Plan Transaction 5

> Trains arriving at a station, ordered by occupancy rate

```sql
Hash Join  (cost=183.75..367.40 rows=8000 width=64)
	  Hash Cond: (schedule.route = r2.route_id)
	  
	  CTE connections
	    ->  Nested Loop  (cost=23.53..69.68 rows=1 width=4)
	          InitPlan 1 (returns $0)
	            ->  Seq Scan on stations stations_1  (cost=0.00..23.38 rows=5 width=4)
	                  Filter: ((name)::text = '''STATION'''::text)
	          ->  Seq Scan on schedule schedule_1  (cost=0.00..24.55 rows=5 width=8)
	                Filter: ((end_time)::date = '2020-08-22'::date)
	          ->  Index Scan using routes_pkey on routes  (cost=0.15..4.17 rows=1 width=4)
	                Index Cond: (route_id = schedule_1.route)
	                Filter: (end_station = $0)
	                
	  InitPlan 5 (returns $5)
	    ->  Result  (cost=10.27..10.29 rows=1 width=32)
	          InitPlan 4 (returns $4)
	            ->  Function Scan on calc_occupancy_rates calc_occupancy_rates_1  
	            	(cost=0.27..10.27 rows=1000 width=4)
	                  InitPlan 3 (returns $3)
	                    ->  CTE Scan on connections  (cost=0.00..0.02 rows=1 width=4)
	                    
	  InitPlan 6 (returns $6)
	    ->  CTE Scan on connections connections_1  (cost=0.00..0.02 rows=1 width=4)
	    
	  ->  Hash Join  (cost=55.51..218.11 rows=8000 width=36)
	        Hash Cond: (calc_occupancy_rates.connection = schedule.connection_id)
	        ->  Nested Loop  (cost=23.69..165.19 rows=8000 width=36)
	              ->  Function Scan on calc_occupancy_rates  
	              	  (cost=0.25..10.25 rows=1000 width=4)
	              ->  Materialize  (cost=23.44..54.96 rows=8 width=32)
	                    ->  Hash Join  (cost=23.44..54.92 rows=8 width=32)
	                          Hash Cond: (r1.start_station = stations.station_id)
	                          ->  Seq Scan on routes r1  
	                          	  (cost=0.00..27.00 rows=1700 width=4)
	                          ->  Hash  (cost=23.38..23.38 rows=5 width=36)
	                                ->  Seq Scan on stations  
	                                	(cost=0.00..23.38 rows=5 width=36)
	                                      Filter: ((name)::text = '''STATION'''::text)
	                                      
	        ->  Hash  (cost=19.70..19.70 rows=970 width=8)
	              ->  Seq Scan on schedule  (cost=0.00..19.70 rows=970 width=8)
	              
	  ->  Hash  (cost=27.00..27.00 rows=1700 width=4)
	        Buckets: 2048  Batches: 1  Memory Usage: 16kB
	        ->  Seq Scan on routes r2  (cost=0.00..27.00 rows=1700 width=4)
```

# Query Plan Transaction 6

> Delete passenger schedules with country and date

```sql
Delete on passengers_schedule  (cost=2212647.95..2212688.55 rows=20 width=6)

	  InitPlan 2 (returns $2)
	    ->  Hash Join  (cost=832.84..1106323.98 rows=92769000 width=4)
	          Hash Cond: (passengers_schedule_1.connection = s1.connection_id)
	          InitPlan 1 (returns $0)
	            ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	          ->  Nested Loop  (cost=34.08..43450.21 rows=3468000 width=4)
	                ->  Seq Scan on passengers_schedule passengers_schedule_1  
	                	(cost=0.00..30.40 rows=2040 width=4)
	                ->  Materialize  (cost=34.08..74.06 rows=1700 width=0)
	                      ->  Hash Join  (cost=34.08..65.56 rows=1700 width=0)
	                            Hash Cond: (r2.start_station = st1.station_id)
	                            ->  Seq Scan on routes r2  
	                            	(cost=0.00..27.00 rows=1700 width=4)
	                            ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                                  ->  Seq Scan on stations st1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  
	          ->  Hash  (cost=448.64..448.64 rows=26190 width=4)
	                ->  Nested Loop  (cost=71.17..448.64 rows=26190 width=4)
	                      ->  Seq Scan on schedule s1  (cost=0.00..19.70 rows=970 width=4)
	                      ->  Materialize  (cost=71.17..101.63 rows=27 width=0)
	                            ->  Hash Join  (cost=71.17..101.50 rows=27 width=0)
	                                  Hash Cond: (st2.city = c1.city_id)
	                                  ->  Seq Scan on stations st2  
	                                  	  (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=70.79..70.79 rows=30 width=4)
	                                        ->  Nested Loop  
	                                        	(cost=0.15..70.79 rows=30 width=4)
	                                              ->  Seq Scan on cities c1  
	                                                  (cost=0.00..25.00 rows=6 width=4)
	                                                    Filter: (country = $0)
	                                              ->  Materialize  
	                                                  (cost=0.15..45.43 rows=5 width=0)
	                                                    ->  Nested Loop  
	                                                        (cost=0.15..45.40 
	                                                         rows=5 width=0)
	                                                          ->  Seq Scan 
	                                                          	  on schedule s2  
	                                                          	  (cost=0.00..24.55 
	                                                          	   rows=5 width=4)
	                                                                Filter: 
	                                                                ((start_time)::date = 
	                                                                '2020-08-22'::date)
	                                                          ->  Index Only Scan 
	                                                              using routes_pkey 
	                                                              on routes r1  
	                                                              (cost=0.15..4.17 
	                                                              rows=1 width=4)
	                                                                Index Cond: 
	                                                                (route_id = s2.route)
	                                                                
	  InitPlan 4 (returns $5)
	    ->  Hash Join  (cost=832.84..1106323.98 rows=92769000 width=4)
	          Hash Cond: (passengers_schedule_2.connection = s1_1.connection_id)
	          
	          InitPlan 3 (returns $3)
	            ->  Seq Scan on countries countries_1  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=34.08..43450.21 rows=3468000 width=4)
	                ->  Seq Scan on passengers_schedule passengers_schedule_2  
	                    (cost=0.00..30.40 rows=2040 width=4)
	                ->  Materialize  (cost=34.08..74.06 rows=1700 width=0)
	                      ->  Hash Join  (cost=34.08..65.56 rows=1700 width=0)
	                            Hash Cond: (r2_1.end_station = st1_1.station_id)
	                            ->  Seq Scan on routes r2_1  
	                                (cost=0.00..27.00 rows=1700 width=4)
	                            ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                                  ->  Seq Scan on stations st1_1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  
	          ->  Hash  (cost=448.64..448.64 rows=26190 width=4)
	                ->  Nested Loop  (cost=71.17..448.64 rows=26190 width=4)
	                      ->  Seq Scan on schedule s1_1  
	                          (cost=0.00..19.70 rows=970 width=4)
	                      ->  Materialize  (cost=71.17..101.63 rows=27 width=0)
	                            ->  Hash Join  (cost=71.17..101.50 rows=27 width=0)
	                                  Hash Cond: (st2_1.city = c1_1.city_id)
	                                  ->  Seq Scan on stations st2_1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=70.79..70.79 rows=30 width=4)
	                                        ->  Nested Loop  
	                                            (cost=0.15..70.79 rows=30 width=4)
	                                              ->  Seq Scan on cities c1_1  
	                                                  (cost=0.00..25.00 rows=6 width=4)
	                                                    Filter: (country = $3)   
	                                              ->  Materialize  
	                                                 (cost=0.15..45.43 rows=5 width=0)
	                                                    ->  Nested Loop  
	                                                        (cost=0.15..45.40 
	                                                         rows=5 width=0)
	                                                          ->  Seq Scan 
	                                                              on schedule s2_1  
	                                                              (cost=0.00..24.55 
	                                                               rows=5 width=4)
	                                                                Filter: 
	                                                                ((start_time)::date = 
	                                                                '2020-08-22'::date)
	                                                          ->  Index Only Scan 
	                                                              using routes_pkey 
	                                                              on routes r1_1  
	                                                              (cost=0.15..4.17 
	                                                               rows=1 width=4)
	                                                                Index Cond: 
	                                                                (route_id = 
	                                                                s2_1.route)
	                                                                
	  ->  Seq Scan on passengers_schedule  (cost=0.00..40.60 rows=20 width=6)
	        Filter: ((connection = $2) OR (connection = $5))
	        
	JIT:
	  Functions: 79
	  Options: Inlining true, Optimization true, Expressions true, Deforming true
```

> Delete connections with country and date

```sql
Delete on schedule  (cost=1768.53..1773.88 rows=2 width=6)

	  InitPlan 2 (returns $2)
	    ->  Hash Join  (cost=127.99..880.11 rows=45475 width=4)
	          Hash Cond: (r2.start_station = st1.station_id)
	          
	          InitPlan 1 (returns $0)
	            ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=71.17..702.32 rows=45900 width=8)
	                ->  Seq Scan on routes r2  (cost=0.00..27.00 rows=1700 width=4)
	                ->  Materialize  (cost=71.17..101.63 rows=27 width=4)
	                      ->  Hash Join  (cost=71.17..101.50 rows=27 width=4)
	                            Hash Cond: (st2.city = c1.city_id)
	                            ->  Seq Scan on stations st2  
	                                (cost=0.00..20.70 rows=1070 width=4)
	                            ->  Hash  (cost=70.79..70.79 rows=30 width=8)
	                                  ->  Nested Loop  (cost=0.15..70.79 rows=30 width=8)
	                                        ->  Seq Scan on cities c1  
	                                            (cost=0.00..25.00 rows=6 width=4)
	                                              Filter: (country = $0)
	                                              
	                                        ->  Materialize  
	                                            (cost=0.15..45.43 rows=5 width=4)
	                                              ->  Nested Loop  
	                                                  (cost=0.15..45.40 rows=5 width=4)
	                                                    ->  Seq Scan 
	                                                        on schedule schedule_1  
	                                                        (cost=0.00..24.55 
	                                                         rows=5 width=8)
	                                                          Filter: 
	                                                          ((start_time)::date = 
	                                                           '2020-08-22'::date)
	                                                    ->  Index Only Scan 
	                                                        using routes_pkey 
	                                                        on routes r1  
	                                                        (cost=0.15..4.17 
	                                                         rows=1 width=4)
	                                                          Index Cond: 
	                                                          (route_id = 
	                                                           schedule_1.route)
	                                                          
	          ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                Buckets: 2048  Batches: 1  Memory Usage: 16kB
	                ->  Seq Scan on stations st1  (cost=0.00..20.70 rows=1070 width=4)
	                
	  InitPlan 4 (returns $5)
	    ->  Hash Join  (cost=127.99..880.11 rows=45475 width=4)
	          Hash Cond: (r2_1.end_station = st1_1.station_id)
	          InitPlan 3 (returns $3)
	            ->  Seq Scan on countries countries_1  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=71.17..702.32 rows=45900 width=8)
	                ->  Seq Scan on routes r2_1  (cost=0.00..27.00 rows=1700 width=4)
	                ->  Materialize  (cost=71.17..101.63 rows=27 width=4)
	                      ->  Hash Join  (cost=71.17..101.50 rows=27 width=4)
	                            Hash Cond: (st2_1.city = c1_1.city_id)
	                            ->  Seq Scan on stations st2_1  
	                                (cost=0.00..20.70 rows=1070 width=4)
	                            ->  Hash  (cost=70.79..70.79 rows=30 width=8)
	                                  ->  Nested Loop  (cost=0.15..70.79 rows=30 width=8)
	                                        ->  Seq Scan on cities c1_1  
	                                            (cost=0.00..25.00 rows=6 width=4)
	                                              Filter: (country = $3)
	                                        ->  Materialize  
	                                            (cost=0.15..45.43 rows=5 width=4)
	                                              ->  Nested Loop  
	                                                  (cost=0.15..45.40 rows=5 width=4)
	                                                    ->  Seq Scan 
	                                                        on schedule schedule_2  
	                                                        (cost=0.00..24.55 
	                                                        rows=5 width=8)
	                                                          Filter: 
	                                                          ((start_time)::date = 
	                                                          '2020-08-22'::date)
	                                                    ->  Index Only Scan 
	                                                    	using routes_pkey 
	                                                    	on routes r1_1  
	                                                    	(cost=0.15..4.17 
	                                                    	 rows=1 width=4)
	                                                          Index Cond: 
	                                                          (route_id = 
	                                                           schedule_2.route)
	                                                          
	          ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                Buckets: 2048  Batches: 1  Memory Usage: 16kB
	                ->  Seq Scan on stations st1_1  (cost=0.00..20.70 rows=1070 width=4)
	                
	  ->  Bitmap Heap Scan on schedule  (cost=8.32..13.66 rows=2 width=6)
	        Recheck Cond: ((connection_id = $2) OR (connection_id = $5))
	        ->  BitmapOr  (cost=8.32..8.32 rows=2 width=0)
	              ->  Bitmap Index Scan on schedule_pkey  (cost=0.00..4.16 rows=1 width=0)
	                    Index Cond: (connection_id = $2)
	              ->  Bitmap Index Scan on schedule_pkey  (cost=0.00..4.16 rows=1 width=0)
	                    Index Cond: (connection_id = $5)
```

# Query Plan Transaction 7

> Set trains to be out of order

```sql
Update on trains  (cost=79.60..102.98 rows=5 width=54)

	  InitPlan 1 (returns $1)
	    ->  Hash Join  (cost=51.48..79.60 rows=12 width=4)
	          Hash Cond: (c2.country = countries.country_id)
	          ->  Seq Scan on cities c2  (cost=0.00..22.00 rows=1200 width=4)
	          ->  Hash  (cost=51.36..51.36 rows=10 width=8)
	                ->  Nested Loop  (cost=0.15..51.36 rows=10 width=8)
	                      ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                            Filter: ((name)::text = '''COUNTRY'''::text)
	                      ->  Materialize  (cost=0.15..28.49 rows=2 width=4)
	                            ->  Nested Loop  (cost=0.15..28.48 rows=2 width=4)
	                                  ->  Seq Scan on producers  
	                                      (cost=0.00..16.12 rows=2 width=8)
	                                        Filter: ((name)::text = '''NAME'''::text)
	                                  ->  Index Only Scan using cities_pkey 
	                                      on cities c1  
	                                      (cost=0.15..6.17 rows=1 width=4)
	                                        Index Cond: (city_id = producers.city)
	                                        
	  ->  Seq Scan on trains  (cost=0.00..23.38 rows=5 width=54)
	        Filter: (producer = $1)
```	

> Delete passenger schedules with trains out of order

```sql
Delete on passengers_schedule  (cost=2212647.95..2212688.55 rows=20 width=6)

	  InitPlan 2 (returns $2)
	    ->  Hash Join  (cost=832.84..1106323.98 rows=92769000 width=4)
	          Hash Cond: (passengers_schedule_1.connection = s1.connection_id)
	          InitPlan 1 (returns $0)
	            ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=34.08..43450.21 rows=3468000 width=4)
	                ->  Seq Scan on passengers_schedule passengers_schedule_1  
	                    (cost=0.00..30.40 rows=2040 width=4)
	                ->  Materialize  (cost=34.08..74.06 rows=1700 width=0)
	                      ->  Hash Join  (cost=34.08..65.56 rows=1700 width=0)
	                            Hash Cond: (r2.start_station = st1.station_id)
	                            ->  Seq Scan on routes r2  
	                                (cost=0.00..27.00 rows=1700 width=4)
	                            ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                                  ->  Seq Scan on stations st1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  
	          ->  Hash  (cost=448.64..448.64 rows=26190 width=4)
	                ->  Nested Loop  (cost=71.17..448.64 rows=26190 width=4)
	                      ->  Seq Scan on schedule s1  (cost=0.00..19.70 rows=970 width=4)
	                      ->  Materialize  (cost=71.17..101.63 rows=27 width=0)
	                            ->  Hash Join  (cost=71.17..101.50 rows=27 width=0)
	                                  Hash Cond: (st2.city = c1.city_id)
	                                  ->  Seq Scan on stations st2  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=70.79..70.79 rows=30 width=4)
	                                        ->  Nested Loop  
	                                            (cost=0.15..70.79 rows=30 width=4)
	                                              ->  Seq Scan on cities c1  
	                                                  (cost=0.00..25.00 rows=6 width=4)
	                                                    Filter: (country = $0)
	                                              ->  Materialize  
	                                                  (cost=0.15..45.43 rows=5 width=0)
	                                                    ->  Nested Loop  
	                                                        (cost=0.15..45.40 
	                                                         rows=5 width=0)
	                                                          ->  Seq Scan 
	                                                              on schedule s2  
	                                                              (cost=0.00..24.55 
	                                                               rows=5 width=4)
	                                                                Filter: 
	                                                                ((start_time)::date = 
	                                                                '2020-08-22'::date)
	                                                          ->  Index Only Scan 
	                                                              using routes_pkey 
	                                                              on routes r1  
	                                                              (cost=0.15..4.17 
	                                                               rows=1 width=4)
	                                                                Index Cond: 
	                                                                (route_id = s2.route)
	                                                                
	  InitPlan 4 (returns $5)
	    ->  Hash Join  (cost=832.84..1106323.98 rows=92769000 width=4)
	          Hash Cond: (passengers_schedule_2.connection = s1_1.connection_id)
	          InitPlan 3 (returns $3)
	            ->  Seq Scan on countries countries_1  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=34.08..43450.21 rows=3468000 width=4)
	                ->  Seq Scan on passengers_schedule passengers_schedule_2  
	                	(cost=0.00..30.40 rows=2040 width=4)
	                ->  Materialize  (cost=34.08..74.06 rows=1700 width=0)
	                      ->  Hash Join  (cost=34.08..65.56 rows=1700 width=0)
	                            Hash Cond: (r2_1.end_station = st1_1.station_id)
	                            ->  Seq Scan on routes r2_1  
	                                (cost=0.00..27.00 rows=1700 width=4)
	                            ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                                  ->  Seq Scan on stations st1_1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  
	          ->  Hash  (cost=448.64..448.64 rows=26190 width=4)
	                ->  Nested Loop  (cost=71.17..448.64 rows=26190 width=4)
	                      ->  Seq Scan on schedule s1_1  
	                          (cost=0.00..19.70 rows=970 width=4)
	                      ->  Materialize  (cost=71.17..101.63 rows=27 width=0)
	                            ->  Hash Join  (cost=71.17..101.50 rows=27 width=0)
	                                  Hash Cond: (st2_1.city = c1_1.city_id)
	                                  ->  Seq Scan on stations st2_1  
	                                      (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=70.79..70.79 rows=30 width=4)
	                                        ->  Nested Loop  
	                                            (cost=0.15..70.79 rows=30 width=4)
	                                              ->  Seq Scan on cities c1_1  
	                                                  (cost=0.00..25.00 rows=6 width=4)
	                                                    Filter: (country = $3)
	                                              ->  Materialize  
	                                                 (cost=0.15..45.43 rows=5 width=0)
	                                                    ->  Nested Loop  
	                                                        (cost=0.15..45.40
	                                                         rows=5 width=0)
	                                                          ->  Seq Scan on schedule s2_1  (cost=0.00..24.55 rows=5 width=4)
	                                                                Filter: ((start_time)::date = '2020-08-22'::date)
	                                                          ->  Index Only Scan using routes_pkey on routes r1_1  (cost=0.15..4.17 rows=1 width=4)
	                                                                Index Cond: (route_id = s2_1.route)
	                                                                
	  ->  Seq Scan on passengers_schedule  (cost=0.00..40.60 rows=20 width=6)
	        Filter: ((connection = $2) OR (connection = $5))
	        
	JIT:
	  Functions: 79
	  Options: Inlining true, Optimization true, Expressions true, Deforming true
```      

> Delete connections with out of order trains

```sql
Delete on schedule  (cost=1768.53..1773.88 rows=2 width=6)

	  InitPlan 2 (returns $2)
	    ->  Hash Join  (cost=127.99..880.11 rows=45475 width=4)
	          Hash Cond: (r2.start_station = st1.station_id)
	          InitPlan 1 (returns $0)
	            ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=71.17..702.32 rows=45900 width=8)
	                ->  Seq Scan on routes r2  (cost=0.00..27.00 rows=1700 width=4)
	                ->  Materialize  (cost=71.17..101.63 rows=27 width=4)
	                      ->  Hash Join  (cost=71.17..101.50 rows=27 width=4)
	                            Hash Cond: (st2.city = c1.city_id)
	                            ->  Seq Scan on stations st2  (cost=0.00..20.70 rows=1070 width=4)
	                            ->  Hash  (cost=70.79..70.79 rows=30 width=8)
	                                  ->  Nested Loop  (cost=0.15..70.79 rows=30 width=8)
	                                        ->  Seq Scan on cities c1  (cost=0.00..25.00 rows=6 width=4)
	                                              Filter: (country = $0)
	                                        ->  Materialize  (cost=0.15..45.43 rows=5 width=4)
	                                              ->  Nested Loop  (cost=0.15..45.40 rows=5 width=4)
	                                                    ->  Seq Scan on schedule schedule_1  (cost=0.00..24.55 rows=5 width=8)
	                                                          Filter: ((start_time)::date = '2020-08-22'::date)
	                                                    ->  Index Only Scan using routes_pkey on routes r1  (cost=0.15..4.17 rows=1 width=4)
	                                                          Index Cond: (route_id = schedule_1.route)
	                                                          
	          ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                Buckets: 2048  Batches: 1  Memory Usage: 16kB
	                ->  Seq Scan on stations st1  (cost=0.00..20.70 rows=1070 width=4)
	                
	  InitPlan 4 (returns $5)
	    ->  Hash Join  (cost=127.99..880.11 rows=45475 width=4)
	          Hash Cond: (r2_1.end_station = st1_1.station_id)
	          InitPlan 3 (returns $3)
	            ->  Seq Scan on countries countries_1  (cost=0.00..22.75 rows=5 width=4)
	                  Filter: ((name)::text = '''COUNTRY'''::text)
	                  
	          ->  Nested Loop  (cost=71.17..702.32 rows=45900 width=8)
	                ->  Seq Scan on routes r2_1  (cost=0.00..27.00 rows=1700 width=4)
	                ->  Materialize  (cost=71.17..101.63 rows=27 width=4)
	                      ->  Hash Join  (cost=71.17..101.50 rows=27 width=4)
	                            Hash Cond: (st2_1.city = c1_1.city_id)
	                            ->  Seq Scan on stations st2_1  (cost=0.00..20.70 rows=1070 width=4)
	                            ->  Hash  (cost=70.79..70.79 rows=30 width=8)
	                                  ->  Nested Loop  (cost=0.15..70.79 rows=30 width=8)
	                                        ->  Seq Scan on cities c1_1  (cost=0.00..25.00 rows=6 width=4)
	                                              Filter: (country = $3)
	                                        ->  Materialize  (cost=0.15..45.43 rows=5 width=4)
	                                              ->  Nested Loop  (cost=0.15..45.40 rows=5 width=4)
	                                                    ->  Seq Scan on schedule schedule_2  (cost=0.00..24.55 rows=5 width=8)
	                                                          Filter: ((start_time)::date = '2020-08-22'::date)
	                                                    ->  Index Only Scan using routes_pkey on routes r1_1  (cost=0.15..4.17 rows=1 width=4)
	                                                          Index Cond: (route_id = schedule_2.route)
	                                                          
	          ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                Buckets: 2048  Batches: 1  Memory Usage: 16kB
	                ->  Seq Scan on stations st1_1  (cost=0.00..20.70 rows=1070 width=4)
	                
	  ->  Bitmap Heap Scan on schedule  (cost=8.32..13.66 rows=2 width=6)
	        Recheck Cond: ((connection_id = $2) OR (connection_id = $5))
	        ->  BitmapOr  (cost=8.32..8.32 rows=2 width=0)
	              ->  Bitmap Index Scan on schedule_pkey  (cost=0.00..4.16 rows=1 width=0)
	                    Index Cond: (connection_id = $2)
	              ->  Bitmap Index Scan on schedule_pkey  (cost=0.00..4.16 rows=1 width=0)
	                    Index Cond: (connection_id = $5)
```  

# Query Plan Backup Transaction 1

> Select passenger names from passengers_schedule, grouped by year and first name

```sql
GroupAggregate  (cost=239.88..262.78 rows=200 width=104)
	  Group Key: (ROW(date_part('year'::text, $0))), passengers.name
	  
	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=31.83..67.60 rows=2040 width=8)
	          Hash Cond: (passengers_schedule_1.connection = schedule.connection_id)
	          ->  Seq Scan on passengers_schedule passengers_schedule_1  (cost=0.00..30.40 rows=2040 width=4)
	          ->  Hash  (cost=19.70..19.70 rows=970 width=12)
	                ->  Seq Scan on schedule  (cost=0.00..19.70 rows=970 width=12)
	                
	  ->  Sort  (cost=172.28..177.38 rows=2040 width=64)
	        Sort Key: passengers.name
	        ->  Hash Join  (cost=19.23..60.14 rows=2040 width=64)
	              Hash Cond: (passengers_schedule.passenger = passengers.passenger_id)
	              ->  Seq Scan on passengers_schedule  (cost=0.00..30.40 rows=2040 width=4)
	              ->  Hash  (cost=14.10..14.10 rows=410 width=36)
	                    Buckets: 1024  Batches: 1  Memory Usage: 8kB
	                    ->  Seq Scan on passengers  (cost=0.00..14.10 rows=410 width=36)
```

# Query Plan Backup Transaction 2

> Delete passenger schedules with country

```sql
Delete on passengers_schedule  (cost=202891772015.13..202891772050.63 rows=10 width=6)

	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=30229.52..202891772015.13 rows=17997186000000 width=4)
	          Hash Cond: (r2.start_station = s1.station_id)
	          ->  Nested Loop  (cost=93.89..232059260.49 rows=18553800000 width=4)
	                ->  Nested Loop  (cost=93.89..136729.24 rows=10914000 width=0)
	                      ->  Nested Loop  (cost=56.89..241.05 rows=10200 width=0)
	                            ->  Hash Join  (cost=34.08..65.56 rows=1700 width=0)
	                                  Hash Cond: (r3.end_station = s2.station_id)
	                                  ->  Seq Scan on routes r3  (cost=0.00..27.00 rows=1700 width=4)
	                                  ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                                        ->  Seq Scan on stations s2  (cost=0.00..20.70 rows=1070 width=4)
	                                        
	                            ->  Materialize  (cost=22.81..48.00 rows=6 width=0)
	                                  ->  Hash Join  (cost=22.81..47.97 rows=6 width=0)
	                                        Hash Cond: (c2.country = countries.country_id)
	                                        ->  Seq Scan on cities c2  (cost=0.00..22.00 rows=1200 width=4)
	                                        ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                                              ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                                                    Filter: ((name)::text = '''COUNTRY'''::text)
	                                                    
	                      ->  Materialize  (cost=37.00..65.87 rows=1070 width=0)
	                            ->  Hash Join  (cost=37.00..60.52 rows=1070 width=0)
	                                  Hash Cond: (s3.city = c1.city_id)
	                                  ->  Seq Scan on stations s3  (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                                        ->  Seq Scan on cities c1  (cost=0.00..22.00 rows=1200 width=4)
	                                        
	                ->  Materialize  (cost=0.00..35.50 rows=1700 width=4)
	                      ->  Seq Scan on routes r2  (cost=0.00..27.00 rows=1700 width=4)
	                      
	          ->  Hash  (cost=13106.88..13106.88 rows=1037900 width=8)
	                ->  Nested Loop  (cost=48.40..13106.88 rows=1037900 width=8)
	                      ->  Index Only Scan using stations_pkey on stations s1  (cost=0.15..60.20 rows=1070 width=4)
	                      ->  Materialize  (cost=48.25..75.36 rows=970 width=4)
	                            ->  Hash Join  (cost=48.25..70.51 rows=970 width=4)
	                                  Hash Cond: (schedule.route = r1.route_id)
	                                  ->  Seq Scan on schedule  (cost=0.00..19.70 rows=970 width=8)
	                                  ->  Hash  (cost=27.00..27.00 rows=1700 width=4)
	                                        ->  Seq Scan on routes r1  (cost=0.00..27.00 rows=1700 width=4)
	                                        
	  ->  Seq Scan on passengers_schedule  (cost=0.00..35.50 rows=10 width=6)
	        Filter: (connection = $0)
	        
	JIT:
	  Functions: 56
	  Options: Inlining true, Optimization true, Expressions true, Deforming true
```

> Delete connections with country

```sql
Delete on schedule  (cost=205063300.60..205063322.72 rows=5 width=6)

	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=33275.78..205063300.60 rows=18190000000 width=4)
	          Hash Cond: (r1.start_station = s1.station_id)
	          ->  Nested Loop  (cost=56.89..212794.70 rows=17000000 width=8)
	                ->  Hash Join  (cost=56.89..263.45 rows=10000 width=0)
	                      Hash Cond: (r2.end_station = s2.station_id)
	                      ->  Nested Loop  (cost=22.81..202.49 rows=10200 width=4)
	                            ->  Seq Scan on routes r2  (cost=0.00..27.00 rows=1700 width=4)
	                            ->  Materialize  (cost=22.81..48.00 rows=6 width=0)
	                                  ->  Hash Join  (cost=22.81..47.97 rows=6 width=0)
	                                        Hash Cond: (c2.country = countries.country_id)
	                                        ->  Seq Scan on cities c2  (cost=0.00..22.00 rows=1200 width=4)
	                                        ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                                              ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                                                    Filter: ((name)::text = '''COUNTRY'''::text)
	                                                    
	                      ->  Hash  (cost=20.70..20.70 rows=1070 width=4)
	                            ->  Seq Scan on stations s2  (cost=0.00..20.70 rows=1070 width=4)
	                            
	                ->  Materialize  (cost=0.00..35.50 rows=1700 width=8)
	                      ->  Seq Scan on routes r1  (cost=0.00..27.00 rows=1700 width=8)
	                      
	          ->  Hash  (cost=14434.65..14434.65 rows=1144900 width=4)
	                ->  Nested Loop  (cost=37.15..14434.65 rows=1144900 width=4)
	                      ->  Index Only Scan using stations_pkey on stations s1  (cost=0.15..60.20 rows=1070 width=4)
	                      ->  Materialize  (cost=37.00..65.87 rows=1070 width=0)
	                            ->  Hash Join  (cost=37.00..60.52 rows=1070 width=0)
	                                  Hash Cond: (s3.city = c1.city_id)
	                                  ->  Seq Scan on stations s3  (cost=0.00..20.70 rows=1070 width=4)
	                                  ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                                        ->  Seq Scan on cities c1  (cost=0.00..22.00 rows=1200 width=4)
	                                        
	  ->  Seq Scan on schedule  (cost=0.00..22.12 rows=5 width=6)
	        Filter: (route = $0)
	        
	JIT:
	  Functions: 42
	  Options: Inlining true, Optimization true, Expressions true, Deforming true
```

> Delete routes with country

```sql
Delete on routes  (cost=405.69..441.19 rows=17 width=6)

	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=59.81..202.85 rows=6294 width=4)
	          Hash Cond: (stations.city = c1.city_id)
	          ->  Nested Loop  (cost=22.81..148.94 rows=6420 width=8)
	                ->  Seq Scan on stations  (cost=0.00..20.70 rows=1070 width=8)
	                ->  Materialize  (cost=22.81..48.00 rows=6 width=0)
	                      ->  Hash Join  (cost=22.81..47.97 rows=6 width=0)
	                            Hash Cond: (c2.country = countries.country_id)
	                            ->  Seq Scan on cities c2  (cost=0.00..22.00 rows=1200 width=4)
	                            ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                                  ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                                        Filter: ((name)::text = '''COUNTRY'''::text)
	                                        
	          ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                ->  Seq Scan on cities c1  (cost=0.00..22.00 rows=1200 width=4)
	                
	  InitPlan 2 (returns $1)
	    ->  Hash Join  (cost=59.81..202.85 rows=6294 width=4)
	          Hash Cond: (stations_1.city = c1_1.city_id)
	          ->  Nested Loop  (cost=22.81..148.94 rows=6420 width=8)
	                ->  Seq Scan on stations stations_1  (cost=0.00..20.70 rows=1070 width=8)
	                ->  Materialize  (cost=22.81..48.00 rows=6 width=0)
	                      ->  Hash Join  (cost=22.81..47.97 rows=6 width=0)
	                            Hash Cond: (c2_1.country = countries_1.country_id)
	                            ->  Seq Scan on cities c2_1  (cost=0.00..22.00 rows=1200 width=4)
	                            ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                                  ->  Seq Scan on countries countries_1  (cost=0.00..22.75 rows=5 width=4)
	                                        Filter: ((name)::text = '''COUNTRY'''::text)
	                                        
	          ->  Hash  (cost=22.00..22.00 rows=1200 width=4)
	                ->  Seq Scan on cities c1_1  (cost=0.00..22.00 rows=1200 width=4)
	                
	  ->  Seq Scan on routes  (cost=0.00..35.50 rows=17 width=6)
	        Filter: ((start_station = $0) OR (end_station = $1))
```

> Delete stations that are in the country

```sql
Delete on stations  (cost=47.97..71.35 rows=5 width=6)

	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=22.81..47.97 rows=6 width=4)
	          Hash Cond: (cities.country = countries.country_id)
	          ->  Seq Scan on cities  (cost=0.00..22.00 rows=1200 width=8)
	          ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                      Filter: ((name)::text = '''COUNTRY'''::text)
	                      
	  ->  Seq Scan on stations  (cost=0.00..23.38 rows=5 width=6)
	        Filter: (city = $0)
```

> Delete producers with address in the country

```sql
Delete on producers  (cost=47.97..64.10 rows=2 width=6)

	  InitPlan 1 (returns $0)
	    ->  Hash Join  (cost=22.81..47.97 rows=6 width=4)
	          Hash Cond: (cities.country = countries.country_id)
	          ->  Seq Scan on cities  (cost=0.00..22.00 rows=1200 width=8)
	          ->  Hash  (cost=22.75..22.75 rows=5 width=4)
	                ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	                      Filter: ((name)::text = '''COUNTRY'''::text)
	                      
	  ->  Seq Scan on producers  (cost=0.00..16.12 rows=2 width=6)
	        Filter: (city = $0)
```

> Delete cities from country

```sql
Delete on cities  (cost=22.75..47.75 rows=6 width=6)

	  InitPlan 1 (returns $0)
	    ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=4)
	          Filter: ((name)::text = '''COUNTRY'''::text)

	  ->  Seq Scan on cities  (cost=0.00..25.00 rows=6 width=6)
	        Filter: (country = $0)
```

> Delete country

```sql
Delete on countries  (cost=0.00..22.75 rows=5 width=6)

	  ->  Seq Scan on countries  (cost=0.00..22.75 rows=5 width=6)
	        Filter: ((name)::text = '''COUNTRY'''::text)
```
