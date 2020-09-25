---
slug: injection-type-vulnerabilities
guid: https://pasztor.at/blog/injection-type-vulnerabilities/
authors:
- janos
categories:
- Technology
date: "2018-02-23T00:00:00Z"
publishDate: "2018-02-23T00:00:00Z"
summary: XSS, SQL injection, and other injection-class vulnerabilities can cause some serious damage. Let's root them out!
images:
- posts/injection-vuln.png
preview: posts/injection-vuln.jpg
tags:
- Software Development
- Security
title: 'Injection vulnerabilities: Bobby Tables and Friends'
---

I recently [found a video where a guy got a free burger](https://www.youtube.com/watch?v=WWJTsKaJT_g) using a very
low-tech form of injection vulnerability. Before you go watch that video, let's talk about what an injection 
vulnerability is and how it can be avoided with the right programming practices.

## Bobby Tables says hi

Let's take a very simple example, an SQL query:

```sql
SELECT
  *
FROM
  users
WHERE
  username="janoszen" AND password="supersecret"
```

Let's put the fact aside that you *really shouldn't store passwords in plain text*, and focus on the query. Let's say
the user inputs a quote mark in their password. Here's what happens:

```sql
SELECT
  *
FROM
  users
WHERE
  username="janoszen" AND password="supersecret""
```

As you can see, this breaks the query and will result in an error. But here's something even nastier. What if the user
enters `janoszen" --` in the username field?

```sql
SELECT
  *
FROM
  users
WHERE
  username="janoszen" -- " AND password="idontknow"
```

As you can see the rest of the query just commented out, so the login is successful without even knowing the password.
This specific vulnerability is called an SQL injection and can be used to do much nastier stuff:

```sql
SELECT
  *
FROM
  users
WHERE
  username="janoszen"; DROP TABLE users; -- " AND password="idontknow"
```

{{% tip %}}
**Joke:** <br />
School principal: Is your son really called `Robert'); DROP TABLE students;--` ? <br />
Mom: Oh yeah, we call him little Bobby Tables.<br />
Via [XKCD](https://xkcd.com/327/)
{{% /tip %}}

SQL injection is part of a class of vulnerabilities called **injection vulnerabilities**, and they can be used in a
wide range of scenarios, such as:

**SQL injection:**
: When writing user data into SQL queries

**Cross-site scripting (XSS):**
: When writing user data into HTML

**JSON injection:**
: When writing user data into JSON structures

**Command execution in shells:**
: When executing Linux shell commands with user data

You get the idea.

## "We'll just block it!"

The impulsive reaction of a junior developer is to just simply block things like quote marks on the input side. However,
that is not the solution because different **you need to handle data differently based on the target format**. HTML
requires much different handling than SQL or JSON.

Luckily, most modern libraries give us proper tools to *escape* untrusted input. In PHP, for example, you can use the
`htmlspecialchars()` function to encode for HTML, `json_encode()` to encode for JSON, and so on. For SQL you usually
want to use parametrized queries like this:

```php
result = query(
  "
    SELECT
      *
    FROM
      users
    WHERE
      username=?
      AND
      password=?
  ",
  username,
  password
);
```

If you have a system where that's not possible, use [prepared statements](https://en.wikipedia.org/wiki/Prepared_statement).

{{% warning %}}
**Danger!** Always process all parameters properly, even if you think it's safe. Later refactoring can open holes in your system if you don't do it.
{{% /warning %}}

{{% warning %}}
**Danger!** Don't write your own algorithm unless you are in the business of doing so. Chances are you will
be rushed and won't have enough time to do your initial research properly, or maintain it later on.
{{% /warning %}}

## Character encodings can kill you

There's one more interesting aspect to injection and it has to do with the bane of all programmers: character encodings.
Let's say you are generating a CSV file and you want to process (escape) quote marks properly, so you add a backslash
(`\`) in front of them:

```csv
"Janos","Pasztor"
"Robert\"","Tables"
```

Looks safe, right? What if someone adds a backslash into the data itself? You of course escape that too, so it becomes
a double backslash (`\\`).

However, there's an added bit of complexity that has to do with how characters are represented in computers. As long as
you only use the US alphabet, all characters can be represented in 8 bits or 1 byte of space. However, if you want to
support international characters, that is no longer sufficient. Character sets like UTF-8 use one or more bytes to 
represent a character.

![](posts/multibyte-injection.svg)

Multibyte character sets pose an added threat when dealing with injections if the data is not treated according to the
target character set. For example, let's say a multi-byte character is stored on two bytes (16 bits). If the second byte
of that 2 byte character is an US-ASCII quote mark (`"`), you may mistakenly escape it with a backslash, resulting in
a changed multi-byte character and a standalone quote mark.

In the best case scenario you end up with broken multi-byte characters. Worst case, you have an injection on your hand.
Hence, the data should always be treated in accordance with the target character set.

## Better security through good programming practices

Escaping everything everywhere is tedious work, which means that us sloppy humans are going to mess it up. In other
words, using the `mysqli_real_escape_string()` function in PHP is not the way to go.

Instead, the problem must be solved on an architecture level. We have already discussed using prepared statements or 
parameters in case of SQL, but let's reiterate: when putting together strings, you can easily use placeholders and then
replace them. Here, for example, a templated CSV:

```csv
?, ?
```

You then replace the question marks with variables, properly escaped for CSV for the target character set. If you need
something more elaborate, like a HTML template, make sure your templating engine escapes by default. This should never
result in an injection:

```html
<h1>Hello {{ name }}!</h1>
```

When you really need the raw data, you should have to work extra hard:

```html
<h1>Hello {{ myinjection | raw }}!</h1>
```

When putting together data structures like JSON, YAML, etc. it is usually a good idea to create a data structure in
the program itself, instead of treating them as a string. This can be anything from creating an array in PHP to
creating full-on object bindings using tools like [Jackson](https://github.com/FasterXML/jackson-databind).

At any rate, when performing output from your application, whether that is to the disk, into a database, to the web
browser, or executing a shell command, creating a formated set of data, **there is almost always a possibility for
injection**. Make sure you keep that in mind and write abstractions for these channels properly *once* so you never have
to deal with them again.