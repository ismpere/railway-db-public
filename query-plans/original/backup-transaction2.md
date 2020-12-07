# Main Query
```sql
Query Text: /*
	* A country disappears. Maybe for political reasons. Whatever.
	* As a consequence, no trains can arrive or come from there anymore.
	* → All routes and thus connections and passengers' schedules are deleted.
	*
	* SCRIPT PARAMS:
	* country: name of the country to be deleted, varchar
	*
	* EXEC SCRIPT:
	* psql -v country="'COUNTRY'" railway-system < transactions/backup_transaction2.sql
	*
	* TEST VALUES:
	psql -v country="'Poland'" [DB-NAME] < transactions/backup_transaction2.sql
	*/
	DELETE FROM countries WHERE name = 'Poland';
	Delete on countries  (cost=0.00..4.79 rows=1 width=6)
	  ->  Seq Scan on countries  (cost=0.00..4.79 rows=1 width=6)
	        Filter: ((name)::text = 'Poland'::text)
```
	        
# Dependencies

## Delete every city with that country:

```sql
Query Text: DELETE FROM ONLY "public"."cities" WHERE $1 OPERATOR(pg_catalog.=) "country"
	Delete on cities  (cost=0.00..373.00 rows=100 width=6)
	  ->  Seq Scan on cities  (cost=0.00..373.00 rows=100 width=6)
	        Filter: (74 = country)
```
	        
### Delete from passenger, for every deleted city:

```sql
Query Text: UPDATE ONLY "public"."passengers" SET "city" = NULL WHERE $1 OPERATOR(pg_catalog.=) "city"
	Update on passengers  (cost=0.00..571.00 rows=100 width=174)
	  ->  Seq Scan on passengers  (cost=0.00..571.00 rows=100 width=174)
	        Filter: ($1 = city)
```
	        
### Delete from producer, for every deleted city:

```sql
Query Text: UPDATE ONLY "public"."producers" SET "city" = NULL WHERE $1 OPERATOR(pg_catalog.=) "city"
Update on producers  (cost=0.00..554.00 rows=100 width=142)
  ->  Seq Scan on producers  (cost=0.00..554.00 rows=100 width=142)
        Filter: ($1 = city)
```
	        
### Delete station, for every deleted city:

```sql
Query Text: DELETE FROM ONLY "public"."stations" WHERE $1 OPERATOR(pg_catalog.=) "city"
	Delete on stations  (cost=0.00..402.59 rows=99 width=6)
	  ->  Seq Scan on stations  (cost=0.00..402.59 rows=99 width=6)
		    Filter: ($1 = city)
```
