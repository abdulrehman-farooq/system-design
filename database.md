# Database

## Summary
In this particular section, I'm going to cover different database concepts that most software engineers should know in 
order to design efficient and scalable systems. 


# ACID Properties

In order to maintain consistency before/after transaction database follows certain properties. These are called ACID 
properties.

_**Atomicity:**_  The whole transaction happens at once or not at all. If one part of the transaction fails, the entire 
transaction need to fails and database state is left unchanged. If the whole transaction completes then the changes are 
visible. 

_**Consistency:**_  The database should be in consistent state before and after the transaction i.e only valid data is
written to the database. Consistency enforces integrity constraints to maintain the accuracy and correctness of data.

_**Isolation:**_  The intermediate state of the transaction should not be visible to other transactions. Changes 
occurring in a particular transaction will not be visible to any other transaction until that particular change in that
transaction is written to memory or has been committed. This property ensures that the execution of transactions 
concurrently will result in a state that is equivalent to a state achieved these were executed serially in some order.

_**Durability:**_ Once the transaction is complete, the changes made to the database should be permanent and should not 
be lost even if the system fails. 


# Transactional Isolation Problem  

Isolation levels define the degree to which a transaction must be isolated from the data modifications made by any other
transaction in the database system. A transaction isolation level is defined by the following phenomena:

_**Dirty Read**_ – A Dirty read is a situation when a transaction reads data that has not yet been committed. 
For example, Let’s say transaction 1 updates a row and leaves it uncommitted, meanwhile, Transaction 2 reads the updated 
row. If transaction 1 rolls back the change, transaction 2 will have read data that is considered never to have existed.

_**Non Repeatable Read**_ – Non Repeatable read occurs when a transaction reads the same rows twice and gets a different 
value each time. For example, suppose transaction T1 reads data. Due to concurrency, another transaction T2 updates the 
same data and commit, Now if transaction T1 rereads the same data, it will retrieve a different value. e.g Transaction 
A reads the account balance, Transaction B deposits money, and then Transaction A reads the balance again, 
finding it changed.

_**Phantom Read**_ – A transaction re-executes a query returning a set of rows that satisfy a search condition and finds 
that the set of rows satisfying the condition has changed due to another recently-committed transaction.For example, 
suppose transaction T1 retrieves a set of rows that satisfy some search criteria. Now, Transaction T2 generates some new 
rows that match the search criteria for transaction T1. If transaction T1 re-executes the statement that reads the rows, 
it gets a different set of rows this time. e.g Transaction A counts the number of accounts with a balance above a certain 
threshold. Transaction B creates a new account that meets the criteria, causing Transaction A to get a different count 
if it reads again.

_**Lost Updates**_: When two transactions concurrently update the same data, one transaction's update might overwrite the 
other's, leading to data loss. e.g Transaction A and Transaction B both withdraw from the same account simultaneously, 
each reading the balance and then updating it. One update might overwrite the other, leading to incorrect balances.


# Transactional Isolation Level

_**Read Uncommitted**_ – Read Uncommitted is the lowest isolation level. In this level, one transaction will read latest 
version of a row that is modified by any transaction, thereby allowing dirty reads. At this level, transactions are 
not isolated from each other.

_**Read Committed**_ – This isolation level guarantees that any data read is committed at the moment it is read.
In innodb the "Read Committed" isolation level, each consistent read, even within the same transaction, sets and reads 
its own fresh snapshot. It acquires a shared lock on Read statement and exclusive lock when we update a 
field.


_**Repeatable Read**_ – The "Repeatable Read" isolation level ensures that if a transaction reads a row, subsequent reads of 
that same row within the same transaction will return the same data, even if other transactions modify the data in the 
meantime. This level prevents non-repeatable reads and dirty reads but allows for phantom reads. Innodb uses Multiversion 
Concurrency Control (MVCC) to manage concurrent transactions. A consistent snapshot of the database is created at the 
start of each transaction and this snapshot is used for all reads within the transaction. It acquires a shared lock on Read 
statement and exclusive lock when we update a field. 

_**Serializable**_

It ensures complete isolation from other transactions, making concurrent transactions behave as if they were executed 
one after another, in sequence. In innodb it will apply share lock creates a shared lock on all select statements
whether you use for share or not. Due to the excessive locks used, there is a greater risk of deadlocks occurring.



# Locking In MYSQL DB 

### Shared and Exclusive Locks
---

A _**Shared Lock:**_ is a kind of lock that allows other transactions to read the locked object, and to also acquire 
other shared locks on it, but not to write to it. 

A _**Exclusive Lock:**_  is a kind of lock that permits the transaction that holds the lock to update or delete a row. 
It doesn't allow other transaction to hold any lock.


> If transaction T1 holds a shared (S) lock on row r A request by T2 for an S lock can be granted  but request by T2 for an X lock cannot be granted immediately T2 has to wait for transaction T1 to release its lock on row r.

> If a transaction T1 holds an exclusive (X) lock on row r, a request from some distinct transaction T2 for a lock of either type on r cannot be granted immediately.T2 has to wait for transaction T1 to release its lock on row r.
 
 
Intention Locks
---
To make locking at multiple granularity levels practical, InnoDB uses intention locks.

An _**intention shared lock (IS)**_ indicates that a transaction intends to set a shared lock on individual rows in a table.

An _**intention exclusive lock (IX)**_ indicates that a transaction intends to set an exclusive lock on individual rows in a table.


>Before a transaction can acquire an S lock on a row in table t, it must first acquire an IS or stronger lock on table t. Before a transaction can acquire an X lock on a row, it must first acquire an IX lock on table t.


Table-level lock type compatibility is summarized in the following matrix.

|          |X        |	IX      |	  S   |	       IS|
| -------- | ------- | -------- | ------- | -------- |
|X  |	Conflict|	Conflict	|Conflict	|Conflict|
|IX	|Conflict	|Compatible	|Conflict	|Compatible|
|S	|Conflict	|Conflict	|Compatible	|Compatible|
|IS	|Conflict	|Compatible	|Compatible	|Compatible|

A lock is granted to a requesting transaction if it is compatible with existing locks, but not if it conflicts with 
existing locks. A transaction waits until the conflicting existing lock is released. If a lock request conflicts with an
existing lock and cannot be granted because it would cause deadlock, an error occurs.

Intention locks do not block anything except full table requests (for example, LOCK TABLES ... WRITE). The main purpose
of intention locks is to show that someone is locking a row, or going to lock a row in the table.

### Record Locks

Record locks are used to lock individual rows in a table. They prevent other transactions from modifying or reading the 
locked rows until the lock is released. Record locks always lock index records, even if a table is defined with no 
indexes. For such cases, InnoDB creates a hidden clustered index and uses this index for record locking.

### Gap Locks 

Gap locks are used to lock gaps between index entries, rather than the entries themselves. They prevent other 
transactions from inserting new rows into the gap between existing index entries.Gap locks in InnoDB are use to is to 
prevent other transactions from inserting to the gap.

Example:
Transaction A locks the gap between the values 10 and 20 in an index.
Transaction B cannot insert a new row with a value between 10 and 20 until Transaction A releases the lock.

### Next Key Locks

Next-key locking is a technique that combines row-level locking (index-record locks) with gap locking. When InnoDB
scans or searches a table index, it sets locks not only on the individual index records but also on the gaps between 
these records. This ensures that no other transactions can insert, update, or delete rows within the locked range.

Example:

Consider a table employees with an index on the salary column. Assume the index has the following entries: 30,000, 40,000, 50,000.

Transaction A performs a query: SELECT * FROM employees WHERE salary BETWEEN 35000 AND 45000 FOR UPDATE.

This query would lock the index records for 40,000 and the gap before it (35,000 to 40,000).
The lock on the gap ensures that no new records with a salary in the range 35,000 to 40,000 can be inserted by other 
transactions. Transaction B tries to insert a new row with a salary of 37,000.

Since Transaction A holds a next-key lock on the gap before 40,000, Transaction B is blocked from inserting the new row 
until Transaction A releases the lock.

### AUTO-INC Locks ** Need To add more details

An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. 
In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their 
own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.

The innodb_autoinc_lock_mode variable controls the algorithm used for auto-increment locking

##  Database Index Basic

An index is a data structure that improves the speed of data retrieval operations on database tables. It acts like a
pointer to the actual data, allowing the database management system to locate specific rows quickly without scanning the entire table.

### Structure Use to store index is B-Tree  

A B-Tree is a self-balancing tree data structure that maintains sorted data and allows searches, sequential access, 
insertions, and deletions in logarithmic time. The B+Tree, a variant of the B-Tree, is typically used in databases.

#### Key characteristics of a B+Tree:

**Nodes:** Internal nodes and leaf nodes.
**Internal** Nodes: Store keys and pointers to child nodes.
**Leaf Nodes:** Store actual data values or pointers to the data (in the case of secondary indexes).

_**Note:** Other structure are also used_ 

### Clustered Index
Determines the physical order of data in a table. The table data is stored in the leaf nodes of the index.
**Characteristics:**
 1. Only one clustered index per table.
 2. Data rows are stored in the order of the index.
 3. Usually created on the primary key.
 4. Efficient for range queries and sorting operations.

### Non-Clustered Index
An index structure separate from the data rows. The leaf nodes contain pointers to the data rows.
**Characteristics:**
 1. Multiple non-clustered indexes can be created on a table.
 2. Does not affect the physical order of data rows.
 3. Useful for columns frequently used in search queries and joins.
 4. Contains index entries and pointers to the actual data rows.

### Types Of Index

**1. Column Index:**   An index on a single column

**2. Compound Index:** An index on multiple columns. If the table has a multiple-column index, any leftmost prefix of the 
index can be used by the optimizer to look up rows. For example, if you have a three-column index on (col1, col2, col3), you
have indexed search capabilities on (col1), (col1, col2), and (col1, col2, col3).

**3. Covering Index:** A covering index is a regular index that provides all the data required for a query without having 
to access the actual table. When a query is executed, the database looks for the required data in the index tree, 
retrieves it, and returns the result.

**4. Partial Index:** An index that contains a subset of the column in a table e.g the first 10 character of a person index

#### Are Too Many Index Bad?

1. It takes a lot of storage space and make the table slower for large database
2. It makes update/add/delete slower because it have to update the index as well.



#### Sow Index Reasons
1. Low Cardinality
2. Large Data Sets
3. Multiple Column Index Traversal:
4. Index Column Used As Function Argument: (In such case index will not be used becuase we have index on column not a function)
5. Data Type Mismatch:

# Database Query Optimization

Many interviewer evaluate candidate query optimization skill. This section will help you master the query optimization
skill needed as Software Engineer which help you in your job plus help you in your interview.

## Select Query Execution Order

Understanding the execution order of a SELECT query is of utmost importance before delving into the intricacies of SQL
query optimization. It plays a crucial role in comprehending the query's functionality and how the database engine
handles data processing and retrieval.

**FROM clause:** The FROM clause is the first part of the query to be processed. It specifies the tables or views from
which the data will be retrieved or modified. If multiple tables are involved in the query, the database engine
determines how these tables are related (via JOINs) and creates a combined virtual table known as the result set.

**WHERE clause:** The WHERE clause is processed after the FROM clause. It filters the data in the result set based on
specified conditions. Rows that do not meet the conditions specified in the WHERE clause are excluded from the result
set.

**GROUP BY clause:** If the query includes a GROUP BY clause, the data in the result set is then grouped into subsets
based on the specified columns. Aggregate functions (such as SUM, AVG, COUNT, etc.) can be used to calculate summary
values for each group.

**HAVING clause:** The HAVING clause is similar to the WHERE clause but is used to filter groups resulting from the
GROUP BY clause. It filters groups based on aggregate functions' results, excluding groups that do not meet the
specified conditions.

**SELECT clause:** The SELECT clause is processed after all the previous clauses. It determines which columns or
expressions should be included in the final result set. Any expressions or calculations specified in the SELECT clause
are also computed at this stage.

**ORDER BY clause:** If the query includes an ORDER BY clause, the result set is then sorted based on the specified
columns or expressions. The ORDER BY clause can sort the data in ascending or descending order.

**LIMIT/OFFSET (Optional):** In some database systems, there might be a LIMIT and/or OFFSET clause at the end of the
query. The LIMIT clause restricts the number of rows returned in the result set, while the OFFSET clause specifies how
many rows to skip before starting to return rows.

<!---

@startuml

start

:FROM Clause;
:WHERE Clause;
:GROUP BY Clause;
:HAVING Clause;
:SELECT Clause;
:ORDER BY Clause;
:LIMIT/OFFSET;

stop

@enduml
--->
![SQL Query Execution Order](./img/sql-query-execution-order.png)

## SQL Query Execution Plain using Explain Query

The First part of query optimization is to understand the execution of query using query execution plan.

> The set of operations that the optimizer chooses to perform the most efficient query is called the “query execution
> plan”, also known as the EXPLAIN
> plan. Your goals are to recognize the aspects of the EXPLAIN plan that indicate a query is optimized well, and to learn
> the
> SQL syntax and indexing techniques to improve the plan if you see some inefficient operations.

We can add Explain in front of SELECT, DELETE, INSERT, REPLACE, and UPDATE statements, and it will give us the execution
plan.

```  
EXPLAIN SELECT * FROM film f JOIN film actor fa ON fa.film id = f.film id JOIN actor a ON a.actor id = fa.actor id;\G
*************************** 1. row ***************************
           id: 1
  select_type: Simple
        table: a
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 200
     filtered: 100.00
        Extra: Using index
*************************** 2. row ***************************
           id: 2
  select_type: SUBQUERY
        table: fa
         type: ref
possible_keys: PRIMARY,idx_fk_film_id
          key: PRIMARY
      key_len: 5
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index
*************************** 3. row ***************************
           id: 2
  select_type: SUBQUERY
        table: f
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index        
3 rows in set, 1 warning (0.00 sec) 
```

The following tables explain the output of the Explain query.

| Column	       | JSON           | 	Meaning                                                                                                                         |                                        
|---------------|----------------|----------------------------------------------------------------------------------------------------------------------------------|
| id	           | select_id      | This is the sequential number of the SELECT within the query.                                                                    |
| select_type   | 	None          | 	The SELECT type                                                                                                                 |
| table	        | table_name	    | The table for the output row                                                                                                     |
| partitions    | 	partitions    | 	The partitions from which records would be matched by the query.                                                                |                       
| type          | 	access_type   | 	The join type                                                                                                                   |                      
| possible_keys | 	possible_keys | 	The possible indexes to choose                                                                                                  |                 
| key	          | key            | 	The index actually chosen                                                                                                       |            
| key_len       | 	key_length    | 	The length of the chosen key                                                                                                    |               
| ref	          | ref	           | The ref column shows which columns or constants are compared to the index named in the key column to select rows from the table. |            
| rows	         | rows	          | Estimate of rows to be examined                                                                                                  |          
| filtered      | 	filtered      | 	Percentage of rows filtered by table condition                                                                                  |
| Extra         | None	          | Additional information                                                                                                           |                          

The Explain query have three format json,table and tree.


Join Types: Best To worst

_**const/ref:**_ The table has at most one matching row, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. const tables are very fast because they are read only once.

_**eq-ref**_: One row is read from this table for each combination of rows from the previous tables. Other than the system and const types, this is the best possible join type. It is used when all parts of an index are used by the join and the index is a PRIMARY KEY or UNIQUE NOT NULL inde.

_**ref:**_ Index Lookup return more than one row

_**ref-or-null**_ Index Lookup return more than one row but do one more pass for null

_**index-merge**_: This join type indicates that the Index Merge optimization is used. 

_**range:**_ Only rows that are in a given range are retrieved, using an index to select the rows. The key column in the output row indicates which index is used. The key_len contains the longest key part that was used. The ref column is NULL for this type.

_**Index:**_ The index join type is the same as ALL, except that the index tree is scanned.

_**ALL:**_ A full table scan is done for each combination of rows from the previous tables. 

Query Processing 

```
for (each row in table a [Actor] ) {
   for (each row in fa [film_actor] matching a.actor _id) {
      for the row in f [film] matching fa.film id ) {
      Add row to output
      }
   }
}

```
```
for (each row in outer table matching where clause) {
   for (each row in inner table matching key value
   and where clause){
   Add to output
   }
}   
```

Rows: It's the critical performance indicator. Total Approximation is the product of all the row in the tables. 

Bad Query indicators:

1. No index is used - NULL in key column
2. Large number of rows estimated
3. Using temporary
4. Using filesort
5. Using derived tables - DERIVED in
6. select type column
7. Joining on derived tables
8. Having dependent subqueries on a large result set



#### Query Optimization Best Practice
1. Negative Clause Are Not as Good As Positive Clause
2. Use UNION ALL Instead of UNION
3. Order By Can be expensive avoid when using small data
4. Use Inner instead of Left Join 
5. Do not use LIKE clause where the wildcards is the start in WHERE clause as it will not use index.  

#### How To Find Slow Queries

1. Slow Queries Logs
2. Process List

Understand How In Queries Works
Understand How Between Queries   
Understand Connection Pools 


# Key Value Database 

key-value database, often referred to as a key-value store, employs a straightforward approach to data storage. 
Within these databases, you'll find a basic string (known as the key), which is invariably unique, alongside a variable-sized 
data field (known as the value). These databases are characterized by their simplicity in both design and implementation.

The key value database use hashtable to store the unique keys which have the pointer to store values. Key-value stores 
have no query language, but they do provide a way to add and remove key-value pairs, some vendors being quite sophisticated. 
Values cannot be queried or searched upon. Only the key can be queried.

## When to use a key-value database

1. Application which have infrequent updates and doesn't have a complex queries
2. Application which Require High-speed Data Retrieval
3. Simple Data Model: When your data can be effectively represented as key-value pairs without complex relationships


## Use cases for key-value databases
1. Session Management on a Large Scale
2. Using Cache to Accelerate Application Responses
3. Storing Personal Data on Specific Users
4. Product Recommendations and Personalized Lists  
5. Managing Player Sessions in Massive Multiplayer Online Games

## REDIS 

### KEYS

In Redis Keys are the unique identifier use to retrieve or store data. They are binary safe which mean any binary sequence 
can be use as key e.g foobar, 45 -3.145. The size of key can 512 MB but long key name are not recommended. These keys are 
case-sensitive. The convention follow by redis community is values separated by `:` user:1000:password

### Redis Data structure
Redis is a data structure server. At its core, Redis provides a collection of native data types that help you solve a wide variety of problems, 
 from caching to queuing to event processing.

1. Strings

2. List 

HashSet 




Referenece  https://redis.com/nosql/key-value-databases/

