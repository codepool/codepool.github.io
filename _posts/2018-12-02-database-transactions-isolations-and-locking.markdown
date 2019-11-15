---
layout: post
title:  "Database transactions, isolations & locking"
date:   2018-12-02 13:02:24 +0530
categories: jekyll update
comments: true
---
We all have encountered situations where we want a bunch of queries to be executed together or none at all. Or we might want to handle situations where the queries can be executed by multiple concurrent connections. Database transactions and isolation levels solve these kinds of problems. While this topic is in context of Mysql InnoDB engine but the concepts work on other RDBMS databases in a similar way.
<br/>
### Problems

1. You want to execute a job only once at a time based on the value of a column (say F) in a table. If F=0, you want to proceed executing the job and update F=1 when job completes successfully, if F=1 you don’t want to do anything. Now, what happens if two concurrent sessions read F=0 simultaneously. You will end up executing your job twice, isn’t it?

1. You maintain a customer’s balance in an account. Let’s say the current balance is B. On receiving a withdrawal request of amount w1 , you read B from the table, run some logic in the code and then update the new balance into the table. What happens if another withdrawal request w2 comes in between the read and update. This new request is going to read the old value B and the final balance might be B-w2 (or B-w1), instead of B-(w1+w2)

1. When two concurrent requests comes in, you might want to allow the other one to fail if one succeeds.

Before we try solving these problems let’s pay attention to following points :
* A new database connection starts a new session.
* Mysql puts all single statement queries by default inside a transaction, and locks any update, so they are safe from concurrency problems implicitly
* If you start a new transaction within the same session before committing/rollbacking the first one, Mysql automatically commits the first transaction. So one session can have only one active transaction at a time.

### Types of locks in Mysql
InnoDB supports both row level and table level locking. Table level locking can be accomplished by explicitly using **LOCK READ** & **LOCK WRITE** commands. Let’s look at the types of row level locking:

#### Shared lock
If a transaction T1 acquires a shared lock, then it is permitted to read that row. Other transactions can still read the row **with or without acquiring shared lock**. Any attempt to update the row by other transactions will wait until T1 completes. A shared lock can be acquired by using **LOCK IN SHARE MODE** after a select query

#### Exclusive lock
If a transaction T1 acquires exclusive lock, then its permitted to read or write on that row. Any other transaction cannot acquire a **shared or exclusive lock**. and hence will wait for read or write until T1 completes. Exclusive lock can be acquired by using **FOR UPDATE** after a select query

Now to solve our problem 1 & 2 we can do the following :

{% highlight sql linenos %}

BEGIN TRANSACTION
select flag from mytable where id=1 FOR UPDATE; /* no other session can read the value */
/*
run business logic
*/
update mytable set flag=1 where id=1;
COMMIT;
{% endhighlight %}

Since **FOR UPDATE** acquires exclusive lock, another concurrent transaction T2 will wait on line 2. After T1 commits, T2 will read the new value 1

### Database Isolations

Isolation levels control the degree at which transaction T1 can be isolated with another transaction T2 when T1 & T2 happen to be in concurrent sessions.
Mysql supports 4 isolation levels — **READ COMMITTED**, **READ UNCOMMITTED**, **REPEATABLE READ**, **SERIALIZABLE**. Default isolation level is **REPEATABLE READ**

### Types Of Database Read Violations

When two transactions happen concurrently following violations can happen :

#### Uncommitted Read/Dirty Read
If a transaction T1 reads a value which is not yet committed by transaction T2, then its called dirty read. It’s called so because it might happen that due to some error T2 could not commit it, still T1 holds that value which is not clean.

#### Non Repeatable read
If a transaction T1 reads a value, shortly after that another transaction T2 updates that value. If T1 reads the value again and sees a different value then its called a non repeatable read

#### Phantom read
If a transaction T1 inserts or deletes some rows and T2 is still able to view the changes even before T1 commits, it’s called a phantom read.

### Violations allowed with different isolation levels
Different isolation levels control what violations are allowed when two transactions execute concurrently.

|| Dirty Read | Non-Repeatable Read | Phantom Read  |
|-----| :------: | :------: | :-----: |
**READ UNCOMMITTED** |  Yes  |  Yes  | Yes
**READ COMMITTED** |  No  |  Yes  | Yes
**REPEATABLE READ** |  No  |  No  | Yes
**SERIALIZABLE** | No | No | No

As you can see **SERIALIZABLE** provides highest level of isolation. We should also take note that higher the level of isolation, lower will be the performance you will receive from the database. Serializable would produce the same results as the one we would have got, had the transactions ran sequentially.

We can solve our mentioned problem 3 using SERIALIZABLE isolation level

{% highlight sql linenos %}
SET SESSION  TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION
select balance from accounts where customer_id=100; /* no other session can update the value */
/*
run business logic
*/
update mytable balance= balance - 500 where customer_id=100
COMMIT;
{% endhighlight %}

**SERIALIZABLE** by default appends **LOCK IN SHARE MODE** in all select queries. As transaction T1 reads at step 3, another concurrent transaction T2 can also read parallelly. But as soon as both try to update on step 7, T2 will receive **deadlock** error while T1 will be successful since T1 started the transaction first.

### REPEATABLE READ vs READ COMMITTED
Repeatable Read and Read Committed have almost same behaviour except repeatable read allows reading the same value within a transaction. Mysql does this by managing multi version concurrency control. A version of the record is maintained as soon as a transaction reads that row. The same version is maintain throughout the transaction even if it’s updated to a new value by another transaction.


Mysql isolations should be used carefully since they might downgrade performance of your application. The default isolation level in Mysql works fine in most of the cases.
