- [Introduction](#orgb35a19c)
- [WAL- theory](#org2c01682)
  - [How WAL Works](#org1748e7a)
  - [Checkpointing](#org5b10655)
  - [Notes](#org7b659e2)
- [Config Pragmas](#orgfdb6f28)
  - [Cache Size](#org6e05f19)
  - [WAL Auto Checkpoint](#org48cc90f)
  - [Max Page Count](#org9c1ff9f)
- [Basic APIs](#orgc4576ff)
  - [sqlite3<sub>open</sub>()](#org91c1eb6)
  - [sqlite3<sub>prepare</sub><sub>v2</sub>()](#orgcbf4441)
  - [sqlite3<sub>step</sub>()](#org7cafc58)
  - [sqlite3<sub>column</sub><sub>xx</sub>()](#orgdf4c1f3)
  - [sqlite3<sub>finalize</sub>()](#org7385fcf)
  - [sqlite3<sub>close</sub>()](#orge28ba10)
  - [sqlite3<sub>exec</sub>() - Convenience wrapper](#orge11ae0c)
  - [VFS](#org047c3d0)
- [Internals](#org731f74c)
  - [Query and Frontend](#org4780c42)
  - [Backend](#org75b0654)
  - [Cursor Abstraction](#orgdc37832)
  - [Data Structures](#org815b0de)
  - [B-Trees](#orgffe90d0)
  - [B+ Trees](#orgb0e8903)
  - [Single Column Indexes](#orgb4456aa)
  - [Multi-Column Indexes](#org49836fd)


<a id="orgb35a19c"></a>

# Introduction

The principal task of an SQL database engine is to evaluate SQL statements of SQL. To accomplish this, the developer needs two objects:

-   The database connection object: sqlite3
-   The prepared statement object: sqlite3<sub>stmt</sub>


<a id="org2c01682"></a>

# WAL- theory


<a id="org1748e7a"></a>

## How WAL Works

The traditional rollback journal works by writing a copy of the original unchanged database content into a separate rollback journal file and then writing changes directly into the database file. In the event of a crash or ROLLBACK, the original content contained in the rollback journal is played back into the database file to revert the database file to its original state. The COMMIT occurs when the rollback journal is deleted.

The WAL approach inverts this. The original content is preserved in the database file and the changes are appended into a separate WAL file. A COMMIT occurs when a special record indicating a commit is appended to the WAL. Thus a COMMIT can happen without ever writing to the original database, which allows readers to continue operating from the original unaltered database while changes are simultaneously being committed into the WAL. Multiple transactions can be appended to the end of a single WAL file.


<a id="org5b10655"></a>

## Checkpointing

One wants to eventually transfer all the transactions that are appended in the WAL file back into the original database. Moving the WAL file transactions back into the database is called a "checkpoint".

Another way to think about the difference between rollback and write-ahead log is that in the rollback-journal approach, there are two primitive operations, reading and writing, whereas with a write-ahead log there are now three primitive operations: reading, writing, and checkpointing.

<span class="underline">By default, SQLite does a checkpoint automatically when the WAL file reaches a threshold size of 1000 pages.</span> (The SQLITE<sub>DEFAULT</sub><sub>WAL</sub><sub>AUTOCHECKPOINT</sub> compile-time option can be used to specify a different default.) Applications using WAL do not have to do anything in order to for these checkpoints to occur. But if they want to, applications can adjust the automatic checkpoint threshold. Or they can turn off the automatic checkpoints and run checkpoints during idle moments or in a separate thread or process.


<a id="org7b659e2"></a>

## Notes

<span class="underline">The WAL file is created when the first connection to the database is opened and is normally removed when the last connection to the database closes. \_ However, if the last connection does not shutdown cleanly, the WAL file will remain in the filesystem and will be automatically cleaned up the next time the database is opened.</span>

-   Beginning in SQLite version 3.7.4, WAL databases can be created, read, and written even if shared memory is unavailable as long as the locking<sub>mode</sub> is set to EXCLUSIVE before the first attempted access. In other words, a process can interact with a WAL database without using shared memory if that process is guaranteed to be the only process accessing the database.


<a id="orgfdb6f28"></a>

# Config Pragmas


<a id="org6e05f19"></a>

## Cache Size

PRAGMA schema.cache<sub>size</sub>; PRAGMA schema.cache<sub>size</sub> = pages; PRAGMA schema.cache<sub>size</sub> = -kibibytes; [Default upto 2MB of pages are cached in memory]

Query or change the suggested maximum number of database disk pages that SQLite will hold in memory at once per open database file. An application defined page cache can choose to ignore this setting. The default page cache that is built into SQLite honors the request. The default suggested cache size is -2000, which means the cache size is limited to 2048000 bytes of memory. The default suggested cache size can be altered using the SQLITE<sub>DEFAULT</sub><sub>CACHE</sub><sub>SIZE</sub> compile-time options. The TEMP database has a default suggested cache size of 0 pages.

If the argument N is positive then the suggested cache size is set to N. If the argument N is negative, then the number of cache pages is adjusted to be a number of pages that would use approximately abs(N\*1024) bytes of memory based on the current page size. SQLite remembers the number of pages in the page cache, not the amount of memory used. So if you set the cache size using a negative number and subsequently change the page size (using the PRAGMA page<sub>size</sub> command) then the maximum amount of cache memory will go up or down in proportion to the change in page size.

The default page cache implemention does not allocate the full amount of cache memory all at once. Cache memory is allocated in smaller chunks on an as-needed basis. The page<sub>cache</sub> setting is a (suggested) upper bound on the amount of memory that the cache can use, not the amount of memory it will use all of the time. This is the behavior of the default page cache implementation, but an applicaction defined page cache is free to behave differently if it wants.


<a id="org48cc90f"></a>

## WAL Auto Checkpoint

PRAGMA wal<sub>autocheckpoint</sub>=N;

This pragma queries or sets the write-ahead log auto-checkpoint interval. When the write-ahead log is enabled (via the journal<sub>mode</sub> pragma) a checkpoint will be run automatically whenever the write-ahead log equals or exceeds N pages in length. Setting the auto-checkpoint size to zero or a negative value turns auto-checkpointing off.

Autocheckpointing is enabled by default with an interval of 1000 or SQLITE<sub>DEFAULT</sub><sub>WAL</sub><sub>AUTOCHECKPOINT</sub>.


<a id="org9c1ff9f"></a>

## Max Page Count

PRAGMA schema.max<sub>page</sub><sub>count</sub>; PRAGMA schema.max<sub>page</sub><sub>count</sub> = N;

Query or set the maximum number of pages in the database file. Both forms of the pragma return the maximum page count. The second form attempts to modify the maximum page count. The maximum page count cannot be reduced below the current database size.


<a id="orgc4576ff"></a>

# Basic APIs


<a id="org91c1eb6"></a>

## sqlite3<sub>open</sub>()

-   Opens a connection to the database and returns an sqlite3 database object.


<a id="orgcbf4441"></a>

## sqlite3<sub>prepare</sub><sub>v2</sub>()

This routine converts SQL text into a prepared statement object and returns a pointer to that object. This interface requires a database connection pointer created by a prior call to sqlite3<sub>open</sub>() and a text string containing the SQL statement to be prepared. This API does not actually evaluate the SQL statement. It merely prepares the SQL statement for evaluation.

Think of each SQL statement as a small computer program. The purpose of sqlite3<sub>prepare</sub>() is to compile that program into object code.

SQLite allows the same prepared statement to be evaluated multiple times. This is accomplished using the following routines:

-   sqlite3<sub>reset</sub>()
-   sqlite3<sub>bind</sub>()

After a prepared statement has been evaluated by one or more calls to sqlite3<sub>step</sub>(), it can be reset in order to be evaluated again by a call to sqlite3<sub>reset</sub>(). Think of sqlite3<sub>reset</sub>() as rewinding the prepared statement program back to the beginning. Using sqlite3<sub>reset</sub>() on an existing prepared statement rather than creating a new prepared statement avoids unnecessary calls to sqlite3<sub>prepare</sub>(). For many SQL statements, the time needed to run sqlite3<sub>prepare</sub>() equals or exceeds the time needed by sqlite3<sub>step</sub>(). So avoiding calls to sqlite3<sub>prepare</sub>() can give a significant performance improvement.

sqlite3<sub>bind</sub>() attaches values to the parameters. Each call to sqlite3<sub>bind</sub>() overrides prior bindings on the same parameter.


<a id="org7cafc58"></a>

## sqlite3<sub>step</sub>()

This routine is used to evaluate a prepared statement that has been previously created by the sqlite3<sub>prepare</sub>() interface. The statement is evaluated up to the point where the first row of results are available. To advance to the second row of results, invoke sqlite3<sub>step</sub>() again. Continue invoking sqlite3<sub>step</sub>() until the statement is complete. Statements that do not return results (ex: INSERT, UPDATE, or DELETE statements) run to completion on a single call to sqlite3<sub>step</sub>().


<a id="orgdf4c1f3"></a>

## sqlite3<sub>column</sub><sub>xx</sub>()

This routine returns a single column from the current row of a result set for a prepared statement that is being evaluated by sqlite3<sub>step</sub>(). Each time sqlite3<sub>step</sub>() stops with a new result set row, this routine can be called multiple times to find the values of all columns in that row.

They can be something like: sqlite3<sub>column</sub><sub>blob</sub>() sqlite3<sub>column</sub><sub>bytes</sub>() sqlite3<sub>column</sub><sub>bytes16</sub>() sqlite3<sub>column</sub><sub>count</sub>() sqlite3<sub>column</sub><sub>double</sub>() sqlite3<sub>column</sub><sub>int</sub>() sqlite3<sub>column</sub><sub>int64</sub>()

-   <span class="underline">The columns returned correspond to the order in which we requested them in the SELECT statement above</span>


<a id="org7385fcf"></a>

## sqlite3<sub>finalize</sub>()

This routine destroys a prepared statement created by a prior call to sqlite3<sub>prepare</sub>(). Every prepared statement must be destroyed using a call to this routine in order to avoid memory leaks.


<a id="orge28ba10"></a>

## sqlite3<sub>close</sub>()

This routine closes a database connection previously opened by a call to sqlite3<sub>open</sub>(). All prepared statements associated with the connection should be finalized prior to closing the connection.


<a id="orge11ae0c"></a>

## sqlite3<sub>exec</sub>() - Convenience wrapper

To run an SQL statement, the application follows these steps:

-   Create a prepared statement using sqlite3<sub>prepare</sub>().
-   Evaluate the prepared statement by calling sqlite3<sub>step</sub>() one or more times.
-   For queries, extract results by calling sqlite3<sub>column</sub>() in between two calls to sqlite3<sub>step</sub>().
-   Destroy the prepared statement using sqlite3<sub>finalize</sub>().

The sqlite3<sub>exec</sub>() interface is a convenience wrapper that carries out all four of the above steps with a single function call. A callback function passed into sqlite3<sub>exec</sub>() is used to process each row of the result set. The sqlite3<sub>get</sub><sub>table</sub>() is another convenience wrapper that does all four of the above steps. The sqlite3<sub>get</sub><sub>table</sub>() interface differs from sqlite3<sub>exec</sub>() in that it stores the results of queries in heap memory rather than invoking a callback.


<a id="org047c3d0"></a>

## VFS

int sqlite3<sub>vfs</sub><sub>register</sub>(sqlite3<sub>vfs</sub>\*, int makeDflt); int sqlite3<sub>vfs</sub><sub>unregister</sub>(sqlite3<sub>vfs</sub>\*);

A virtual filesystem (VFS) is an sqlite3<sub>vfs</sub> object that SQLite uses to interact with the underlying operating system. Most SQLite builds come with a single default VFS that is appropriate for the host computer. New VFSes can be registered and existing VFSes can be unregistered. The following interfaces are provided.


<a id="org731f74c"></a>

# Internals


<a id="org4780c42"></a>

## Query and Frontend

A query goes through a chain of components in order to retrieve or modify data. The front-end consists of the:

-   tokenizer
-   parser
-   code generator

The input to the front-end is a SQL query. The output is sqlite virtual machine bytecode (essentially a compiled program that can operate on the database).


<a id="org75b0654"></a>

## Backend

The back-end consists of the:

-   Virtual Machine
-   B-tree
-   Pager
-   Os Interface

-   The virtual machine takes bytecode generated by the front-end as instructions. It can then perform operations on one or more tables or indexes, each of which is stored in a data structure called a B-tree. The VM is essentially a big switch statement on the type of bytecode instruction.

-   Each B-tree consists of many nodes. <span class="underline">Each node is one page in length</span>. The B-tree can retrieve a page from disk or save it back to disk by issuing commands to the pager.

-   The pager receives commands to read or write pages of data. It is responsible for reading/writing at appropriate offsets in the database file. It also keeps a cache of recently-accessed pages in memory, and determines when those pages need to be written back to disk.

-   The os interface is the layer that differs depending on which operating system sqlite was compiled for. In this tutorial, I’m not going to support multiple platforms.


<a id="orgdc37832"></a>

## Cursor Abstraction

Cursor object represents a location in the table. They are like iterators in c++ collections.

Things you might want to do with cursors:

-   Create a cursor at the beginning of the table
-   Create a cursor at the end of the table
-   Access the row the cursor is pointing to
-   Advance the cursor to the next row

Later, we will also want to:

-   Delete the row pointed to by a cursor
-   Modify the row pointed to by a cursor
-   Search a table for a given ID, and create a cursor pointing to the row with that ID

A cursor generally also has a reference to the table it’s part of (so our cursor functions can take just the cursor as a parameter).

-   Finally, it has a boolean called end<sub>of</sub><sub>table</sub>. This is so we can represent a position past the end of the table (which is somewhere we may want to insert a new row).


<a id="org815b0de"></a>

## Data Structures

BTree for Index in Sqlite B+Tree for tables in Sqlite


<a id="orgffe90d0"></a>

## B-Trees

-   keeps keys in sorted order for sequential traversing
-   uses a hierarchical index to minimize the number of disk reads Allows most of the tree to be kept in RAM
-   uses partially full blocks to speed insertions and deletions

-   Unlike a hash map it allows traversing a range of values in a quicker fashion.
-   Unlike a binary tree, each node in a B-Tree can have more than 2 children. Each node can have up to m children, where m is called the tree’s “order”. To keep the tree mostly balanced, we also say nodes have to have at least m/2 children (rounded up).

-   Exceptions:
    -   Leaf nodes have 0 children
    -   The root node can have fewer than m children but must have at least 2
    -   If the root node is a leaf node (the only node), it still has 0 children


<a id="orgb0e8903"></a>

## B+ Trees

A B+ tree can be viewed as a B-tree in which each node contains only keys (not key–value pairs), and to which an additional level is added at the bottom with linked leaves.

The primary value of a B+ tree is in storing data for efficient retrieval in a block-oriented storage context — in particular, filesystems. This is primarily because unlike binary search trees, B+ trees have very high fanout (number of pointers to child nodes in a node,[1] typically on the order of 100 or more), which reduces the number of I/O operations required to find an element in the tree.

-   Internal nodes store values No (can be changing now)
-   Nodes with children are called “internal” nodes. Internal nodes and leaf nodes are structured differently:
-   Num keys in BTree is m-1. In B+ Trees num keys is set to as many as will fit.


<a id="orgb4456aa"></a>

## Single Column Indexes


<a id="org49836fd"></a>

## Multi-Column Indexes

While creating an index that has multiple columns, SQLite uses the additional columns as the second, third, … sort keys.

SQLite sorts the data on the multicolumn index by the first column specified in the CREATE INDEX statement. Then it sorts the duplicate values by the second column, and so on.

The following statement creates a multicolumn index on the first<sub>name</sub> and last<sub>name</sub> columns of the contacts table: `CREATE INDEX idx_contacts_name ON contacts (first_name, last_name);`

SQLite uses Btree for index
