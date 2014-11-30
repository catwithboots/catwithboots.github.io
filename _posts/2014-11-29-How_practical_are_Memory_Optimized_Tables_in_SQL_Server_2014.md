---
layout: post
category: sqlserver
title: How practical are Memory Optimized Tables in SQL Server 2014
tagline: by Cihat Genç
tags: [sqlserver, microsoft]
share: true
---
As most of you probably already know, with the launch of SQL Server 2014 a new feature called “In-Memory OLTP” (aka Hekaton) is introduced. As the name already suggests the idea is to run your database completely in memory. The possible benefits are pretty obvious; speed! 

<!--more-->

A part of my daily work is to tackle performance issues on SQL Server, thus this feature sounded very interesting. Since I’m just starting to discover it’s possibilities, I figured why not share it with you.

Hekaton is a part of the SQL Engine, so there is no need for installation of additional features. It is already there when you install the SQL Server engine. Just one note, as you can guess, it is unfortunately only an enterprise feature. I won’t get into much detail how Hekaton works because there are quite some blog posts about that already, but I will try to explain the basics before getting to how practical it is at the moment. 

The whole concept of Hekaton is to eliminate the performance bottlenecks encountered with the traditional disk based tables like locks, latches (a lightweight lock-like mechanism that protects access to read and change in memory structures) and disk I/O. 

**Locks and latches**

Hekaton is using an optimistic MVCC (Multi Version Concurrency Control) model, what is already there for some time in Oracle and a bit in SQL Server as of version 2005 with snapshot isolation. Also Hekaton will never modify an existing row, so there is no need for locking. Instead data modification (update) is done by deleting the existing row and inserting a new row. If a row is updated multiple times you will get multiple versions of the row. So the result of a query will depend on the transaction start time and the timestamp in the header of the row.
So what about latches? Simplified a latch is a lightweight in memory lock that makes sure that the (meta)data pages in memory are not changed by another transaction. With In-Memory OLTP the concept of pages does not exist (it just stored rows in memory, no pages), with this latches don't exist either.

**Disk I/O**

This is a part that needs to be explained a little bit. Since the database is in memory you don't have to do any disk I/O, but there is a nuance. All metadata is compiled code  and held in memory (including tables/indexes and natively compiled stored procedures). For the actual data you have two options;


1. Schema_only are non-durable objects, so basically this is completely in memory and with a crash or restart of SQL Server you will lose the data in these objects (not the objects themselves). 

2. Then you have schema_and_data objects which are durable. You will keep the data in case of a SQL Server restart. This means the transaction log will be used and that the data will be streamed out sequentially to disk (not a traditional MDF/NDF layout) in order of the transaction log using a background thread to ensure recovery. 

**Requirements and constraints**

Since Hekaton is handling metadata and data in a different way than traditional disk based data there are quite some constraints for using this. A few of the important ones in my opinion are; 

•	Datatype restrictions: Lob/max, clr and xml are not allowed

•	Row length:  Limited to 8060k

•	Metadata: Once a metadata object (table/index) is created you cannot alter it. This mean if you need to change the table or index, you need to create a new one and migrate all existing data to the new tables.

•	You need to have an idea how big your table will be, because the memory will be pre reserved and not be able to grow beyond that amount.

•	No DML triggers allowed (which is actually not a bad thing imho)

•	Identity columns only with SEED and increment of 1

•	Constraints: Foreign keys and check constraints are not allowed. A primary key is required (for durable tables). No unique indexes allowed (except the one created automatically with the primary key). No more than 8 indexes allowed (which is also actually not a bad thing imho)

•	A column that is used in an index is required to be in Windows BIN2 collation

•	Once an in-memory filegroup is created you cannot drop it from the database.

**So, how practical is it then?**

I'll answer this with one of the most annoying answers; it depends.

You have to consider the limitations carefully. If you need features which are not available in memory optimized tables than you can't make use of it unless you change your table structure. This is in some cases possible and in some not. 

You can combine in-memory tables with disk based tables. So if there are some tables that meet the requirements and some not you can still make use of it for those tables.

I personally think the current version of Hekaton is very useful for processing data for web-caching/ETL or maybe even an alternative for temporary tables/table variables. Due to the limitations you cannot just convert your disk based database into an in-memory database. Therefore the use of this feature will be relatively limited. I've spoken with multiple people close to Microsoft at SQL Pass 2014 and all of them say the same. Microsoft is aware of the limitations and most probably with a next version this feature will become more mature and hopefully have less limitations.

For my own projects I will investigate if there would be a possible benefit of using Hekaton. Will keep you posted!
