---
layout: post
title:  "Mysql Pandamonium"
date:   2015-12-15 12:51:10
categories:
author: laurelfan
---

So we were doing some scaling work to prepare for <a href="https://hourofcode.com">Hour of Code</a> last week. Part of this was looking at the mysql server logs for slow queries.

We saw some stuff like this taking a long time:

````
select count(*) from users where username='tanya\U+1F43C' or email='tanya\U+1F43C'
````

This query happens every time someone tries to log in -- it looks for a row in the User table matching the username or email that the user entered. Obviously this is a really common query and needs to be fast so it should be optimized. Since it's such a simple query this is done by indexing the username and email columns. It would be really weird if these columns weren't indexed -- that's pretty much the obvious thing to index on that table. Well, when things get weird, ask the database to explain itself:

````
mysql> explain select count(*) from users where username='tanya\U+1F43C' or email='tanya\U+1F43C' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
         type: ALL
possible_keys: index_users_on_username_and_deleted_at,index_users_on_email_and_deleted_at
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 10529368
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
````

For those who aren't familiar with databases or mysql, this is an explain plan. Relational databases have an *optimizer* that takes a query and tries to figure out the fastest way to run it. You can ask the optimizer what it is thinking using the <a href="https://dev.mysql.com/doc/refman/5.5/en/execution-plan-information.html">explain</a> (aka explain plan) command. The weird thing here -- these indexes were created at the time that we created these tables -- we r not dum (ok, not in this instance). And you can tell by the `possible_keys` section of the plan that it found the indexes, but if you look at `rows`, you can tell that it decided not to use them.

What's weird about this query: that \U thing going on there is 🐼. Yes, that's right, it's a panda face emoji. Could that be confusing our database? Here's the explain plan for the exact same query with a more normal string:

````
mysql> explain select count(*) from users where username='no_panda' or email='no_panda' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
         type: index_merge
possible_keys: index_users_on_username_and_deleted_at,index_users_on_email_and_deleted_at
          key: index_users_on_username_and_deleted_at,index_users_on_email_and_deleted_at
      key_len: 768,767
          ref: NULL
         rows: 2
        Extra: Using sort_union(index_users_on_username_and_deleted_at,index_users_on_email_and_deleted_at); Using where
1 row in set (0.01 sec)
````

Yep. That plan above is using the indexes just like we wanted to (compare `key` and `rows` with above). So what is interesting about our cute and deadly friend 🐼? It takes 4 characters in UTF-8. UTF-8 is designed so that the most "common" characters take fewer bytes -- ASCII is one byte, Latin-1 is two bytes, most of Chinese is 3 bytes. Characters that take 4 bytes include less common languages and also most of the not-so-rare emoji. Mysql detail here: the users table was created with the utf8 charset (which only supports up to 3 byte utf8) not the <a href="https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html">utf8mb4</a> charset. I'd researched that already because we knew that we couldn't store emoji in emails and usernames (from getting exceptions in <a href="http://honeybadger.io">Honeybadger</a>), but I didn't know it was also causing a perf problem. It's super weird that it does.

There's an easy way to <a href="https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-upgrading.html">migrate the charset</a> to allow those characters to be stored (and I would guess that it would fix the index problem also), but it had in the past been low priority (since you probably don't actually have a panda in your email address, and usernames are now automatically generated without special characters). It would be the correct way to fix this but changing the database configuration like this was a little high risk to do during the hour of code.

So, to work around it we had to basically look for strings with utf8mb4 characters and return early before trying to read or write to the database (we knew we'd never find any of these values since it was impossible to write them). You can check out <a href="https://github.com/code-dot-org/code-dot-org/pull/5994">PR#5994</a> if you're interested in the gory details.
