---
author: Saurabh Dalakoti
pubDatetime: 2024-07-28T14:51:42Z
modDatetime: 2024-07-28T14:54:32Z
title: Design Blog App
featured: false
draft: false
tags:
  - Engineering
  - SystemDesign
description: Write way to structure blog app
---

# Why you bother reading this article?

While designing the blog app, we have 2 choices

- put blog content inside the database
- put blog content outside the database

This article benchmarks both of these approaches and lets see, which approach is efficient at large scale, when blog content is too much like 64KB, or 1MB or even more.

# The project setup

Let's get started, a simple goLang application

```shell
mkdir blog-app
cd blog-app
go mod init dalakoti07/sd/blog
go mod tidy
```

other SQL commands

```sql
\dt

```

# The code for Benchmark

```go
package main

import (
   "bytes"
   "database/sql"   "fmt"   "log"   "math/rand"   "time"
   _ "github.com/lib/pq"
)

const (
   // Database connection string
   connStr = "user=postgres dbname=playground_db sslmode=disable"
)

const (
   // Size of the text file in bytes
   fileSize = 1 * 1024 * 1024 // 3 MB
)

// Function to generate random text of specified size
func generateRandomText(size int) string {
   var buffer bytes.Buffer
   chars := "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
   for buffer.Len() < size {
      buffer.WriteString(chars)
   }
   return buffer.String()[:size]
}

// Function to create 1000 blog articles in SQL rows
func createArticlesInRows(db *sql.DB) {
   // Generate 3 MB of random text
   text := generateRandomText(fileSize)

   for i := 1; i <= 1000; i++ {
      isLongArticle := false
      content := "This is the content of the article."
      if i%5 == 0 {
         content = text
         isLongArticle = true
      }
      _, err := db.Exec(
         fmt.Sprintf("INSERT INTO %s (title, content, longArticle) VALUES ($1, $2, $3)", tableName),
         fmt.Sprintf("Article %d", i),
         content,
         isLongArticle,
      )
      if err != nil {
         log.Fatal(err)
      }
   }
}

// Function to create 1000 blog articles with file paths
func createArticlesWithFiles(db *sql.DB) {
   for i := 1; i <= 1000; i++ {
      filePath := fmt.Sprintf("/path/to/articles/article_%d.txt", i)
      _, err := db.Exec(
         fmt.Sprintf("INSERT INTO %s (title, file_path) VALUES ($1, $2)", tableName),
         fmt.Sprintf("Article %d", i), filePath,
      )
      if err != nil {
         log.Fatal(err)
      }
   }
}

// Function to benchmark querying all articles
func benchmarkQuery(db *sql.DB, useFiles bool) time.Duration {
   start := time.Now()

   var rows *sql.Rows
   var err error

   if useFiles {
      rows, err = db.Query(fmt.Sprintf("SELECT title,file_path FROM %s", tableName))
   } else {
      rows, err = db.Query(fmt.Sprintf("SELECT title,content FROM %s", tableName))
   }
   if err != nil {
      log.Fatal(err)
   }
   defer rows.Close()

   for rows.Next() {
      var title string
      var content string
      if useFiles {
         err = rows.Scan(&title, &content)
      } else {
         err = rows.Scan(&title, &content)
      }
      if err != nil {
         log.Fatal(err)
      }
   }

   return time.Since(start)
}

var tableName = "blog_inplace_content"

func main() {
   rand.Seed(time.Now().UnixNano())

   db, err := sql.Open("postgres", connStr)
   if err != nil {
      log.Fatal(err)
   }
   defer db.Close()

   _, err = db.Exec(fmt.Sprintf("DROP TABLE IF EXISTS %s", tableName))
   if err != nil {
      log.Fatal(err)
   }

   approachOne(db)
   // Drop and recreate table for file paths approach
   _, err = db.Exec(fmt.Sprintf("DROP TABLE IF EXISTS %s", tableName))
   if err != nil {
      log.Fatal(err)
   }
   approachTwo(db)
}

func approachTwo(db *sql.DB) {
   _, err := db.Exec(
      fmt.Sprintf(
         "CREATE TABLE %s (id SERIAL PRIMARY KEY, title VARCHAR(255) NOT NULL, file_path VARCHAR(255), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)",
         tableName,
      ),
   )
   if err != nil {
      log.Fatal(err)
   }

   fmt.Println("Inserting articles with file paths...")
   createArticlesWithFiles(db)
   durationFiles := benchmarkQuery(db, true)
   fmt.Printf("Querying with file paths took: %v\n", durationFiles)
}

func approachOne(db *sql.DB) {
   _, err := db.Exec(fmt.Sprintf("create table %s (", tableName) +
      "id Serial PRIMARY KEY," +
      "title VARCHAR(255) NOT NULL," +
      "content TEXT NOT NULL," +
      "longArticle BOOLEAN DEFAULT FALSE," +
      "created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);",
   )
   if err != nil {
      log.Fatal(err)
   }

   // Insert articles and benchmark
   fmt.Println("Inserting articles with content in rows...")
   createArticlesInRows(db)
   durationRows := benchmarkQuery(db, false)
   fmt.Printf("Querying with content in rows took: %v\n", durationRows)
}
```

When storing 3MB of data, I means storing a file path is 2000 times faster, milli seconds and micro seconds

```sql
Inserting articles with content in rows...
Querying with content in rows took: 475.581541ms
Inserting articles with file paths...
Querying with file paths took: 243.041µs

```

Lets do this for 1 MB of data

```shell
Inserting articles with content in rows...
Querying with content in rows took: 135.909292ms
Inserting articles with file paths...
Querying with file paths took: 232.833µs

```

Lets do this for 100 KB data, considering 5 chars on each words, and utef uses 4 bytes and 5000 words in a article which is `4*5*5000` = 100 Kb

```shell
Inserting articles with file paths...
Querying with file paths took: 409.083µs
Inserting articles with content in rows...
Querying with content in rows took: 13.894291ms
```

That is also significant, lets do one more thing, like add a flag to DB that article is long or short, and then would make query of non short items and then lets see what happens.

```shell
Inserting articles with file paths...
Querying with file paths took: 615.166µs
Inserting articles with content in rows...
Querying with content in rows took: 886.625µs

```

Lets talk how database handle queries, and data retreival. Modern databases use something called [predicate push](https://airbyte.com/data-engineering-resources/predicate-pushdown#:~:text=Predicate%20pushdown%20is%20a%20query,of%20data%20transmitted%20and%20processed.).
**Predicate pushdown** is a query optimization **technique** used in database technologies. It enables developers to **filter** data at the data source, reducing the amount of data transmitted and processed.

# Conclusion

So dumping all the blog content in database would slow the `select *` query, so putting a content off the db row is a better choice. Maybe in a text file or a mongo db separately.

# Next Step

benchmark search query, with 2 approaches

- blog content in db
- blog content in mongo
