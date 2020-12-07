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
