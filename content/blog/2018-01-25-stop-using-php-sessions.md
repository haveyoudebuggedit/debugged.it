---
slug: stop-using-php-sessions
guid: https://pasztor.at/blog/stop-using-php-sessions/
authors:
- janos
categories:
- Technology
date: "2018-01-25T00:00:00Z"
publishDate: "2018-01-25T00:00:00Z"
summary: OK, that title may be a little provocative, but PHP sessions have a fatal
  flaw that make them problematic with modern-day web applications. Let's take a look.
images:
- posts/stop-using-php-sessions.png
preview: posts/stop-using-php-sessions.jpg
tags:
- PHP
- Software Development
title: Stop using PHP sessions!
---

## How do PHP sessions work?

Here's how the web works: your browser sends a request to a server, gets a response and that's it. The server has the
most extreme case of Alzheimer you have ever seen. The next time you send a request, it is not going to know who you
are and what you did literally seconds ago. Just logged in? Who are you again?

Every HTTP request is treated as completely separate, and as strange as that may sound, that is a good thing. It enables
HTTP to be scaled over dozens, or even thousands of servers. After all, your browser doesn't have to know which Google
server it just talked to, right? Why would it need to?

However, there's a problem. Even something as simple as a login system requires the server to know who you are. It
needs to have *state* that is associated with you, the user.

That's what **sessions** are for. Let's take a look how these work:

![](posts/stop-using-php-sessions/sessions.svg)

Now, the cookie in this context is not a fancy Christmas treat, but a tiny piece of information stored in the browser. Once set, the browser sends this tiny piece of information to the server that originally set it.

As you may notice, this gives us an excellent vehicle to identify the user. Now, we don't just *trust* the data in the cookie, we have to verify it against the database. No funny business here.

## The problem with (PHP) sessions

Now, this sounds quite sensible, right? Let's take a look at how
[PHP does sessions](http://php.net/manual/en/book.session.php) in a bit more detail. At their simplest, PHP sessions can be used like this:

```php
<?php

session_start();

$_SESSION['thing'] =
  'see you next request';
```

```php
<?php

session_start();

//Outputs 'see you next request'
//if called after the first script
echo($_SESSION['thing']);
```

What you basically have is a giant bag. This bag will travel with the user as long as they come to the site within
the lifetime of the session. Normally, as a sane developer you would do this:

```php
<?php

//after login

$_SESSION['currentuser'] =
  $user;
```

This would enable you to retrieve the current user on any page you are on. Convenient, right? Ok, cool, we have this wonderful tool, let's put more stuff in it. How about information what the user viewed last? And maybe their language choice? Oh, damn, caching is too hard, let's put the translations in there...

Yeah. That's usually how sessions end up looking in PHP application. Everyone abuses it to store everything under the sun because it is terribly *convenient*. But that's not the worst, not by a long shot.

## Race conditions!

OK, so now you have developed your wonderful web application, with Angular, React and what not. It works perfectly on your developer machine, it is time to deploy. To deploy, your sysadmin installs a cluster, you switch the sessions to be stored in MongoDB (because you want to be with the cool kids on the block), and traffic starts to pour in.

Suddenly you notice something *strange*. Things start to break, and you have no idea why. When trying to reproduce the error on your dev machine, you can't. On the production system it clearly breaks... what happened?

Let's roll back to your dev machine. Here's what happens for a single request:

![](posts/stop-using-php-sessions/sessions2.svg)

That's all good, but what happens when your frontend developer fires two requests in quick succession? Let's take a look:

![](posts/stop-using-php-sessions/sessions-concurrent.svg)

OK, this might have been a bit long, but the point is that PHP 1 locks the session file so the second process (PHP 2)
has to wait for PHP 1 to finish processing the request and writing back the session data to the session file.

Now here's the interesting part, let's look at what happens in a cluster with a MongoDB session storage:

![](posts/stop-using-php-sessions/sessions-mongodb.svg)

The thing is, while **the file session backend locks the session, all other backends don't.** This means that
when *two parallel requests* happen, they **can and will overwrite each others changes** to the session data.

This isn't huge problem as long as you only store the current users ID in the session, but again, the session tends to *organically grow*, and that's putting it mildly. The more changes you make to a session, the greater the risk becomes that there will be a race condition. And double requests **do** happen, even if you don't do AJAX. Think of double clicks on submit buttons, or even your non-existent `favicon.ico` request could land on your PHP script.

## Hacks and workarounds

OK, granted, there are some workarounds. You can use `SELECT... FOR UPDATE` in MySQL to lock a session (if you don't plan to scale) or you could use so-called [spinlocks](https://en.wikipedia.org/wiki/Spinlock) with timeouts. You could even put a network filesystem under your sessions if you like getting up at 2 AM to an outage. But let's face it, these are hacks more than anything. In a world where you have a fully blown application running in the browser, allowing one request in parallel just *isn't cool any more*.

## Lower the granularity of locks

The biggest problem with PHP's session handling is that it gives you a false sense of security. It *pretends* that locks are a part of the session as a contract in the dev environment, but they really aren't. And sessions without locks are just an accident waiting to happen, especially the way PHP does them where it is incredibly easy to just store any crap in them.

Later on, when you discover that sessions are biting you in the behind, there is no easy way to get rid of them. Your application code probably just knows about one giant session object where it can store stuff in, it doesn't know how to handle a segmented data storage that you could lock separately. Nah, you need to **rewrite your whole application**. Ok, maybe not the whole, but a significant chunk of it.

So, in order to avoid this disaster, you need to **lower the lock granularity**. This means that you need to lock
only what you need. How do you do that? Well, you have a database, right? So if you want to store credentials, you
can give them a table. Say, create one table like this:

```sql
CREATE TABLE access (
  id VARCHAR(255) PRIMARY KEY,
  user_id BIGINT NOT NULL,
  expires DATETIME,
  INDEX i_user_id(user_id),
  CONSTRAINT fk_access_user_id
    FOREIGN KEY (user_id)
    REFERENCES user(id)
    ON UPDATE CASCADE
    ON DELETE CASCADE
);
```

Now, what you do is you send the user the access ID in a cookie and whenever a user comes back, you can just look it up. Do you need locks on this? Nope, not really, since you only create it once, then update the expiry time and finally delete it when needed. Totally safe.

Next up, shopping cart. This one is interesting, because if the user clicks around like crazy, strange things can
happen. So to avoid confusion, you can use *transactions* for your updates, and then always send back the current
state of the shopping cart:

```sql
START TRANSACTION;

DELETE FROM
  cart_items
WHERE
  card_id=?
  AND
  card_item_id=?;

SELECT 
  *
FROM
  cart_items
WHERE
  cart_id=?
  
COMMIT;
```

See? Again, no locks needed, just make sure the user always sees the latest state and doesn't get a mismatching update.

And so on, so forth. Yes, I'm vastly oversimplifying things, but you don't *need* a session to store all your user
state. Besides, dropping the session as a state storage also has some neat advantages, like being able to share your cart between the desktop and mobile device, etc.

If you really don't want to access the database all the time, well, you can still store a lot of stuff in cookies
and local storage. Just be sure to keep the security aspect in mind.

And if you *really* need locks, make them as granular as possible and make them explicit in your business logic.
However, it is best to avoid them if possible. You can do that by organizing your data structure in a way that makes them unnecessary, as shown in the examples above.

## It's not just PHP

Having one giant blob of data for a user on the server side is by no means just a PHP problem. However, PHP makes it particularly easy to shoot yourself in the foot. Sessions as a means of storing state are a liability in the era of client side applications. They may work for small webshops, but they are more trouble than they are worth, and I found that dropping the concept of sessions made my code a lot cleaner.

*Can* you build highly scalable sessions with distributed locking? Sure, but I hope your sysadmin likes running a
[Zookeeper](https://zookeeper.apache.org/) cluster for you.