---
slug: proper-time-handling-with-php-and-mysql
guid: https://pasztor.at/blog/proper-time-handling-with-php-and-mysql/
authors:
- janos
categories:
- Technology
date: "2013-05-07T00:00:00Z"
publishDate: "2013-05-07T00:00:00Z"
summary: Few developers actually know that not only character encodings but also time
  handling can cause you headaches when it comes to PHP and MySQL. Contrary to popular
  belief, PHP’s time handling actually works quite reasonably if you know how time
  actually works. If you don’t, you may be in for a big surprise when you add 3 days
  to a date and end up with a date 4 days from now. The answer lies within the *NIX
  time handling.
tags:
- PHP
- Software Development
title: Proper time handling with PHP and MySQL
---

## The Basics

Let’s start with the basics. (If you’re impatient, you may want to skip this section.) Hopefully all of you came to learn in elementary school that the World can be divided into 24 _timezones_ in order to provide a common sensation for morning, noon and evening. **The timezone for a country is decided by the country itself.** Not long ago for example [Samoa changed timezone](http://www.abc.net.au/news/2011-12-30/samoa-skips-friday-in-time-zone-change/3753350). To make things worse, some countries don’t shift their time by a full hour, but by fractions. French Polynesia for example is at GMT – 8.5 hours. If that wouldn’t make a developer’s life hard enough, there is also _daylight saving time_, in short DST. If you live in a DST country, you probably assume DST exists everywhere. [Far from it](http://www.worldtimezone.com/daylight.html). In case you don’t know what DST is: some countries decided to shift their timezone by one hour just for the winter time so people can get up later and thereby save electric power.

**Please note:** all examples are written for the CEST timezone, so examples may not work in your timezone.

## Time calculations in PHP

So how does this big ball of wax look in the IT world? You may be familiar with the method of converting time into _UNIX timestamps_, which is the amount of seconds since the 1<sup>st</sup> of January 1970\. in the UTC timezone (formerly: GMT) because this is a positive integer, it is easy to calculate with. Let’s calculate the time for now in 3 days. (Hint: 86400 seconds = 24 hours)

```php?start_inline=1
echo(
    date(
        "Y-m-d H:i:s",
        time()+86400*3
    )
);
```

Looks good, right? Well, not exactly. Let’s take the 23<sup>rd</sup> of march, 2012 14:00.

```php?start_inline=1
echo(
    date(
        "Y-m-d H:i:s",
        mktime(14, 0, 0, 3, 23, 2012)+86400*3
    )
);
```

What just happened? The result should have been `2012-03-26 14:00:00`, instead it is `2012-03-26 15:00:00`! In the CEST timezone there was a DST change in the night of the 24<sup>th</sup> of March, which resulted in the _one hour shift_.

It gets even worse if you round to full days, because then you’d end up with 4 days instead of 3\. If you are using such code to calculate the expiry of a service, you’re probably in for some overtime debugging, especially if you have a larger code base.

Lucky for us, PHP has a proper function: [strtotime()](http://php.net/strtotime)

```php?start_inline=1
echo(
    date(
        "Y-m-d H:i:s",
        strtotime(
            "+3 days",
            mktime(14, 0, 0, 3, 23, 2012)
        )
    )
);
```

As you can see, you just got the correct result: `2012-03-26 14:00:00`

This is all good and well, however the code still depends on the timezone set on the server. If you had the wrong timezone set, you’d end up with faulty calculations. Since server operators can do very little about this issue, **developers should set the correct timezone in their code** using the [date_default_timezone_set()](http://php.net/date_default_timezone_set) function. For example:

```php?start_inline=1
date_default_timezone_set("Europe/Vienna");
```

If you want to have somewhat cleaner calculations, you may want to use UTC timezone on your servers. This doesn’t entitle you to using timestamps however, because UTC does have leap seconds to throw your calculations off.

**Summary:** when calculating in timestamp / seconds, you take the exact time difference. IF you want a timezone / DST aware calculation, don’t use timestamps.

## Time handling in MySQL

Now that we’ve taken PHP apart, let’s look at the other suspect: MySQL. Few know, but MySQL has it’s own time handling. If you’re working with MySQL, you should definitely read about the difference between [`DATETIME` and `TIMESTAMP`](http://dev.mysql.com/doc/refman/5.1/en/datetime.html). If you read the manual, it becomes very clear that there’s a fundamental difference: `TIMESTAMP` internally converts all times to UTC from the connection’s timezone whereas `DATETIME` does not. To find out what timezone the server is set to, run the following query:

```sql
SELECT @@global.time_zone, @@session.time_zone;
```

If the server runs on default configuration, you’ll get back `SYSTEM`, which means that the MYSQL server is using the underlying operating system’s timezone settings. Unfortunately this is not directly accessible from the MySQL connection, in other words you’re left to guesswork. If you’re using [MySQL’s date and time functions](http://dev.mysql.com/doc/refman/5.1/en/date-and-time-functions.html), you can run into some nasty bugs because the timezone can be set to just about anything. (If you’re of course only using `DATETIME`, this whole thing doesn’t matter much.)

As demonstrated with PHP, MySQL is prone to the same kind of timezone issue when [DATE_ADD()](http://dev.mysql.com/doc/refman/5.1/en/date-and-time-functions.html#function_date-add) is used carelessly:

```sql
SELECT DATE_ADD(
    '2012-03-23 14:00:00',
    INTERVAL 3 DAY
);
```

If you’ve been lazy and request time in timestamps, you’re in for later bugs. **Don’t do this:**

```sql
SELECT FROM_UNIXTIME(
    UNIX_TIMESTAMP(
        '2012-03-23 14:00:00'
    )+86400*3
);
```

If you’re not working with timestamp manipulations but are using the [DATE_ADD()](http://dev.mysql.com/doc/refman/5.1/en/date-and-time-functions.html#function_date-add) function for example, that won’t work either:

```php?start_inline=1
SELECT DATE_ADD(
    '2012-03-23 14:00:00',
    INTERVAL 3 DAY
);
```

As you might have expected, you got back 15:00 in the time part.

In order to change your timezone settings in MySQL, you first need to check if the timezones have been loaded:

```sql
SELECT CONVERT_TZ(
    NOW(),
    'CET',
    'UTC'
);
```

If you’ve got back `NULL`, you need to load the timezone data. If you’re running a Linux machine, just run this command:

```bash
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql
```

Of course if you’re on a shared server, just ask your system administrator. Once this procedure is complete, the [CONVERT_TZ()](http://dev.mysql.com/doc/refman/5.1/en/date-and-time-functions.html#function_convert-tz) function will come to life and what’s more important, we’ll gain the ability to set the timezone on a connection level:

```sql
SET time_zone='UTC';
```

… and with this little magic trick the above code returns the expected result.

## tl;dr

*   Always set a PHP timezone with [date_default_timezone_set()](http://php.net/date_default_timezone_set).
*   Always set a MySQL timezone after connecting the server with `SET time_zone`.
*   **In both PHP and MySQL only use timestamp calculation if you need the exact second-interval.**
