---
slug: logging-php-errors-to-syslog
guid: https://pasztor.at/blog/logging-php-errors-to-syslog/
authors:
- janos
categories:
- Technology
date: "2012-04-26T00:00:00Z"
publishDate: "2012-04-26T00:00:00Z"
summary: Ever so often I get to set up hosts for running PHP. When running a load
  balanced solution, you have more hosts and reading logs gets complicated, development
  gets tedious. So what helps, is a central logging server. This is pretty easy to
  set up with syslog-ng, however PHP has a annoying habit of logging everything with
  the NOTICE error level.
tags:
- DevOps
- PHP
title: Logging PHP errors to syslog-ng
---

What we need to do is change the error level. Unfortunately the syslog-ng OSE documentation states:

> The following macros in syslog-ng OSE are hard macros and cannot be modified: BSDTAG, CONTEXT_ID, DATE, DAY, FACILITY_NUM, FACILITY, FULLDATE, HOUR, ISODATE, LEVEL_NUM, **LEVEL**, MIN, MONTH_ABBREV, MONTH_NAME, MONTH, MONTH_WEEK, , PRIORITY, PRI, SDATA, SEC, SEQNUM, SOURCEIP, STAMP, TAG, TAGS, TZOFFSET, TZ, UNIXTIME, WEEK_DAY_ABBREV, WEEK_DAY_NAME, WEEK_DAY, WEEK, YEAR_DAY, YEAR.

So we can’t just change the level using a rewrite. There is a small trick to circumvent this limitation. The message has to leave syslog-ng, be rewritten externally and re-enter it. The easiest way to do this is the _logger_ utility. First we create a filter, that matches PHP error messages:

```
filter f_php_error {
    facility(user) and
    message("PHP (Parse|Compile|Fatal|Core) error");
};
```

Next we need to create the logger destination to rewrite the message:

```
destination d_logger_error {
    program("/usr/bin/logger -p user.err -t php" template("$MSG\n"));
};
```

Note, that the logger is only launched once and the messages are sent to `STDIN`.

Finally we create our log loop. We need to use the `final` flag to stop processing the message by the normal log paths. This also means, that the log path needs to be around the top of the config file before the other log paths. Of course exchange the source to match your source specification for `/dev/log`.

```
log {
    source(s_all);
    filter(f_php_error);
    destination(d_logger_error);
    flags(final);
};
```

What’s left are the warnings. It’s the same story:

```
filter f_php_warning {
    facility(user) and
    message("PHP (|User) warning");
};

destination d_logger_warning {
    program("/usr/bin/logger -p user.warning -t php" template("$MSG\n"));
};

log {
    source(s_all);
    filter(f_php_warning);
    destination(d_logger_warning);
    flags(final);
};
```

There you go, you have all the messages with correct error level and the errors pop out when using a coloring syslog viewer.