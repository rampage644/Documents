## WHY?

+ Make ZeroVM execution transactional
+ Provide `revert` ability to user-space filesystem

## HOW?

+ Utilize append only ZeroVM channels (_CDR_)
+ Filesystem on top of 1 file

## Possible solutions

+ Porting CoW filesystem (like `ZFS`, `btrfs`)
+ Porting Log-Structured filesystem (there are some, no production usage though)
+ Construct filesystem from `LMDB`+`SQLightning`+`libsqlfs`
+ Build filesystem on top of any appendable KV storage (see Contastine post at google groups as example)

## Proposed storage stack

Use `LMDB` as KV storage. Build filesystem on top of it. Use `libsqlfs` as possible filesystem implementation. Use `SQLightning` as adapter between them.

Arised problems:

1. `SQLightning` project is very immature. Tons of tests are failing. Spent 2 hours trying to get the whole test suite run (by excluding failing tests) with no success.
2. `libsqlfs` + `SQLightning` + `LMDB` doesn't work (should i say _out-of-the-box_?) at host.
3. `LMDB` _is_ Copy-on-Write, but it is not appendable. I managed to hack it to become truely appendable, but it disables some features (see [here](https://symas.com/getting-down-and-dirty-with-lmdb-qa-with-symas-corporations-howard-chu-about-symass-lightning-memory-mapped-database/) question about database size increasing). 

### Solution:

Have to invest effort (_no estimation!_) to make `SQLightning` stable. Dig deep into `sqlite3` (btree/locks mechanism) + master `lmdb`.

That will just get make proposed stack work (I hope). I think we could face following problems afterwards:
1. `lmdb` uses `mmap` and friends to manage pages (memory <-> HDD). We will definitely have some issues due to lack of `mmap` support. However, `lmdb` seems to work under ZeroVM in a simple case.
2. DB size increase very rapidly. We will again face with `mmap` issues cause we have to load whole database into memory instead of just necessary pages.
3. External support for rollback/versioning. I mean who will supply adequate database to untrusted?


### SQLightning benchmark

#### SQL expressions

Using `tool/speedtest.tcl` with some modifications (just to make sqlite3 run instead of mysql/plsql and unkonwn sqlite248). 

Running 4 tests: 

```sql
CREATE TABLE t1(a INTEGER, b INTEGER, c VARCHAR(100));
INSERT INTO t1 VALUES(1,13153,'thirteen thousand one hundred fifty three');
INSERT INTO t1 VALUES(2,75560,'seventy five thousand five hundred sixty');
... 995 lines omitted
INSERT INTO t1 VALUES(998,66289,'sixty six thousand two hundred eighty nine');
INSERT INTO t1 VALUES(999,24322,'twenty four thousand three hundred twenty two');
INSERT INTO t1 VALUES(1000,94142,'ninety four thousand one hundred forty two');
```

```sql
BEGIN;
CREATE TABLE t2(a INTEGER, b INTEGER, c VARCHAR(100));
INSERT INTO t2 VALUES(1,298361,'two hundred ninety eight thousand three hundred sixty one');
... 24997 lines omitted
INSERT INTO t2 VALUES(24999,447847,'four hundred forty seven thousand eight hundred forty seven');
INSERT INTO t2 VALUES(25000,473330,'four hundred seventy three thousand three hundred thirty');
COMMIT;
```

```sql
SELECT count(*), avg(b) FROM t2 WHERE b>=0 AND b<1000;
SELECT count(*), avg(b) FROM t2 WHERE b>=100 AND b<1100;
SELECT count(*), avg(b) FROM t2 WHERE b>=200 AND b<1200;
... 94 lines omitted
SELECT count(*), avg(b) FROM t2 WHERE b>=9700 AND b<10700;
SELECT count(*), avg(b) FROM t2 WHERE b>=9800 AND b<10800;
SELECT count(*), avg(b) FROM t2 WHERE b>=9900 AND b<10900;
```

```sql
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%one%';
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%two%';
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%three%';
... 94 lines omitted
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%ninety eight%';
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%ninety nine%';
SELECT count(*), avg(b) FROM t2 WHERE c LIKE '%one hundred%';
```


| Test                | SQLightning, [s] | SQLite3, [s] |
| ------------------- | ---------------- | ------------ |
| INSERTs             | 2.26             | 5.18         |
| INSERTs transaction | 0.234            | 0.236        |
| SELECT w/o index    | 0.423            | 0.455        |
| SELECT w/ strings   | 1.29             | 1.40         |


### Benchmarking utility

Used source from [here][lmdb-benchmark]. Database file reside at tmpfs which is inmemory filesystem.

Raw results are:

```
SQLite:     version 3.7.17 Original
Date:       Wed Mar 19 17:51:03 2014
CPU:        2 * Intel(R) Core(TM) i7-3520M CPU @ 2.90GHz
CPUCache:   6144 KB
Keys:       16 bytes each
Values:     100 bytes each
Entries:    1000000
RawSize:    110.6 MB (estimated)
------------------------------------------------
fillseqsync  :      23.178 micros/op;    4.8 MB/s  
156 /run/tmpdb/dbbench_sqlite3-1.db
3736    /run/tmpdb/dbbench_sqlite3-1.db-wal
fillrandsync :      28.000 micros/op;    4.0 MB/s  
156 /run/tmpdb/dbbench_sqlite3-2.db
3816    /run/tmpdb/dbbench_sqlite3-2.db-wal
fillseq      :      11.519 micros/op;    9.6 MB/s     
154928  /run/tmpdb/dbbench_sqlite3-3.db
4200    /run/tmpdb/dbbench_sqlite3-3.db-wal
fillseqbatch :       5.653 micros/op;   19.6 MB/s     
154928  /run/tmpdb/dbbench_sqlite3-4.db
4304    /run/tmpdb/dbbench_sqlite3-4.db-wal
fillrandom   :      17.117 micros/op;    6.5 MB/s     
154536  /run/tmpdb/dbbench_sqlite3-5.db
4204    /run/tmpdb/dbbench_sqlite3-5.db-wal
fillrandbatch :      13.308 micros/op;    8.3 MB/s    
154564  /run/tmpdb/dbbench_sqlite3-6.db
5568    /run/tmpdb/dbbench_sqlite3-6.db-wal
overwrite    :      26.446 micros/op;    4.2 MB/s     
201476  /run/tmpdb/dbbench_sqlite3-6.db
5568    /run/tmpdb/dbbench_sqlite3-6.db-wal
readrandom   :       9.232 micros/op;                 
readseq      :       2.897 micros/op;   32.9 MB/s     
readreverse  :       2.793 micros/op;   34.1 MB/s  
------------------------------------------------
SQLite:     version 3.7.17 LMDB-based
Date:       Wed Mar 19 17:59:14 2014
CPU:        2 * Intel(R) Core(TM) i7-3520M CPU @ 2.90GHz
CPUCache:   6144 KB
Keys:       16 bytes each
Values:     100 bytes each
Entries:    1000000
RawSize:    110.6 MB (estimated)
------------------------------------------------
fillseqsync  :      19.733 micros/op;    5.6 MB/s  
156 /run/tmpdb/dbbench_sqlite3-1.db
3736    /run/tmpdb/dbbench_sqlite3-1.db-wal
fillrandsync :      25.439 micros/op;    4.3 MB/s  
156 /run/tmpdb/dbbench_sqlite3-2.db
3816    /run/tmpdb/dbbench_sqlite3-2.db-wal
fillseq      :      11.758 micros/op;    9.4 MB/s     
154928  /run/tmpdb/dbbench_sqlite3-3.db
4200    /run/tmpdb/dbbench_sqlite3-3.db-wal
fillseqbatch :       5.677 micros/op;   19.5 MB/s     
154928  /run/tmpdb/dbbench_sqlite3-4.db
4304    /run/tmpdb/dbbench_sqlite3-4.db-wal
fillrandom   :      17.957 micros/op;    6.2 MB/s     
154536  /run/tmpdb/dbbench_sqlite3-5.db
4204    /run/tmpdb/dbbench_sqlite3-5.db-wal
fillrandbatch :      13.073 micros/op;    8.5 MB/s    
154564  /run/tmpdb/dbbench_sqlite3-6.db
5568    /run/tmpdb/dbbench_sqlite3-6.db-wal
overwrite    :      27.327 micros/op;    4.0 MB/s     
201476  /run/tmpdb/dbbench_sqlite3-6.db
5568    /run/tmpdb/dbbench_sqlite3-6.db-wal
readrandom   :       9.260 micros/op;                 
readseq      :       2.873 micros/op;   33.2 MB/s     
readreverse  :       2.888 micros/op;   33.0 MB/s 
```

No lmdb-based advantage found. Feel free to hold your own experiment.


## Prototypes

I coded two very primitive filesystem prototypes:

+ `lmdb`-based 
+ log structered one (just ensure every write is sequential)

Filesystems are written in python with `python-fuse`. For source code see [my github repo][repo]

Filesystem supports only read/write operations and no directory support (that is, every file is in a root directory).

Some thoughts:

+ Problems are the same: architecure of filesystem organization.
+ Main difference is how data is stored: sequentially written to some file or somehow by lmdb (but also sequntially).
+ And lmdb wastes much more space, filesystem file grows faster (no _quantitive_ experiment was held).

Don't get me wrong: I'm not saying lmdb (or other KV storage) in inappropriate for filesystem implementation, but in will require much more complicated design to implement full-featured filesystem.

## Resume

My resume:

+ If we really need appendable filesystem, log structred ones are better suited for it. We could port something or just use existing design.
+ I don't really understand the need in KV storage for filesystem base. Will appreciate if someone explains me this.

## Links

+ LMDB interesting [article](http://banksco.de/p/lmdb-the-leveldb-killer.html)
+ LMDB [benchmarking][lmdb-benchmark] 
+ My github repo [here][repo] 


[repo]: https://github.com/rampage644/appendfs
[lmdb-benchmark]: http://symas.com/mdb/microbench/

