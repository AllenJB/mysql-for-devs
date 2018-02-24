# Introduction

This document is designed to be a brief but concise guide for developers to take their MySQL knowledge beyond creating and running queries to keep their databases performant. It's a "pocket guide" that tries to summarize the information already available in the MySQL manual and cover the most common use cases, with references for those who want to delve deeper.

This guide presumes that the reader is familiar with writing SQL queries and basic table creation, but may not have gone much beyond that.

This guide is largely based on my personal experience and knowledge, backed up with references to the best materials I've found on topics. As such there are many topics that it does not cover, because I don't feel I have enough knowledge of these areas to include them here. These include views, tuning server configuration and high availability / clustered setups. This also means that some of the recommendations (such as the tools section) are similarly opinionated around the tools I use and materials I've read.

# Tools & References

There is a wide range of tools and countless blogs and tutorials related to MySQL. This section lists a selection you should consider using. Many will be referenced throughout the rest of this guide.

## Tools

- [MySQL Workbench](https://dev.mysql.com/downloads/workbench/) - A GUI client and database designer, including EER diagrams, contextual help and server status dashboard and performance reports.
- [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit) - Collection of command line utilities for analyzing and administering MySQL servers ([Documentation](https://www.percona.com/doc/percona-toolkit/LATEST/index.html))
- [Percona Monitoring & Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management) - Web-based application for on-going monitoring of MySQL and MongoDB servers. Includes a selection of preconfigured Grafana dashboards for monitoring both MySQL and the server it's on and a query analyzer tool to find and analyzer queries that are causing database load.

## References

- [Oracle MySQL Reference Docs](https://dev.mysql.com/doc/refman/5.7/en/)
- [Percona Server Reference Docs](https://www.percona.com/doc/percona-server/LATEST/index.html)
- [Rick James' MySQL Docs](http://mysql.rjweb.org/)
- [Awesome MySQL](https://shlomi-noach.github.io/awesome-mysql/) - Curated list of MySQL resources.
- [Use The Index, Luke](http://use-the-index-luke.com) - A guide to database performance for developers, not specifically focused on MySQL.
- [Join Visualizer](http://sql-joins.leopard.in.ua/)
- Book: [High Performance MySQL](http://www.highperfmysql.com/) - While lacking information from the latest MySQL releases, this is still an excellent guide.
- YouTube: [Percona](https://www.youtube.com/user/PerconaMySQL)

## Blogs & News

- [MySQL Server Team Blog](https://mysqlserverteam.com/) - The latest releases and explanations of features and changes.
- [Planet MySQL](https://planet.mysql.com/) - Aggregated blogs from the community

## Q&A Resources

- [DBA StackExchange](https://dba.stackexchange.com)
- [#mysql on Freenode IRC](irc://freenode.net/mysql)
- [#percona on Freenode IRC](irc://irc.freenode.net/percona)
- [Oracle MySQL Forums](https://forums.mysql.com/)
- [SQL Fiddle](http://sqlfiddle.com) / [DB Fiddle](https://www.db-fiddle.com/) - Useful for presenting your schema, query and sample data when asking questions

# Distributions

There are 3 main MySQL distributions in common use. This section will briefly examine each.

As to which distribution you should use, you should evaluate each for its benefits and drawbacks in relation to your project.

While there may be some implementation specific differences, the rest of this guide should apply to developers using any of these distributions.

## "Oracle" MySQL

**Website:** https://mysql.com/

**Repository:** https://github.com/mysql/mysql-server

The original MySQL distribution, first released in 1995 by MySQL AB, later purchased by Sun Microsystems, who were in turn purchased by Oracle.

The Community Edition is licensed under GPLv2, with a proprietary enterprise edition that offers additional features such as InnoDB hot backup, high-availability and clustering management.

While it has public source code repositories, these are maintained separately from the main development repository and only updated when releases are made. Security updates are released in quarterly batches.

Oracle also develop and distribute related libraries and tools, including [MySQL Workbench](https://dev.mysql.com/downloads/workbench/)

## Percona MySQL

**Website:** https://www.percona.com/software/mysql-database/percona-server

**Repository:** https://github.com/percona/percona-server

An alternative distribution, built by [Percona](https://percona.com/) (founded 2006), based on Oracle MySQL code and following the same version numbers. This distribution includes improvements focused on performance, security and stability.

Percona MySQL is released under the GPLv2 license and includes features which Oracle restrict to enterprise editions as well as additional features such as the XtraDB and TokuDB storage engines. Percona focus on additional support and services for their revenue, rather than enterprise software versions.

For more information on the changes, see the [Percona Server Documentation](https://www.percona.com/doc/percona-server/LATEST/index.html)

Percona MySQL development is considered much more open than Oracle's, with development taking place in public repositories and frequently incorporating community contributions.

Percona also develop a range of tools and monitoring solutions that can be used with all MySQL distributions, as well as distributing other databases (Mongo DB).

## MariaDB

**Website:** https://mariadb.org/

**Repository:** https://github.com/MariaDB/server

Created by some of the MySQL AB / Sun Microsystems developers who worked on what is now known as "Oracle MySQL" in response to Oracle's acquisition of the project. Similar to Percona MySQL, MariaDB originally followed Oracle MySQL's version numbers, but is now much closer to a full fork with its own version numbers and independent feature development. Starting with version 10, the project no longer implements all the features available in Oracle MySQL.

MariaDB has a community development model and is released under the GPLv2 license, with a copyright assignment requirement. It contains additional features such as new storage engines (including XtraDB).

MariaDB's development is governed by the MariaDB Foundation, which is run along-side the MariaDB Corporation who own the MariaDB trademark. MariaDB Corporation is used to fund MariaDB development with training, services and support, including MaxScale (a proxy/firewall solution for high availability / scalability).

# Data Storage (Engines)

MySQL features a number of different storage engines, with the ability to add more through a plug-in like system. This section will cover the most frequently encountered "core" storage engines.

There are more that you'll likely want to investigate, particularly as your needs grow (for example, MariaDB's ColumnStore and Percona's TokuDB) but for the large portion of users, you'll be perfectly able to get by using the core set.

## Terms and Features

### ACID Compliance

[ACID](https://en.wikipedia.org/wiki/ACID) is a set of properties of database transactions, intended to guarantee validity even in the event of errors or unexpected failures (eg. power failures).

A database that is ACID compliant should always leave the data in a valid, expected state. Queries should never be only partially successful and once a query has reported success, that change should not be lost (eg. because of a power failure while the change was still in a disk write buffer).

### Transactions

Transactions provide the ability to "ring-fence" a sequence of queries so that:

* If any query fails, an error occurs such as the connection is lost part-way through executing the sequence, or the user requests it, the entire sequence is rolled-back
* Other sessions executing at the same time will see the original version of the data until the transaction is completed

### Per-Row Locking

When updating the contents of a table, it is desirable to lock out other sessions from changing the data to ensure that conflicting changes are not executed and data integrity is maintained. Some storage engines lock the entire table to ensure this.

Other storage engines implement per-row locking, meaning that they only need to lock the rows that they are updating. This allows other sessions to perform concurrent updates on the table as long as they aren't attempting to update the same rows.

### Online ALTER TABLE Support

When modifying a table schema, to ensure data integrity, some storage engines will lock the entire table from any other changes. For large tables this can create a significant impact on the system, often requiring down time to make application changes.

Recent versions of MySQL have introduced support for online `ALTER TABLE` queries for engines that support it. In these cases MySQL allows other sessions to continue viewing and updating data in the table, allowing for schema changes without interrupting the application.

At the time of writing, not all `ALTER TABLE` changes are supported, and certain scenarios can cause online `ALTER TABLE` statements to fail (particularly if unique indexes are involved).

For more details, see [MySQL Manual: InnoDB and Online DDL](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl.html)

## MyISAM

Formerly the default engine, and as such frequently encountered, particularly in older projects, [MyISAM](https://dev.mysql.com/doc/refman/5.7/en/myisam-storage-engine.html) is now considered deprecated and should not be used. Even after the introduction of InnoDB, it was still considered the better choice for some use cases (such as read-heavy workloads), but over time these advantages have been diminished.

The MyISAM engine is not ACID compliant. It has no support for transactions and no per-row locking (meaning that updates lock the entire table). It has no support for online `ALTER TABLE` commands (the table is locked and all queries are blocked).

## InnoDB

[InnoDB](https://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html) is the default storage engine and recommended for most use cases. It is ACID compliant with support for transactions, per-row locking and online `ALTER TABLE` operations.

## XtraDB

[XtraDB](https://www.percona.com/doc/percona-server/LATEST/percona_xtradb.html) is Percona's fully backward-compatible replacement for InnoDB and is used by default when you specify "InnoDB" when using Percona Server. It's also available on [MariaDB](https://mariadb.com/kb/en/library/xtradb-and-innodb/) but is not used by default. It includes a range of scalability and performance enhancements, additional configuration and metrics.

You can check if XtraDB is enabled using: `SHOW ENGINES;`

## Memory

As its name suggests, tables using the [Memory storage engine](https://dev.mysql.com/doc/refman/5.7/en/memory-storage-engine.html) are only stored in-memory. They are not saved to disk. Memory tables are visible to all sessions (unless created as a temporary table).

All data is lost when the server halts or restarts. Memory tables are not ACID compliant, do not support transactions and have no per-row locking.

Percona features an [improved memory storage engine](https://www.percona.com/doc/percona-server/LATEST/flexibility/improved_memory_engine.html).

## Blackhole

Tables created with the [Blackhole storage engine](https://dev.mysql.com/doc/refman/5.7/en/memory-storage-engine.html) never store any rows - `INSERT` statements will succeed as normal but no data is actually inserted. Update and delete triggers are not activated. The auto-increment value (if an auto-increment column is present) never changes.

But statements are logged in MySQL's binary log (in "statement" mode - not in "row" or "mixed" mode).

Use cases for the Blackhole engine include:

* Replication where data should only be stored on some servers
  * Blackhole operations are "no op" and incur very little load on servers
* Verification of dump file syntax

## Temporary Tables

MySQL allows tables to be explicitly [created as "temporary"](https://dev.mysql.com/doc/refman/5.7/en/create-table.html#create-temporary-table). These are not to be confused with automatically created internal temporary tables used for operations such as `GROUP BY`. 

Temporary tables can use any of the "Memory", "MyISAM" or "InnoDB" engines (the default is "InnoDB"). Where the table data is stored is based on the selected storage engine.

Temporary tables are only visible to the current session and are automatically dropped when the session closes. They are not shown in `SHOW TABLES` (but Percona provides the `SHOW GLOBAL TEMPORARY TABLES` command).

A temporary table with the same name as another table will hide the existing table (for that session) until the temporary table is dropped.

You cannot refer to the same temporary table multiple times in the same query.

Temporary tables are known to have [issues with replication](https://dev.mysql.com/doc/refman/5.7/en/replication-features-temptables.html).

# Character Sets & Collations

## Character Sets

A character set defines the format in which text is stored. 

Historically there were a large number of character sets, primarily these were localized based on language - some with only minor differences, or simply extensions of previous sets, while others are completely different.

Previous versions of MySQL used the ISO-8859-1 (also known an "Latin-1") character set by default. At the time of writing, some Linux distributions still set this as the default for new installs. ISO-8859-1, as the "Latin-1" moniker suggests, is designed to only represent western European, Latin-based languages such as English and French.

In modern day computing, the de-facto standard is UTF-8, which is designed to allow the representation of most written languages, scientific and mathematical symbols and emoji. It was additionally designed to be compatible with the ASCII standard character set that was used as the basis for many other character sets, which helps to ensure that most text files should still be accessible even if opened using UTF-8.

You may also encounter UTF-16 and UTF-32, however these are less commonly used because of the lack of ASCII compatibility and the additional storage requirements.

For an abbreviated history and overview of how UTF-8 works: [YouTube: Computerphile (Tom Scott): Characters, Symbols and the Unicode Miracle](https://youtu.be/MijmeoH9LT4)

### "utf8" / "utf8mb3" vs "utf8mb4"

When they initially implemented the "utf8" character set in MySQL, the developers decided to only support the (up to 3-bytes-per-character) "Basic Multi-lingual Plane" (BMP) as a performance optimization. This supports almost all modern languages, but not historic scripts and more importantly (field specific and game) symbols, notations, emoji and other pictographic sets.

They later implemented an additional character set, "utf8mb4", which covers the entire Unicode range. They also introduced "utf8mb3" as an alias of the original "utf8" character set.

The performance advantage of the utf8mb3 character set no longer exists (in fact, the utf8mb4 character set is faster) and utf8mb4 is the recommended default, regardless of whether you may need the additionally supported characters. "utf8mb3" is considered deprecated as of MySQL 8.0

References:

* [MySQL Server Team: When to use utf8mb3 over utf8mb4](http://mysqlserverteam.com/mysql-8-0-when-to-use-utf8mb3-over-utf8mb4/) (May 2017)
* [Wikipedia: Plane (Unicode)](https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane)

### Other character sets

There are some cases where you may want to consider the use of other character sets. In particular when you have values that you know are limited to the ASCII character set, such as UUIDs, hex values or other limited base-n (such as hashes) and alpha-numeric codes (eg. ISO country or currency codes).

https://bugs.mysql.com/bug.php?id=84440

## Collations

Collations define the rules used for sorting and comparing values (including for unique constraints) when using a given character set.

### "unicode" vs "general" vs "bin"

In MySQL, collations are based on one of 3 core rule sets, denoted by one of these 3 terms appearing in the collation name.

The "unicode" collations are based on the Unicode standard rule set and take into account language-specific conversions such as contractions and ß = ss in German.

The "general" collations do not implement the full Unicode standard rule set. They contain a a number of "shortcuts" (for example, all characters in the Supplementary Multi-lingual Plane, such as emoji, are considered equal) or omissions, which means that they may not always sort as expected, particularly for non-Latin-based languages.

The "bin" collations implement "blind binary" comparison, ignoring all rules for similarity and case.

The "general" collations were originally implemented as a performance enhancement. Today the performance difference between "general" and "unicode" collations is generally considered negligible. The "bin" collations are not generally used.

### Additional Rule Variants

In addition to the base rulesets mentioned above, MySQL implements additional rule variations, denoted by abbreviations in the collation names.

"ci" rulesets implement case insensitivity for sorting and comparison. "cs" indicates a case sensitive collation.

"ai" rulesets implement accent insensitivity for sorting and comparison. This naming convention was introduced in MySQL 8.0. Legacy character sets are accent insensitive if they are case insensitive. "as" indicates an accent sensitive character set.

### References

* [MySQL Server Team: Sushi=Beer!? An Introduction to UTF8 support in MySQL 8.0](https://mysqlserverteam.com/sushi-beer-an-introduction-of-utf8-support-in-mysql-8-0/) (January 2017)
* [MySQL Server Team: MySQL 8.0.1: Accent and Case Sensitive Collations for utf8mb4](http://mysqlserverteam.com/mysql-8-0-1-accent-and-case-sensitive-collations-for-utf8mb4/) (April 2017)
* [MySQL Manual: Collation Naming Conventions](https://dev.mysql.com/doc/refman/8.0/en/charset-collation-names.html)

## Storage & Communication

When working with MySQL, there are 2 separate sets of settings relating to characters sets & collations that you should be aware of. The first is the settings used to store data in tables, while the second are the settings used to communicate with the client.

Note that while collation and character set can be separately specified, the collation implicitly sets the character set, so the latter can be safely omitted.

### Storage

Character sets and collations are set on a per-column basis in the table schema. If not explicitly specified, the column collation / character set is set from the table default. The table default collation / character set is set at table creation time (from the database default if unspecified). Similarly, the database default collation / character set is set at database creation time (from the server default if unspecified).

I would recommend that you should always explicitly specify the collation on databases and tables in schema files to avoid unexpected values.

### Connection / Communication

Whenever you create a MySQL connection, you should explicitly specify the character set you will be using to communicate with MySQL. While there are a range of settings that allow you to tweak individual aspects, as a general rule you should use `SET NAMES '<charset>'` - this command automatically sets all the related settings. It is not necessary to specify a collation on connections.

If the connection character set / collation is different from the one used for storage, MySQL will automatically try to convert values, but may not be successful. 

Example of a failure to convert between the connection and storage collations:

```
Illegal mix of collations (latin1_swedish_ci,IMPLICIT) and (utf8_general_ci,COERCIBLE) for operation '='
SELECT * FROM `site`.`users`
    WHERE `users`.`email` = 'example@yahoo.co.uk' 
    AND `users`.`password` = 'exampleÄy09' 
    AND `status` = 'active'
```

### References

* [MySQL Manual: Specifying Character Sets and Collations](https://dev.mysql.com/doc/refman/5.7/en/charset-syntax.html)
* [MySQL Manual: Connection Character Sets and Collations](https://dev.mysql.com/doc/refman/5.7/en/charset-connection.html)
* [MySQL Manual: Configuring Application Character Set and Collation](https://dev.mysql.com/doc/refman/5.7/en/charset-applications.html)

## Additional Reading

* [MySQL Server Team: Debugging Character Set Issues by Example](http://mysqlserverteam.com/debugging-character-set-issues-by-example/) (June 2017)
* [MySQL Server Team: MySQL 8.0 Collations: Migrating from Older Collations](http://mysqlserverteam.com/mysql-8-0-collations-migrating-from-older-collations/) (April 2017)

# sql_mode (or "How to Stop Silently Ignoring Problems")

Historically, MySQL has been a little bit "loose" in its adherence to the SQL standard and liberal in its handling of values. This can result in some unexpected behavior. Fortunately this has been improved in recent versions using the `sql_mode` setting.

The default (and recommended) setting as of MySQL 5.7.7 is: `NO_ENGINE_SUBSTITUTION, ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, ERROR_FOR_DIVISION_BY_ZERO, NO_ZERO_DATE, NO_ZERO_IN_DATE, NO_AUTO_CREATE_USER`

This section will briefly cover the default values. The [complete list of available sql_mode values](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html) is extensive and allows you to configure MySQL to act more like other databases or the SQL standard.

## NO_ENGINE_SUBSTITUTION

With this mode is enabled, when running `CREATE TABLE` or `ALTER TABLE` statements and a storage engine is specified that is disabled or not compiled in, error instead of substituting the default storage engine (as specified by the `default_storage_engine` server setting, which defaults to InnoDB).

## STRICT_TRANS_TABLES

Enables "strict mode" for transactional storage engines (eg. InnoDB) and when possible for non-transactional storage engines (eg. MyISAM).

With this mode enabled:

* If a value is missing in a new row to be inserted and the column has no default value
* OR if the value to be inserted into a column is invalid (eg. text in an int column)

Then error instead of substituting a value (and producing a warning, which usually gets ignored because nobody checks for MySQL warnings in their code).

Side Note: In versions 5.7.4 to 5.7.7, this mode included the behavior of the `ERROR_FOR_DIVISION_BY_ZERO`, `NO_ZERO_IN_DATE`, and `NO_ZERO_DATE` modes.

### Difference to STRICT_ALL_TABLES

The similar `STRICT_ALL_TABLES` mode unconditionally enables the strict behavior for all storage engines.

This is not usually desirable because it can mean that multi-row insert statements may fail part-way through.

`STRICT_TRANS_TABLES` will only error for non-transactional storage engines if:

* You're updating or inserting a single row
* OR you're updating or inserting multiple rows and the first row is invalid

For examples, see [Noel Herrick: Difference Between strict_all_tables and strict_trans_tables](https://www.noelherrick.com/blog/mysql-strict_all_tables-vs-strict_trans_tables)

## ERROR_FOR_DIVISION_BY_ZERO

With this mode enabled, attempting to divide by zero results in an error instead of substituting `NULL`.

## NO_ZERO_DATE

With this mode enabled, all-zero date values ('0000-00-00') are not permitted. Unless `ALLOW_INVALID_DATES` is also enabled, this mode implicitly affects invalid dates such as '2004-04-31' as these dates are converted to '0000-00-00'.

## NO_ZERO_IN_DATE

With this mode enabled, zero is not permitted as part of any date value (eg. '2001-01-00'). All-zero date values are still permitted (unless `NO_ZERO_DATE` is also enabled).

## ONLY_FULL_GROUP_BY

When enabled, if a query uses `GROUP BY` then either:

* The column must appear in the `GROUP BY` list
* OR the column must use an aggregate function (eg. `min`, `max`, `sum`, `avg`)

This behavior conforms with the SQL/92 and SQL/99 standard specifications.

Using the previous behavior, the value used for a column is non-deterministic - the server is free to choose any value for a given row. Depending on the data, this may not matter, but the server cannot know that will always be the case. In MySQL, chose values cannot be influenced by `ORDER BY` because they are selected before that is actioned.

If you don't care what value is selected or you know that the column will always contain the same value for the data set, use the `any_value()` "aggregate" function.

**Example**: 

```mysql
SELECT name, address, MAX(age) FROM t GROUP BY name
```

With this mode enabled, the above query will result in the error message: `ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'mydb.t.address' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by`

In this example, we know that the address will always be the same for any given "name" value. To fix this query, we use the `any_value` function:

```mysql
SELECT name, ANY_VALUE(address), MAX(age) FROM t GROUP BY name
```

### References

* [Gabriela.io: GROUP BY: Are You Sure You Know It?](https://blog.gabriela.io/2016/03/03/group-by-are-you-sure-you-know-it/)
* [MySQL Manual: MySQL Handling of GROUP BY](https://dev.mysql.com/doc/refman/5.7/en/group-by-handling.html)
* [MySQL Manual: sql_mode: ONLY_FULL_GROUP_BY](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by)

# Date/Time and Time Zones

In MySQL, the `date`, `time` and `datetime` types have no time zone.

The time zone that MySQL uses for dates is configurable via the `time_zone` setting. By default MySQL will use the system time zone of the server it's on and set the `time_zone` setting to the (unhelpful) value of "SYSTEM".

You can check what the current global and session `time_zone` setting with: `SELECT @@global.time_zone, @@session.time_zone;`

It's important to be aware of the time zone that MySQL uses because problems will arise if the MySQL and application time zones are different (eg. values generated in the application code will not match the values generated by MySQL's `NOW()`).

It's possible to use named time zones with MySQL, but it requires the time zone database to be manually loaded into MySQL (using `mysql_tzinfo_to_sql`) and there's no facility for automatic updates when the system time zone database is updated.

It's a common fallacy that time zones don't often change. To give you an idea of how frequently time zone definitions change, the ["Olson" zoneinfo database](https://en.wikipedia.org/wiki/Tz_database), used by most operating systems, had in some of the busier recent years: 18 revisions in 2005, 16 in 2006, 10 in 2016 and 14 in 2011.

In applications that (may) have to deal with multiple time zones, it's recommended to store all values in UTC and convert them only when needed (eg. for display). When you want to query the database, you first convert the query date/time values to UTC. 

This can be achieved in MySQL, without needing to load in the time zone database, with: `SET time_zone = +00:00;`

If you must store non-UTC values, you should always include an additional field that specifies the time zone.

When using replication, it's important to note that MySQL assumes that all servers are using the same time zone. This is another reason to always use an explicit time zone and always use UTC in MySQL. Even if all servers are using the same non-UTC time zone and their clocks are sync'd, it's possible for problems to arise if the time zone definition being used ever gets out of sync (eg. one servers time zone database package isn't updated at the same time as others).

## References

* [MySQL Manual: MySQL Server Time Zone Support](https://dev.mysql.com/doc/refman/5.7/en/time-zone-support.html)
* [MySQL Manual: Replication and Time Zones](https://dev.mysql.com/doc/refman/5.7/en/replication-features-timezone.html)
* [kdeldycke/awesome-falsehood: Dates and Time](https://github.com/kdeldycke/awesome-falsehood#dates-and-time)

# Data Types

## Strings

The field size of string fields, as specified in the schema, is measured in characters. There are 2 functions for measuring the length of string fields - `length()` measures the length in bytes, while `char_length()` measures the length in characters.

### CHAR and VARCHAR

`CHAR` fields are fixed-length string fields with a maximum length of 255 characters. The field size (as specified in the schema) is measured in characters. You can insert values less than the field length - internally they will be right-padded with spaces, but these are automatically stripped from returned records.

**Example:** Unique `CHAR` field handling of trailing spaces (result tables have been omitted for brevity)

```mysql
CREATE TABLE `test_uchar` (
	`id` int(10) unsigned auto_increment,
	`uchar` char(3),
	PRIMARY KEY `id` (`id`),
	UNIQUE INDEX `uchar` (`uchar`)
) COLLATE='utf8mb4_unicode_ci' ENGINE=InnoDB;

INSERT INTO `test_uchar` (`uchar`) VALUES ('TE');
/* Affected rows: 1  Found rows: 0  Warnings: 0  Duration for 1 query: 0.016 sec. */
INSERT INTO `test_uchar` (`uchar`) VALUES ('TE');
/* SQL Error (1062): Duplicate entry 'TE' for key 'uchar' */
/* Affected rows: 0  Found rows: 0  Warnings: 0  Duration for 0 of 1 query: 0.000 sec. */
INSERT INTO `test_uchar` (`uchar`) VALUES ('TE ');
/* SQL Error (1062): Duplicate entry 'TE' for key 'uchar' */
/* Affected rows: 0  Found rows: 0  Warnings: 0  Duration for 0 of 1 query: 0.000 sec. */
SELECT * FROM `test_uchar` WHERE `uchar` = 'TE';
/* Affected rows: 0  Found rows: 1  Warnings: 0  Duration for 1 query: 0.015 sec. */
SELECT * FROM `test_uchar` WHERE `uchar` = 'TE ';
/* Affected rows: 0  Found rows: 1  Warnings: 0  Duration for 1 query: 0.015 sec. */
```



`VARCHAR` fields are variable-length string fields with a maximum length of 65,535 characters (but beware of MySQL's [row size limits](https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html#row-size-limits) at larger values or if using multiple large fields). The size of the value stored in each record is the value + 2 bytes (or +1 byte for varchar fields with a length <= 255 characters).

**Example:** Unique `VARCHAR` field handling of trailing spaces (result tables have been omitted for brevity)

```mysql
CREATE TABLE `test_uvchar` (
	`id` int(10) unsigned auto_increment,
	`uvchar` varchar(3),
	PRIMARY KEY `id` (`id`),
	UNIQUE INDEX `uvchar` (`uvchar`)
) COLLATE='utf8mb4_unicode_ci' ENGINE=InnoDB;

INSERT INTO `test_uvchar` (`uvchar`) VALUES ('TE');
/* Affected rows: 1  Found rows: 0  Warnings: 0  Duration for 1 query: 0.016 sec. */
INSERT INTO `test_uvchar` (`uvchar`) VALUES ('TE');
/* SQL Error (1062): Duplicate entry 'TE' for key 'uvchar' */
/* Affected rows: 0  Found rows: 0  Warnings: 0  Duration for 0 of 1 query: 0.000 sec. */
INSERT INTO `test_uvchar` (`uvchar`) VALUES ('TE ');
/* SQL Error (1062): Duplicate entry 'TE ' for key 'uvchar' */
/* Affected rows: 0  Found rows: 0  Warnings: 0  Duration for 0 of 1 query: 0.000 sec. */
SELECT * FROM `test_uvchar` WHERE `uvchar` = 'TE';
/* Affected rows: 0  Found rows: 1  Warnings: 0  Duration for 1 query: 0.031 sec. */
SELECT * FROM `test_uvchar` WHERE `uvchar` = 'TE ';
/* Affected rows: 0  Found rows: 1  Warnings: 0  Duration for 1 query: 0.032 sec. */
```



### TEXT

`TEXT` are variable length string fields and come in 4 variants - `TINYTEXT`, `TEXT`, `MEDIUMTEXT` and `LONGTEXT`. `LONGTEXT` fields can [store](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html#data-types-storage-reqs-strings) up to 4GB (minus up to 4 bytes to store the length) - the number of characters that will fit into this obviously depends on the string. `TEXT` fields cannot have a default value.

| Type       | Max. Size (inc. length) | Max. Size Formula                |
| ---------- | ----------------------- | -------------------------------- |
| TINYTEXT   | 256 Bytes               | *L* + 1 bytes, where *L* < 2^8^  |
| TEXT       | 256 kB                  | *L* + 2 bytes, where *L* < 2^16^ |
| MEDIUMTEXT | 16 MB                   | *L* + 3 bytes, where *L* < 2^24^ |
| LONGTEXT   | 4 GB                    | *L* + 4 bytes, where *L* < 2^32^ |

Trailing spaces in values are not trimmed, except for the purposes of unique checks. There are restrictions on sorting and indexing `TEXT` fields (see [`max_sort_length` documentation](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_sort_length)).

If you need to perform unique checks against the contents of `TEXT` fields, consider storing a hash of the value and comparing that instead.

`TINYTEXT` should generally be avoided as it offers no advantages over an equivalent `VARCHAR` field.

`TEXT` (and `BLOB`) fields are stored separately from the normal records - They always incur a disk access and are always sorted on-disk.

#### References

* [MySQL Manual: The BLOB and TEXT types](https://dev.mysql.com/doc/refman/5.7/en/blob.html)

## Numbers

### Caveats with Large Numbers

MySQL arithmetic is done using **signed** `BIGINT` or `DOUBLE` values, so you should not use unsigned big integers larger than 9,223,372,036,854,775,807 (63 bits) except with bit functions.

You should also ensure you're aware how large numbers will work with your selected platform / language. Some languages, such as PHP, will automatically convert numbers to strings if they exceed their limits.

```mysql
SELECT (9223372036854775806 + 1) AS `answer`;
/* Affected rows: 0  Found rows: 1  Warnings: 0  Duration for 1 query: 0.047 sec. */
SELECT (9223372036854775807 + 1) AS `answer`;
/* SQL Error (1690): BIGINT value is out of range in '(9223372036854775807 + 1)' */
/* Affected rows: 0  Found rows: 0  Warnings: 0  Duration for 0 of 1 query: 0.000 sec. */
```



### INT

A whole number. The field size is the number of characters (rather than the number of bits or bytes). The field is signed unless explicitly declared `UNSIGNED`, which affects the range of available values.

If the `ZEROFILL` option is enabled, values will be left padded with zeroes.

| Type      | Bits | Char. Size |                             Signed Range |                  Unsigned Range |
| --------- | ---: | ---------: | ---------------------------------------: | ------------------------------: |
| TINYINT   |    8 |          3 |                              -128 to 127 |                        0 to 255 |
| SMALLINT  |   16 |          5 |                        -32,768 to 32,767 |                     0 to 65,535 |
| MEDIUMINT |   24 |          8 |                  -8,388,608 to 8,388,607 |                 0 to 16,777,215 |
| INT       |   32 |         10 |          -2,147,483,648 to 2,147,483,647 |              0 to 4,294,967,295 |
| BIGINT    |   64 |         20 | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | 0 to 18,446,744,073,709,551,615 |

**Note:** With the exception of `BIGINT`, add 1 to character size above for signed fields - the '-' counts as a character.

### FLOAT / DOUBLE

A floating point (imprecise) value. Like `INT` fields, they are signed unless explicitly declared `SIGNED`.

Because of their imprecision, `FLOAT` and `DOUBLE` fields can cause issues when seeding replication slaves via mysqldump.

As a general rule, due to [the problems with floating point values](https://dev.mysql.com/doc/refman/5.7/en/problems-with-float.html), you should always prefer `DECIMAL` over `FLOAT` and `DOUBLE` fields.

### DECIMAL

A precise numeric value (ie. doesn't use floats). Size is measured in characters and does not include the decimal point or the minus sign.

The size is specified as: (total digits, decimals), so a `DECIMAL(5,2)` has a range of -999.99 to 999.99 inclusive.

### AUTO_INCREMENT

Not a type, but intricately related are `AUTO_INCREMENT` values, which allow for a server controlled value that automatically increments with each record inserted.

Note that `AUTO_INCREMENT` generated values only are always unsigned, so if your field is signed you're immediately reducing the number of possible values by half.

`AUTO_INCREMENT` will not generate a value that is equal to or less than any value already in the table.

You can still manually enter values for `AUTO_INCREMENT` fields and the manually entered value will be used instead of generating a value.

While the next value can be modified by changing the `AUTO_INCREMENT` value on the table, it cannot be set to a value equal to or less than any value currently in the table. Also, when using InnoDB, this value is reset is the server is restarted.

#### How long will it take to fill up?

A quick comparison of `INT` vs `BIGINT` for (auto incrementing) id fields:

At 1 record per second, an `UNSIGNED INT` field will fill up after 4,294,967,295 seconds or 136 years, 29 weeks, 3 days, 6 hours, 28 minutes and 15 seconds.

At 1,000 records per second, it will fill up in 7 weeks, 17 hours, 2 minutes and 47 seconds.

At 1 record per second, an `UNSIGNED BIGINT` id will fill up after 18,446,744,073,709,551,615 seconds, or 586,549,402,018 years, 3 weeks, 3 days, 15 hours, 30 minutes and 7 seconds.

At 1,000,000 records per second, it will fill up in 586,549 years, 20 weeks, 6 days, 8 hours, 1 minute and 49 seconds.

If you restrict it to a signed `BIGINT` value (to avoid issues that may occur in some languages), you'll still get a good 293,274 years and change before you run into issues.

So for the vast majority of use cases you'll never need to worry about a `BIGINT` filling up.

## Binary

### BIT

An unsigned fixed width bit value. The size is measured in bits, with a maximum of 64.

Until MySQL 5.0, `BIT` was an alias for `TINYINT(1)`, but is now its own type.

`BIT` values are rarely used in practice. They have no useful functionality over `INT` types.

You have to be careful how `BIT` values are represented in queries (eg. PDO's emulated prepares assumes all values should be represented as strings)

**Example:** Value handling for `BIT` fields:

```mysql
CREATE TABLE `test_bit` (
	`id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
	`tbit` BIT(32) NOT NULL,
	PRIMARY KEY (`id`)
);
INSERT INTO test_bit (`tbit`) VALUES (1), ("1");
SELECT `id`, HEX(`tbit`) FROM test_bit;
/* Affected rows: 0  Found rows: 2  Warnings: 0  Duration for 1 query: 0.016 sec. */
+----+-----------+
| id | HEX(tbit) |
+----+-----------+
|  1 | 1         |
|  2 | 31        |
+----+-----------+
```



### BINARY and VARBINARY

The `BINARY` and `VARBINARY` types are basically "binary strings" that are treated purely as binary values. The size of these fields is measured in bytes, with a maximum of 65,535.

`BINARY` values are right-padded with 0x00 (null byte).

Comparison and sorting are based on numeric values of the bytes.

### BLOB

The Binary Large OBject types are a "binary string". This column type cannot have a default value. Like `TEXT`, it comes in 4 variants - `TINYBLOB`, `BLOB`, `MEDIUMBLOB` and `LONGBLOB` - and value size limits are the same as for the equivalent `TEXT` fields.

The same notes from the `TEXT` types apply equally to `BLOB` fields.

## ENUM

`ENUM` columns are a string object with a single value from a list of permitted values.

Internally, values are automatically encoded as numbers and decoded on output, allowing for efficient storage and comparison while avoiding "magic numbers" in the database.

You're unlikely to hit the limits on the number of permitted values, but obviously as the size of this list increases, the complexity of managing the list increases.

### Numeric Values

You should avoid using ENUM with numerical values.

Using the following simple schema:

```mysql
CREATE TABLE `test_enum` (
  `numbers` ENUM('0','1','2')
)
```

First let's show how the values are stored internally:

| String value     | Internal index |
| ---------------- | -------------: |
| NULL             |           NULL |
| '' (error value) |              0 |
| '0'              |              1 |
| '1'              |              2 |
| '2'              |              3 |

As you can see, there's a clear mismatch between the internal value and the string value. This mismatch might not be caused by a simple off-by-one difference as above, but might be caused by future manipulation of the permitted values list.

If we then insert some values into the table, then select them:

```mysql
INSERT INTO `test_enum` (`numbers`) VALUES (2),('2'),('3');
SELECT * FROM `test_enum`;
+---------+
| numbers |
+---------+
| 1       |
| 2       |
| 2       |
+---------+
```

Note that you'll get no warnings or errors performing these queries, even with strict sql_mode enabled.

As you can see, the results are probably not what you would expect. Where the quotes weren't used on the first record, MySQL stored that directly, and it's then mapped back to the string value of '1' in the results. Where we used a value that should have been illegal, MySQL decided we meant the internal index value instead, mapping the value back to '2' in the results.

### Replacing Items in the Value List

The behavior you most have to be aware of with `ENUM` fields is how to correctly change the value list without unexpectedly changing the values in the database.

The following examples use the following schema and initial data:

```mysql
CREATE TABLE `test_enum2` (
	`rowid` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
	`options` ENUM('red','blue','green') NOT NULL,
	PRIMARY KEY (`rowid`)
);

INSERT INTO `test2_enum2` (`options`) VALUES ('red', 'blue', 'green');
```

Which gives us the following data set:

| rowid | options |
| ----- | ------- |
| 1     | "red"   |
| 2     | "blue"  |
| 3     | "green" |

#### The Wrong Way

We want to replace the 'green' option with 'yellow', so we run the following query:

```mysql
ALTER TABLE `test2_enum2` CHANGE COLUMN `options` `options` ENUM('red','blue','yellow');
```

If we're running with strict sql_mode, we'll get an error:

`SQL Error (1265): Data truncated for 'options' at row 3`

If we're not running strict mode, the query will "succeed", but if we select our data, we'll now see:

| rowid | options |
| ----- | ------- |
| 1     | "red"   |
| 2     | "blue"  |
| 3     | ""      |

The "green" value became invalid, as was replaced with the "" (empty string) error value. We've essentially corrupted the data in our database.

We have no way of reversing this action - We can't simply update all records with the "" value to "yellow" because it's possible existing records also had the "" value.

#### The Right Way

So what some might consider the most obvious method to replace values didn't work as expected. 

Starting from our initial schema and data, the correct way to handle this is a 3-step process:

```mysql
ALTER TABLE `test_enum2`
	CHANGE COLUMN `options` `options` ENUM('red','blue','green','yellow');

UPDATE `test_enum2` SET `options` = 'yellow' WHERE `options` = 'green';

ALTER TABLE `test_enum2`
	CHANGE COLUMN `options` `options` ENUM('red','blue','yellow');
```

Now if we select the table data, we see:

| rowid | options  |
| ----- | -------- |
| 1     | "red"    |
| 2     | "blue"   |
| 3     | "yellow" |

## NULL

While not a distinct column type, the `NULL` value has some properties that you should definitely be aware of.

### NULL != NULL

Null is not equal to itself. 

This means that if you have a nullable field `f` and you select rows `WHERE f = null`, you will always get no results.

To find rows with `NULL` values, you need to instead select rows `WHERE f IS NULL`.

### UNIQUE checks

Stemming from the fact that `NULL != NULL` is that `NULL` is always considered a unique value.

Consider the following scenario:

```mysql
CREATE TABLE `test_null_unique` (
  `c1` INT NOT NULL,
  `c2` INT,
  `c3` INT NOT NULL,
  UNIQUE KEY `c1_c2` (`c1`, `c2`)
);

INSERT INTO `test_null_unique` (`c1`, `c2`, `c3`)
VALUES (1, NULL, 0), (2, NULL, 0), (2, NULL, 0), (3, 1, 0);
```

If we select all rows, we'll see:

| c1   | c2   | c3   |
| ---- | ---- | ---- |
| 1    | NULL | 0    |
| 2    | NULL | 0    |
| 2    | NULL | 0    |
| 3    | 1    | 0    |

This affects all unique checks, so if we then execute an `ON DUPLICATE KEY UPDATE`:

```mysql
INSERT INTO `test_null_unique` (`c1`, `c2`, `c3`)
VALUES (2, NULL, 0)
ON DUPLICATE KEY UPDATE `c3` = `c3` + 1;
```

We now see a new row added, instead of the value on an existing row updated:

| c1   | c2   | c3   |
| ---- | ---- | ---- |
| 1    | NULL | 0    |
| 2    | NULL | 0    |
| 2    | NULL | 0    |
| 3    | 1    | 0    |
| 2    | NULL | 0    |

### References

* [Modern SQL: NULL - Indicating the Absence of Data](https://modern-sql.com/concept/null)
* [Modern SQL: The Three Valued Logic of SQL](https://modern-sql.com/concept/three-valued-logic)

# Other Advice

## Avoiding SELECT *

`SELECT *` is one of those things you'll frequently get told not to do. `SELECT *` is stupid we're told. But `SELECT *` works. And if it's stupid but it works, it's not *that* stupid, right?

A lot of the time, as with many such "rules", many people can go a long time without anyone ever explaining *why* it exists.

There are several cases when `SELECT *` should be explicitly avoided:

* The table contains `TEXT` or `BLOB` fields (or any of their variants)
  In addition to the fact that these fields often contains large amounts of data, they are stored "off page", separately from the rest of the row data. Extra work has to be performed when retrieving them. If you didn't really need these fields, this extra work is unnecessary.
* All the fields you need are in an index
  If all the values required are contained in the index used, MySQL can completely skip examining the table data and pull the values straight from the index instead.
* Joining multiple tables
  When joining multiple tables, you often don't need some or all of the fields from one of the tables (perhaps you're joining across a link table that contains extra data in the rows, such as the link record creation date, that isn't needed for the results processing). Not selecting these reduces the size of the temporary table that is implicitly created. This makes the data more likely to fit into memory, speeding up queries.
* Queries that use `GROUP BY`
  When you use `GROUP BY`, if the field is not explicitly in the `GROUP BY` clause, and doesn't use an aggregate function, then the value you get should be considered random. If the `ONLY_FULL_GROUP_BY` sql_mode is enabled, then the query won't be allowed. For more information, read up on `ONLY_FULL_GROUP_BY` in the sql_mode section.

