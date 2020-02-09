# Log Structured Merge Trees
## Links
* [Log Structured Merge Trees - ben stopford](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/)
* [Compaction in apache cassandra](http://www.datastax.com/dev/blog/leveled-compaction-in-apache-cassandra)
* [The Log: What every software engineer should know about real-time data’s unifying abstraction | LinkedIn Engineering](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
* [SSTable and Log Structured Storage: LevelDB - igvita.com](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)
* [Turning the database inside-out with Apache Samza | Hacker News](https://news.ycombinator.com/item?id=9145197)

They show that, somewhat counter intuitively, sequential disk access is faster than randomly accessing main memory

![](Log Structured Merge Trees/ChartGo.png)

The fastest way to have sequential writes is a simple append-only file suitable for journals: however, this is useful only for replication and message queues. 

Obviously, to perform more complex workloads like key based search or range scans, we need more than just a sequential log.  To do this, there are basically 4 approaches:

1. Sorted File 
	* Sort the data and save it in a file
		* If the data has fixed width, you can use binary search
		* If not, we have to use a file index and scan
2. Hash and split into buckets
	* Hash the data into buckets, and save each bucket as a file
	* This file can then be scanned for reads
3. B+ Trees or [ISAM](https://en.wikipedia.org/wiki/ISAM)
4. External File
	* Leave the data as a log/heap and create a separate hash/index into it

In all the above approaches, data is stored in the filesystem and indexed so that it can be retrieved. Thus, you have two different structures : the data file and indexed data (which might also be serialized into the disk).

When data needs to be updated,  updates have to be made in specific places in the hash file in [2] above or the B+ Tree files in [3]  above. This requires slow, random IO. This in-place update is slow and limiting.

Another solution is to have an in-memory hash index into the journal file (log file). This works pretty well as the random access happens in memory. This approach is used in products like Riak and Oracle Coherence.

![](Log Structured Merge Trees/Journal6.png)

## Base log structured merge trees Algorithm 
#log structured merge trees/base algorithm#
Instead of one large file index (index into a journal/log file) which can cause significant delays in write or scatter-gun the filesystem, batches of writes are written sequentially to a smaller set of index files.

* Each file contains a batch of changes covering a short period of time
* Each file is sorted before its written so that searching later is fast
* Files are immutable; never updated : duplicate entries are created in the newer files that will supersede previous entries/records in older files
* Newer updates go into newer files
* **Reads inspect all the files**
* Periodically, files are merged together into a new file

### Read 
#log structured merge trees/read#
* When a read is requested, the in-memory data structure holding the latest writes is checked (`memtable`).
* If that is a miss, then each sorted file in reverse chronological order is inspected, until the requested key is found.
* This process (of inspecting/reading multiple sorted files till the key is found) is very slow unless compaction of files is done
* To improve this read performance, an in-memory index to the file is maintained : this provides a lookup _close_ to the key value being looked up. Since the file is sorted, this lookup isn’t very expensive.
* As the number of these files increases, even this approach will become expensive. Most implementations avoid the read to visit multiple files through the use of a **bloom filter**

### Compaction 
#log structured merge trees/compaction#
Compaction is the process that reduces the number of sorted files and resolves duplicates and deletions. This helps to keep the reads performant (read performance degrades as the number of files to inspect increases without compaction).

#### Compaction Algorithm
#log structured merge trees/compaction/basic#
* To keep log structured merge trees read relatively fast, number of files must be reduced
* When a certain number of files (say, 5) each have reached a minimum size (say, 10 rows each), they are merged into a single file (in this example, containing max. 50 rows)
* As more and more 10 row files are created, this process continues. 
* Eventually, there will be 5  merged files (each containing ~50 rows). At this point, these 5 files will be merged into one 250 row file.
#log structured merge trees/compaction/level based#

The above approach (to merging) was __size__ based. LevelDB, RocksDB and Cassandra implement the merge based on “levels” that optimize reads by guaranteeing uniqueness of a key in a file per level (i.e, each key, if exists, will exist only in a single file per level) **except** for the first level (where writes are batched and hence sequential).

> Files are merged into upper levels one file at a time. As a level fills, a single file is plucked from it and merged into the file for the level above it.  

### Sequential Read Vs Sequential Write Tradeoff 
- All writes are batched and written only in sequential chunks
* There is additional IO during compaction
* Reads are not sequential : multiple reads across many files must be done (i.e, `scatter-gun` reads happen)
* So in a Log Structured Merge Trees, we trade off sequential writes for sequential reads.
* With in-memory file indexes and bloom filters, we can optimize the read performance

![](Log Structured Merge Trees/Journal31.png)


#log structured merge trees#
