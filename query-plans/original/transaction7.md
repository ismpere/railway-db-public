# Main Query

```sql
Query Text: /*
	* All trains produced in country X are out of order.
	* → Change train states to out_of_order, cancel connections with out_of_order.
	*
	* SCRIPT PARAMS:
	* country: the country, varchar
	*
	* EXEC SCRIPT:
	* psql -v name="'NAME'" -v country="'COUNTRY'" railway-system < transactions/transaction7.sql
	*
	* TEST VALUES:
	psql -v country="'Germany'" [DB-NAME] < transactions/transaction7.sql
	*/
```
	
## Update trains to out of order

```sql
	UPDATE trains SET train_state = 'out_of_order'
	WHERE producer IN (
		SELECT producer_id
		FROM producers
		JOIN cities ON city = city_id
		JOIN countries ON country = country_id
		WHERE countries.name = 'Germany'
	);
	Update on trains  (cost=963.45..1403.95 rows=90 width=71)
	  ->  Hash Semi Join  (cost=963.45..1403.95 rows=90 width=71)
	        Hash Cond: (trains.producer = producers.producer_id)
	        ->  Seq Scan on trains  (cost=0.00..387.00 rows=20000 width=49)
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
	                                            
## Delete connections

### Delete from passengers_schedule, for every connection with this train:

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
