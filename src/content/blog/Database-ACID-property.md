---
author: Saurabh Dalakoti
pubDatetime: 2024-08-24T16:51:09Z
modDatetime: 2024-08-24T14:51:42Z
title: SQL database Isolation and Concurrency
draft: false
tags:
  - backend
description: Discussing Isolation (ACID) and concurrency in SQL databases
---

# Starting the project

```shell
mkdir db-acid
cd db-acid
go mod init dalakoti07/sd/db-acid
go mod tidy
```

# Create Table

```sql
CREATE TABLE accounts ( id SERIAL PRIMARY KEY, name VARCHAR(50) NOT NULL, balance NUMERIC );

INSERT INTO accounts (name, balance) VALUES ('SAURABH',0);

INSERT INTO accounts (name, balance) VALUES ('NITIN',100);

```

## Simulate transaction and see db isolation out of the box

After connecting to the database, lets open 2 terminals and see how db transactions are in isolation by default.

```sql
BEGIN;

UPDATE accounts SET BALANCE = 50 where name = 'SAURABH';

UPDATE accounts SET BALANCE = 50 where name = 'NITIN';
```

Now, if we see in other terminal with command as `select * from accounts;` we get

```txt
id |  name   | balance 
----+---------+---------
  2 | SAURABH |       0
  3 | NITIN   |     100
```

which implies that transaction is in isolation by default

# Code Simulation for Isolation

## The baseline - sequential code

Lets write a simple go program which simulates a simple bank, money coming and going to a particular account

```go
// transferMoney transfers a specified amount from one account to another within a single transactionfunc transferMoney(db *sql.DB, fromAccount string, toAccount string, amount float64) error {
   // Begin a transaction
   tx, err := db.Begin()
   if err != nil {
      return err
   }
   defer func() {
      if err != nil {
         tx.Rollback()
      } else {
         err = tx.Commit()
      }
   }()

   // Step 1: Deduct amount from the source account
   _, err = tx.Exec("UPDATE accounts SET balance = balance - $1 WHERE name = $2", amount, fromAccount)
   if err != nil {
      return err
   }

   // Step 2: Add amount to the destination account
   _, err = tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE name = $2", amount, toAccount)
   if err != nil {
      return err
   }

   return nil
}


```

```go
func TestAccountTransfers(t *testing.T) {
   db, err := setupDB()
   if err != nil {
      t.Fatalf("Failed to connect to the database: %v", err)
   }
   defer db.Close()

   // Delete table if it exists
   _, err = db.Exec(`DROP TABLE IF EXISTS accounts`)
   if err != nil {
      t.Fatalf("Failed to drop table: %v", err)
   }

   // Create the table
   _, err = db.Exec(`
   CREATE TABLE accounts (      id SERIAL PRIMARY KEY,      name VARCHAR(255) NOT NULL,      balance NUMERIC NOT NULL   )`)
   if err != nil {
      t.Fatalf("Failed to create table: %v", err)
   }

   // Insert SAURABH account
   _, err = db.Exec(`INSERT INTO accounts (name, balance) VALUES ($1, $2)`, "SAURABH", 0)
   if err != nil {
      t.Fatalf("Failed to insert SAURABH account: %v", err)
   }

   // create 100 accounts
   for i := 0; i < 100; i++ {
      _, err := db.Exec(`INSERT INTO accounts (name, balance) VALUES ($1, $2)`, "Account"+fmt.Sprintf("-%d", i), 100)
      if err != nil {
         t.Fatalf("Failed to create account %d: %v", i, err)
      }
   }

   // transfer money from 100 accounts to SAURABH account
   for i := 0; i < 100; i++ {
      err = transferMoney(db, "Account"+fmt.Sprintf("-%d", i), "SAURABH", 100)
      if err != nil {
         fmt.Printf("error in transferring amount from %s to Saurabh", "Account"+fmt.Sprintf("-%d", i))
      }
   }

   if err != nil {
      t.Fatalf("Failed to commit transaction: %v", err)
   }

   // Validate the results
   row := db.QueryRow(`SELECT balance FROM accounts WHERE name = $1`, "SAURABH")
   var balance float64
   err = row.Scan(&balance)
   if err != nil {
      t.Fatalf("Failed to query SAURABH balance: %v", err)
   }

   assert.Equal(t, 10_000.0, balance, "Balance of SAURABH should be 1000")
}
```

## The real world scenario

The issue with above implementation was that it was sequential, it was not concurrent, and as a wise man said, the world is concurrent not sequential. So lets simulate concurrent stuff.

```go
// create TotalAccounts accounts
for i := 0; i < TotalAccounts; i++ {
   _, err := db.Exec(`INSERT INTO accounts (name, balance) VALUES ($1, $2)`, "Account"+fmt.Sprintf("-%d", i), 100)
   if err != nil {
      t.Fatalf("Failed to create account %d: %v", i, err)
   }
}

// use goroutine
var wg sync.WaitGroup
// transfer money from TotalAccounts accounts to SAURABH account
for i := 0; i < TotalAccounts; i++ {
   wg.Add(1)
   go func(i int) {
      defer wg.Done()
      err = transferMoney(db, "Account"+fmt.Sprintf("-%d", i), "SAURABH", 100)
      if err != nil {
         fmt.Printf("%s error in transferring amount from %s to Saurabh\n", err.Error(), "Account"+fmt.Sprintf("-%d", i))
      }
   }(i)
}
wg.Wait()
if err != nil {
   t.Fatalf("Failed to commit transaction: %v", err)
}
println("All transfer done")
```

> It would work fine, on 100 go-routines, but on 1000 go-routines it would give `read tcp 127.0.0.1:64324->127.0.0.1:5432: read: connection reset by peer` and `write tcp 127.0.0.1:64148->127.0.0.1:5432: write: broken pipe`

```txt
oo_much_money_test.go:123:
                Error Trace:    /Users/saurabhdalakoti/IdeaProjects/low-level-designs/db-acid/too_much_money_test.go:123
                Error:          Not equal:
                                expected: 100000
                                actual  : 83000
                Test:           TestAccountTransfers
                Messages:       Balance of SAURABH should be 1000
--- FAIL: TestAccountTransfers (0.69s)

```

## Increasing the concurrent connection in postgres

```go

// Configure connection pool
db.SetMaxOpenConns(100) // Adjust according to your needs
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(30 * time.Minute) // Adjust according to your needs
```

```sql
SHOW config_file;
```

make an entry that says

```txt
max_connections = 2000  # Adjust this number based on your requirements`
```

that also did not worked same error, and test suit failed

```txt
write tcp [::1]:60300->[::1]:5432: write: broken pipe error in transferring amount from Account-946 to Saurabh
read tcp [::1]:60310->[::1]:5432: read: connection reset by peer error in transferring amount from Account-298 to Saurabh
dial tcp [::1]:5432: connect: connection reset by peer error in transferring amount from Account-974 to Saurabh
dial tcp [::1]:5432: connect: connection reset by peer error in transferring amount from Account-506 to Saurabh
dial tcp [::1]:5432: connect: connection reset by peer error in transferring amount from Account-476 to Saurabh
dial tcp [::1]:5432: connect: connection reset by peer error in transferring amount from Account-545 to Saurabh
All transfer done
    too_much_money_test.go:131:
                Error Trace:    /Users/saurabhdalakoti/IdeaProjects/low-level-designs/db-acid/too_much_money_test.go:131
                Error:          Not equal:
                                expected: 100000
                                actual  : 65400
                Test:           TestAccountTransfers
                Messages:       Balance of SAURABH should be 1000
--- FAIL: TestAccountTransfers (1.10s)

```

![Image](../../assets/images/acid/Pasted_image_20240824190751.png)

## Solve with Locking and mutex

Since we want to limit db access to only a few coroutines not to 1000 at a time, hence a locking mechanism would be better.

```go
var mutex sync.Mutex

// transferMoney transfers a specified amount from one account to another within a single transaction

func transferMoney(db *sql.DB, fromAccount string, toAccount string, amount float64) error {
   mutex.Lock()
   defer mutex.Unlock()
   ....
}

```

![Image](../../assets/images/acid/Pasted_image_20240824191027.png)

# Summarising it up

Yes, the example provided demonstrates how to use transactions in Go with PostgreSQL, which ensures ACID (Atomicity, Consistency, Isolation, Durability) properties. Here’s a brief overview of how the example addresses ACID principles:

## 1. **Atomicity**

The code ensures atomicity by using transactions. Each `transferMoney` function call starts a transaction with `tx, err := db.Begin()`. Within this transaction:

- If any error occurs during the transaction, the `tx.Rollback()` call will undo all changes made during the transaction.
- If no error occurs, `tx.Commit()` will save all changes to the database.

This ensures that either both the debit and credit operations are completed, or neither is.

## 2. **Consistency**

Consistency is maintained by performing both debit and credit operations within the same transaction. This ensures that the database remains in a consistent state, with the total amount of money correctly accounted for before and after the transaction.

## 3. **Isolation**

Isolation is achieved because each transaction is isolated from others until it is committed. PostgreSQL’s default isolation level is **Read Committed**, which means:

- Transactions will see committed data from other transactions.
- Uncommitted changes from other transactions are not visible.

In your case, each `transferMoney` call operates within its transaction, ensuring that concurrent operations are handled without interfering with each other. However, for more rigorous isolation, you can configure the isolation level explicitly if needed:

`_, err = tx.Exec("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")`

## 4. **Durability**

Durability ensures that once a transaction is committed, it will persist even in the event of a system crash. PostgreSQL handles this automatically by ensuring that changes are written to disk.

# Does this locking affect read queries?

No, it does not affect the read access, as the lock is on different CS function

```go
func readBalanceOfSaurabh(db *sql.DB, t *testing.T) {
   start := time.Now()
   row := db.QueryRow(`SELECT balance FROM accounts WHERE name = $1`, "SAURABH")
   var balance float64
   err := row.Scan(&balance)
   if err != nil {
      t.Fatalf("Failed to query SAURABH balance: %v", err)
   }
   elapsed := time.Since(start)
   fmt.Printf("Read SAURABH's account balance %v and in time %v\n", balance, elapsed)
}

func testFunction(){
	// ...
	readBalanceOfSaurabh(db, t)
	// ....
}
```

```txt
=== RUN   TestAccountTransfers
Read SAURABH's account balance 800 and in time 3.812625ms
All transfer done
--- PASS: TestAccountTransfers (0.43s)
PASS
ok      dalakoti07/sd/db-acid   0.647s

```

# Using SERIALIZABLE as TRANSACTION ISOLATION

Lets see thing changes when Transaction Isolation is made Serialisable how does it affect things

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Nothing happened in this case, the metric time to run `SERIALIZABLE` test case was similar to that of `READ COMMITTED`
