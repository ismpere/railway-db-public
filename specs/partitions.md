# Partition Specification

## Partition 1

**What:** Partition table cities by country.
**Type:** value list
**Used in:**

- [in transaction 3 when we query data from the cities table for insertion](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#create-stations)

- [in transaction 7 when deleting connections by train producer's country](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction7.md#delete-from-passengers_schedule-for-every-connection-with-this-train)



- [in backup transaction 2 when we are deleting by country (because for most tables, they find out the country via cities](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction2.md#dependencies)

- [in backup transaction 5 we join with cities and also filter by country which again needs city](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction5.md)

## Partition 2

**What:** Partition table trains by train_type.
**Type:** value list
**Used in:**

- [in transaction 1 where available seats are fetched by train](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#prerequisites)

- in transaction 3 in [join statement](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#join-routes-and-train-for-every-starting-time) and [inserting new connections](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#insert-connections)

## Partition 3

**What:** Partition table seats by seat_id
**Type:** range
**Used in:**

- [in backup transaction 4](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction4.md)
- [fetching seats in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#getting-the-seats)
- [assigning seats to passengers in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#cross-joining-passengers-with-connection-and-their-seat-for-every-passenger)
- [in the main query of transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#main-query)
- [calculating occupancy rates of the seats in transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#prerequisites)

## Partition 4

**What:** Partition wagons by wagon_id
**Type:** range
**Used in:**

- [in backup transaction 4](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction4.md)
- [getting seats in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#getting-the-seats)
- [calculating occupancy rate in transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#calculate-occupancy-rate)

## Partition 5

**What:** Partition passengers_schedule by hash of all 3 foreign keys
**Type:** hash
**Used in:**

- [checking for free seats in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#for-every-seat)
- [insert into passengers_schedule in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#main-query)
- [checking if seats are occupied in transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#for-every-connection)
- [deleting from passengers_schedule in transaction 7](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction7.md#delete-from-passengers_schedule-for-every-connection-with-this-train)

## Partition 6

**What:** Partition schedule by connection_id
**Type:** hash
**Used in:**

- [matching connection with passengers in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#cross-joining-passengers-with-connection-and-their-seat-for-every-passenger)
- [insert into schedule in transaction 3](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#insert-connections)
- [in transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#main-query)
- [deleting connections in transaction 7](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction7.md#delete-from-passengers_schedule-for-every-connection-with-this-train)

## Partition 7

**What:** Partition routes by route_id
**Type:** hash
**Used in:**

- [in transaction 1](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction1.md#main-query)
- [join](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#join-routes-and-train-for-every-starting-time) in transaction 3
- [creating routes in transaction 3](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#create-routes)
- [inserting connections in transaction 3](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction3.md#insert-connections)
- [in transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#main-query)
- [in the prerequisites for transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/transaction5.md#for-every-connection)
- [in backup transaction 5](https://github.com/ADB-Team/railway-db-public/blob/main/query-plans/original/backup-transaction5.md)
