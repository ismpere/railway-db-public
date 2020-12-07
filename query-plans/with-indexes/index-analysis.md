Below are the results of the experiments performed when implementing the indexes

# Transaction 1

## Times without indexes

**Min**: 0,472 s

**Max**: 6,05 s

**Average**: 1,18777777777778 s

**Median**: 0,602 s


## Times with hash index

**Min**: 0,417 s

**Max**: 0,582 s

**Average**: 0,502 s

**Median**: 0,494 s

## Times with b-tree index

**Min**: 0,415 s

**Max**: 0,739 s

**Average**: 0,538222222222222 s

**Median**: 0,57 s

## Analysis

In this query, the times are very similar despite the mean, since in the original case it is distorted due to an extreme case.

In this case the index used is index_stations_name, which is used in the transaction but does not make a big difference.

This index was not the one suggested for this transaction but postgresql has decided to use it in the filters by using that parameter as well.

Finally, there is a slight advantage of the hash-type index over the b-tree.



<!-- New Transaction -->


# Transaction 3

## Times without indexes

**Min**: 3,1 s

**Max**: 3,907 s

**Average**: 3,67877777777778 s

**Median**: 3,749 s


## Times with hash index

**Min**: 3,168 s

**Max**: 4,315 s

**Average**: 3,80444444444444 s

**Median**: 3,869 s

## Times with b-tree index

**Min**: 3,126 s

**Max**: 3,988 s

**Average**: 3,60177777777778 s

**Median**: 3,564 s

## Analysis

In this query, as in the previous one, the execution times are very similar.

In this case the index used is index_stations_name, which is used in the transaction but has not improved the times in any case.

This is due to the fact that in operations like insert, having to update the indexes increases the execution time in some queries.

Finally, there is a slight advantage of the b-tree type index over the hash type.




<!-- New Transaction -->


# Transaction 5

## Times without indexes

**Min**: 5,738 s

**Max**: 5,967 s

**Average**: 5,87122222222222 s

**Median**: 5,879 s


## Times with hash index

**Min**: 4,29 s

**Max**: 4,677 s

**Average**: 4,41666666666667 s

**Median**: 4,378 s

## Times with b-tree index

**Min**: 4,305 s

**Max**: 4,866 s

**Average**: 4,55422222222222 s

**Median**: 4,483 s

## Analysis

In this query, the execution times are better when implementing the indexes.
In this case postgresql did not use the suggested index, which was index_stations_name.
This is due to the use of other indexes implemented in the transaction subqueries.

Finally, there is a slight advantage of the hash-type index over the b-tree type.


<!-- New Transaction -->


# Transaction 7

## Times without indexes

**Min**: 1,967 s

**Max**: 2,098 s

**Average**: 2,033 s

**Median**: 2,035 s


## Times with hash index

**Min**: 2,027 s

**Max**: 2,734 s

**Average**: 2,24955555555556 s

**Median**: 2,167 s

## Times with b-tree index

**Min**: 2,05 s

**Max**: 2,777 s

**Average**: 2,29866666666667 s

**Median**: 2,105 s

## Analysis

In this query, as in the previous one, the execution times are very similar.
In this case postgresql did not use the suggested index, which was index_countries_name.

We believe that this is due to the fact that since it has to go through all the cities it is not profitable to obtain the data for each of the cities through its index.

Finally, you can see a slight advantage of the b-tree type index over the hash type.


<!-- New Transaction -->


# Transaction backup 2

## Times without indexes

**Min**: 1,045 s

**Max**: 1,197 s

**Average**: 1,07422222222222 s

**Median**: 1,056 s


## Times with hash index

**Min**: 1,1 s

**Max**: 1,394 s

**Average**: 1,17911111111111 s

**Median**: 1,119 s

## Times with b-tree index

**Min**: 1,098 s

**Max**: 1,264 s

**Average**: 1,16077777777778 s

**Median**: 1,147 s

## Analysis

In this query, as in the previous one, the execution times are very similar although they have slightly worsened with the indexes.
In this case postgresql did not use the suggested index, which was index_countries_name.

We believe that this is due to the fact that since it has to go through all the cities it is not profitable to obtain the data for each of the cities through its index.

Finally, you can see a slight advantage of the b-tree type index over the hash type.


<!-- New Transaction -->


# Transaction backup 4

## Times without indexes

**Min**: 1,661 s

**Max**: 1,926 s

**Average**: 1,73411111111111 s

**Median**: 1,726 s


## Times with hash index

**Min**: 2,229 s

**Max**: 2,966 s

**Average**: 2,42211111111111 s

**Median**: 2,32 s

## Times with b-tree index

**Min**: 2,16 s

**Max**: 2,357 s

**Average**: 2,227 s

**Median**: 2,194 s

<!-- ## Analysis -->


<!-- New Transaction -->


# Transaction backup 5

## Times without indexes

**Min**: 5,82 s

**Max**: 5,895 s

**Average**: 5,84766666666667 s

**Median**: 5,838 s


## Times with hash index

**Min**: 3,764 s

**Max**: 3,965 s

**Average**: 3,852 s

**Median**: 3,87 s

## Times with b-tree index

**Min**: 3,721 s

**Max**: 4,028 s

**Average**: 3,84033333333333 s

**Median**: 3,787 s

<!-- ## Analysis -->

