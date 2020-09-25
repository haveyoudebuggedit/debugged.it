---
slug: sql-introduction-part2
guid: https://pasztor.at/blog/sql-introduction-part2/
authors:
- janos
categories:
- Technology
date: "2016-10-21T00:00:00Z"
publishDate: "2016-10-21T00:00:00Z"
summary: In the previous episode of our journey we have spoken about the basics of
  writing an SQL query. However, we did not speak of the limitations you can place
  on SQL tables.
tags:
- Software Development
title: Introduction to SQL databases Part 2
---

If you remember the [previous part](/blog/sql-introduction-part1), in SQL we organize data into tables with fixed column
names. Having such a structure allows us to insert data sets as rows. If we need more complex data structures, we can do
that by creating multiple tables.

## Keys and Constraints

So far so good. However, you may have realized that nothing stops us from inserting the same ID twice. What's worse, we
could reference a student that does not exist or a class that does not exist.

For example:

```sql
INSERT INTO students (
  id,
  student_name
) VALUES (
  1,
  'Joe'
);

INSERT INTO students (
  id,
  student_name
) VALUES (
  1,
  'Suzy'
);

INSERT INTO students_classes (
  student_id,
  class_id
) VALUES (
  51234,
  96543
);
```

Let's do a `FULL JOIN` on these:

```sql
SELECT
  S.id,
  S.student_name,
  SC.student_id,
  SC.class_id
FROM
  students AS S
  FULL JOIN students_classes SC
    ON SC.student_id=S.student_id;
```

<figure>
<table>
<thead>
<tr><th>S.id</th><th>S.student_name</th><th>SC.student_id</th><th>SC.class_id</th></tr>
</thead>
<tbody>
<tr><td>1</td><td>Joe</td><td><code>NULL</code></td><td><code>NULL</code></td></tr>
<tr><td>1</td><td>Suzy</td><td><code>NULL</code></td><td><code>NULL</code></td></tr>
<tr><td><code>NULL</code></td><td><code>NULL</code></td><td>51234</td><td>96543</td></tr>
</tbody>
</table>
</figure>

Well done, our database is now a mess. We have duplicate ID entries, and our rows are not matching up. Great. As you
might imagine, this could lead to bugs and unforeseen problems in your programs, and huge discrepancies in business
analytics. So let's make sure this doesn't happen.

### Unique keys

First of all, let's learn how to create a constraint to allow only one value in the `student_id` column:

```sql
ALTER TABLE students ADD CON``STRAINT u_student_id UNIQUE (student_id);
```

Or if you would create the table anew:

```sql
CREATE TABLE students (
  student_id INT,
  student_name VARCHAR(255),
  CONSTRAINT u_student_id UNIQUE (student_id)
);
```

{{% warning %}}
**Be careful!** Depending on your server, unique keys may allow multiple rows with `NULL` values!
{{% /warning %}}

### Primary keys

However, there's a problem. Columns in unique keys are still allowed to have `NULL` values. While we can work around
that using a `NOT NULL` statement on the column, there is a better way to describe the primary column in a table is to
add a `PRIMARY KEY`.

```sql
ALTER TABLE students ADD CONSTRAINT pk_students PRIMARY KEY (student_id);
```

Or for new tables:

```sql
CREATE TABLE students (
  student_id INT,
  student_name VARCHAR(255),
  CONSTRAINT pk_students PRIMARY KEY (student_id)
);
```

### Primary keys vs. unique keys

Let's look at the difference:

<figure>
<table>
<thead><tr><th></th><th>Primary key</th><th>Unique key</th></tr></thead>
<tbody>
<tr><td>Can contain more than one column</td><td>Yes</td><td>Yes</td></tr>
<tr><td>Can contain <code>NULL</code> value</td><td>No</td><td>Yes (unless the column is <code>NOT NULL</code>)</td></tr>
<tr><td>More than one key in a table</td><td>No</td><td>Yes</td></tr>
</tbody>
</table>
</figure>

There are a few more differences, but most of those are unique to the SQL server you are using.

{{% warning %}}
**Common misconception** A lot of university teachers would tell you that every table needs a numeric primary key.
This is not true, you can create a primary key on a textual (`VARCHAR`) column, or you could create a primary key
using multiple columns. If your students are given a textual identification, for example, this can be easily used to
make your database mimic your business model more closely. On the other hand, having a column named `id` or `table_id`
will make your database easier to understand for a lot of people.
{{% /warning %}}

### Making the database fast (indexes)

Before we get into the data consistency issue, let's take a short detour. You are probably playing around with small
tables, so you most likely didn't notice speed problem. If you are moving to larger tables, you'll quickly realize that
your database becomes incredibly slow.

As you might have guessed, that's what why use indexes. By default, databases store all data in one big chunk, sorted by
the primary key (which is an index by the way). Without proper indexes, the database may need to read the full table in
order to get the data you need. This is called a *full table scan*.

We can speed up queries to a table by adding indexes. However, while **indexes speed up reads, they slow down writes**,
since the indexes need to be updated. Let's do that:

```sql
CREATE INDEX i_student_id ON students_classes (student_id);
```

As a general rule, adding indexes should be added as needed, by analyzing the queries that are slow. There is a
multitude of tools to monitor slow queries, but they are database specific so that we won't go into them here. However,
you *should* monitor your database servers for queries that perform poorly and fix them by adding the proper indexes.

### Foreign keys

Returning to the data consistency problem, we still have the issue of referencing values that do not exist. (Referencing
a non-existing class, etc.) We also have a tool for that, called foreign keys.

Foreign keys mean “only allow values that appear in the other table too” (or `NULL`, if allowed). To create a foreign
key you will need to add an index (regular, unique key or primary key) on the column that you want to reference. After
that, you can create a constraint like we did before:

```sql
ALTER TABLE students_classes
  ADD CONSTRAINT fk_students_student_id
    FOREIGN KEY (student_id)
    REFERENCES students(id);

ALTER TABLE students_classes
  ADD CONSTRAINT fk_students_class_id
    FOREIGN KEY (class_id)
    REFERENCES classes(id);
```

If you now try to insert a value that is not present in the referenced table, you should get an error.

One question remains: what happens if you *change* or *delete* a linked value? The SQL standard defines five actions:

- `CASCADE`: If the row is updated, the linked value is updated as well. If it is deleted, the referencing row is also deleted.
- `RESTRICT`: Block the change or deletion if the row is referenced.
- `NO ACTION`: Ignore the change or deletion. This will result in an invalid value in the referencing row.
- `SET NULL`: Set the referencing row to `NULL` on change/deletion.
- `SET DEFAULT`: set the referencing row to the default value on change/deletion.

Using it is pretty simple:

```sql
ALTER TABLE students_classes
  ADD CONSTRAINT fk_students_student_id
    FOREIGN KEY (student_id)
    REFERENCES students(id)
    ON UPDATE CASCADE 
    ON DELETE RESTRICT;
```

This basically translates to: “Only allow values in `students_classes.student_id` that also appear in
`students.student_id`. When the contents of the `students.student_id` field are updated, also update
`students_classes.student_id`. When the row in `students` that we are referencing is deleted, block that deletion.”

Still reads like an English sentence, right?

## Advanced queries

Well, now we've created an unholy mess with our tables, time to get some more data out of it. Here's some advanced
techniques for getting out the data you have successfully deposited in your database.

### Sorting the results

Sorting the results is easy, but may also need an index to be fast. Let's do just that:

```sql
SELECT
  class_id,
  class_name
FROM
  classes
ORDER BY
  class_name ASC;
```

### Fetching only the first X rows

In addition to `ORDER BY`, you can also limit the number of rows you get. Unfortunately, the syntax is different from
database to database. The following queries will fetch ten classes, starting at class 20.

MySQL, PostgreSQL, SQLite:

```sql
SELECT
  class_id,
  class_name
FROM
  classes
ORDER BY
  class_name ASC
LIMIT 10 OFFSET 20
```

Microsoft SQL Server, Oracle:

```sql
SELECT
  class_id,
  class_name
FROM
  classes
ORDER BY
  class_name ASC
OFFSET 20 FETCH NEXT 10 ROWS ONLY
```

Yeah, so much for SQL being a “standard”.

### Grouping results

We've talked about the duplication issue with joins before. One nice way of getting rid of said duplications is to use
the `GROUP BY` functionality. As the name says, it will take the specified column and group the results by that column.

If we wanted to count the number of students attending each class, for example, we'd do it like this:

```sql
SELECT
  classes.class_name,
  COUNT(*)
FROM
  classes
  INNER JOIN students_classes ON
    students_classes.class_id =
        classes.class_id
  INNER JOIN students ON
    students.student_id =
        students_classes.student_id
GROUP BY
  classes.class_name
```

This will aggregate all the rows by the class name, and then provide a count of how many items the group has.

### Subqueries

Now you have a lot of tools at your disposal. However, even that may not be enough to get the result you need. You
could, for example, use subqueries:

```sql
SELECT
  student_name,
  (
    SELECT
      COUNT(*)
    FROM
      students_classes
    WHERE
      students_classes = students.student_id
  ) AS classes_attended 
FROM
  students
```

As you can see, we have added a second query in brackets. This query will be executed for each row and the results will
be added. Any column from our outer query can be used in the subquery.

You can use subqueries as:

- **Fields**: in this case, your query has to return one row and one column 
- **Tables**: your subquery results will be used as a table you can join upon
- **Where**: The results of your subquery can be used as a condition.

## Let's talk about security

### Permissions

Most SQL databases allow you to restrict permissions for each of the users. Your production application should *never*
use the root user, nor should it have permissions to create or drop tables, databases. If you need to create new tables
in production, that usually indicates bad database design.

Ideally, you would restrict database permissions the basic query permissions and use a separate user to modify the
database structure.

### SQL injections

Another important security object, especially when talking about a web application, are SQL injections. This can happen
when dealing with user input. Imagine this query in any modern web language:

```java
string sql = 'SELECT
  id
FROM
  users
WHERE
  username="' + username + '"
  AND
  password="' + password + '"
```

Besides the fact that you should never save passwords in plain text, there is a much bigger problem. What if I provide
`username = 'admin'` and `password='" OR 1=1'` as input? Well, here's the resulting query:

```sql
SELECT
  id
FROM
  users
WHERE
  username="admin"
  AND
  password="" OR 1=1
```

This is called an SQL injection and will lead to your admin user being selected, regardless of the password supplied.
There are many ways of doing this, but in the end, they lead to leaked data (for example your whole username/password
database), privilege escalation (logging in as admin) or worst case scenario, deletion of data (inject a
`DROP DATABASE`). Ouch.

So you need to protect against SQL injection. And don't even try writing your custom “defense” functions, the specifics
depend on the SQL server type, version, and implementation details. Instead, use prepared statements.

Prepared statements consist of two parts. First, you prepare your query, for example:

```sql
SELECT
  id
FROM
  users
WHERE
  username=?
  AND
  password=?
```

When you execute the query, the server will safely replace the question marks with the parameters. Check your
programming language for details on how to do this properly.

## Conclusion

As you can see, SQL is a very powerful and sophisticated tool. It can start off easy but lead to mile-long queries with
execution times of hours or even days.

SQL is a timeworn standard if you can even call that. Most SQL implementations differ from the standard and each other
considerably, so looking at the database engine documentation is most recommended.

You will also find a lot more features, like useful functions, or other language constructs like `UNION` queries and
`CHECK` constraints, which we won't detail here.

When writing queries, try to keep it simple. Sometimes you don't need to solve everything in one query. You can extract
parts of the data, and put it in a different table for further processing. In fact, this is a good way of doing it, if
you don't need the result right away. What's more, there are tools for that, called ETL (Extract, Transform, and Load).
But that's a story for another day.
