大致介绍了一下Cassandra的存储机制，通过将最新的写操作放在内存中的Memtable，然后定期刷新到磁盘持久化为 SSTable，Cassandra将随机写操作转换成了顺序写操作，这可以提升IO性能。

最新写入的脏数据是在内存Memtable表中，因此必须有机制来确保异常情况下，能够将内存中的数据恢复出来。和关系型数据库系统一样，Cassandra也是采用的先写日志再写数据的方式，其日志称之为Commitlog。

和Memtable/SSTable不一样的是，Commitlog是server级别的，不是Column Family级别的 。每个Commitlog文件的大小是固定的，称之为一个Commitlog Segment，目前版本(0.7)中，这个大小是128MB，这是硬编码在代码(src\java\org\apache\cassandra \db\Commitlog.java)中的。当一个Commitlog文件写满以后，会新建一个的文件。当旧的Commitlog文件不再需要时，会自动清除。

每个Commitlog文件(Segment)都有一个固定大小（大小根据Column Family的数目而定）的CommitlogHeader 结构，其中有两个重要的数组，每一个Column Family在这两个数组中都存在一个对应的元素。其中一个是位图数组(BitSet dirty )，如果Column Family对应的Memtable中有脏数据，则置为1，否则为0，这在恢复的时候可以指出哪些Column Family是需要利用Commitlog进行恢复的。另外一个是整数数组(int[.md](.md) lastFlushedAt )，保存的是Column Family在上一次Flush时日志的偏移位置，恢复时则可以从这个位置读取Commitlog记录。通过这两个数组结构，Cassandra可以在异常重启服务的时候根据持久化的SSTable和Commitlog重构内存中Memtable的内容，也就是类似Oracle等关系型数据库的实例恢复。

当Memtable flush到磁盘的SStable时，会将所有Commitlog文件的dirty数组对应的位清零，而在Commitlog达到大小限制创建新的文件时，dirty数组会从上一个文件中继承过来。如果一个Commitlog文件的dirty数组全部被清零，则表示这个Commitlog在恢复的时候不再需要，可以被清除。因此，在恢复的时候，所有的磁盘上存在的Commitlog文件都是需要的。

参考文章：
[1](1.md).http://wiki.apache.org/cassandra/ArchitectureCommitLog
[2](2.md).http://blog.csdn.net/starxu85/archive/2010/03/20/5399180.aspx


---


org.apache.cassandra.db.commitlog (Commitlog-Current millisecond.log)

CommitLogHeader.java
```
// 用一个map就代表了位图数组，整数数组的功能
private Map<Integer, Integer> cfDirtiedAt; // position at which each CF was last flushed


返回Commit log header path
String getHeaderPathFromSegment(CommitLogSegment segment);
String getHeaderPathFromSegmentPath(String segmentPath);

包含两个方法: serialize, deserialize; 其中serialize用来将CommitLogHeader写入到header文件中，格式为：the number of key-value mappings in this map, the checksum of key-value mappings in this map, key#0, value#0, key#1, value#1 ....; 而deserialize按照这个序列读取commit log header，构造数据结构CommitLogHeader.

static CommitLogHeaderSerializer serializer = new CommitLogHeaderSerializer();
    void serialize(CommitLogHeader clHeader, DataOutput dos);
    CommitLogHeader deserialize(DataInput dis);

// test if the column family id was in map(id => position)
boolean isDirty(Integer cfId);

// get the cfId's position(at which each CF was last flushed);
int getPosition(Integer cfId);

// put the (cfId, position) into map
void turnOn(Integer cfId, long position);

// remove cfId from map
void turnOff(Integer cfId);

// test if map empty
boolean isSafeToDelete();

// CLH(dirty+flushed={key#0:value#0,key#1:value#1, ........})
String toString();

// key#0,key#1,key#2 ....
String dirtyString();

// call serialize
void writeCommitLogHeader(CommitLogHeader header, String headerFile);

// call deserialize
CommitLogHeader readCommitLogHeader(String headerFile);

// 返回所有column families中最小的last flush position
int getReplayPosition();
```

CommitLogSegment.java
```

// point to the commitlog file: "CommitLog-" + System.currentTimeMillis() + ".log"
private final BufferedRandomAccessFile logWriter; 

private final CommitLogHeader header;

// check if the filename is the commit log
boolean possibleCommitLogFile(String filename);

// write CommitLogHeader into disk, the file name is xxx.header
void writeHeader();

// 用BufferedRandomAccessFile来代表文件，以后所有要对文件的操作都针对它了 
BufferedRandomAccessFile createWriter(String file);

// 这个应该是CommitLogSegment中最重要的方法了，写log操作
// 1) update header
// 2) write mutation, w/ checksum on the size and data (length, the checksum of length, value, the checksum of value)
CommitLogSegment.CommitLogContext write(RowMutation rowMutation, Object serializedRow)
```

CommitLog.java
```
/*
 * Commit Log tracks every write operation into the system. The aim
 * of the commit log is to be able to successfully recover data that was
 * not stored to disk via the Memtable. Every Commit Log maintains a
 * header represented by the abstraction CommitLogHeader. The header
 * contains a bit array and an array of longs and both the arrays are
 * of size, #column families for the Table, the Commit Log represents.
 *
 * Whenever a ColumnFamily is written to, for the first time its bit flag
 * is set to one in the CommitLogHeader. When it is flushed to disk by the
 * Memtable its corresponding bit in the header is set to zero. This helps
 * track which CommitLogs can be thrown away as a result of Memtable flushes.
 * Additionally, when a ColumnFamily is flushed and written to disk, its
 * entry in the array of longs is updated with the offset in the Commit Log
 * file where it was written. This helps speed up recovery since we can seek
 * to these offsets and start processing the commit log.
 *
 * Every Commit Log is rolled over everytime it reaches its threshold in size;
 * the new log inherits the "dirty" bits from the old.
 *
 * Over time there could be a number of commit logs that would be generated.
 * To allow cleaning up non-active commit logs, whenever we flush a column family and update its bit flag in
 * the active CL, we take the dirty bit array and bitwise & it with the headers of the older logs.
 * If the result is 0, then it is safe to remove the older file.  (Since the new CL
 * inherited the old's dirty bitflags, getting a zero for any given bit in the anding
 * means that either the CF was clean in the old CL or it has been flushed since the
 * switch in the new.)
 */
// roll after log gets this big
int SEGMENT_SIZE = 128*1024*1024; 

// CommitLog was made up many CommitLogSegment
Deque<CommitLogSegment> segments = new ArrayDeque<CommitLogSegment>();

// how to deal with commit log, period or bash ..
ICommitLogExecutorService executor;

// change the segment size
void setSegmentSize(int size);

// get the number of CommitLogSegment
int getSegmentCount();

void recover();

/*
 * Adds the specified row to the commit log. This method will reset the
 * file offset to what it is before the start of the operation in case
 * of any problems. This way we can assume that the subsequent commit log
 * entry will override the garbage left over by the previous write.
 */
// update commit log header and write the data serializedRow into the last of CommitLogSegment
void add(RowMutation rowMutation, Object serializedRow);

/*
 * This is called on Memtable flush to add to the commit log
 * a token indicating that this column family has been flushed.
 * The bit flag associated with this column family is set in the
 * header and this is used to decide if the log file can be deleted.
 */
void discardCompletedSegments(final Integer cfId, final CommitLogSegment.CommitLogContext context);
 
                 |
                 |
                 V
/**
 * Delete log segments whose contents have been turned into SSTables. NOT threadsafe.
 *
 * param @ context The commitLog context .
 * param @ id id of the columnFamily being flushed to disk.
 *
 */
private void discardCompletedSegmentsInternal(CommitLogSegment.CommitLogContext context, Integer id) {
    /*
     * Loop through all the commit log files in the history. Now process
     * all files that are older than the one in the context. For each of
     * these files the header needs to modified by resetting the dirty
     * bit corresponding to the flushed CF.
     */
    // 处理当前commitlog所有的CommitLogSegment
    Iterator<CommitLogSegment> iter = segments.iterator(); 
    while (iter.hasNext())
    {
        CommitLogSegment segment = iter.next();
        // header决定了当前正在处理的segment属于哪个commit log
        CommitLogHeader header = segment.getHeader(); 

            // 最后一个需要处理的Segment, 记住最后flush的position
            if (segment.equals(context.getSegment())) 
            {
                // we can't just mark the segment where the flush happened clean,
                // since there may have been writes to it between when the flush
                // started and when it finished. so mark the flush position as
                // the replay point for this CF, instead.
                if (logger.isDebugEnabled())
                    logger.debug("Marking replay position " + context.position + " on commit log " + segment);
                header.turnOn(id, context.position);
                segment.writeHeader();
                break;
            }

            // 因为CF已经被flush了，所以这个segment对应的commit log header对应的bit置1
            header.turnOff(id);

            // 当一个commit log的CommitLogHeader中的bitset都置0了，表示可以删除这个commit log了
            if (header.isSafeToDelete()) 
            {
                segment.close(); // close the commit log
                DeletionService.submitDelete(segment.getHeaderPath()); // delete the commit log and commig log header
                DeletionService.submitDelete(segment.getPath());
                // usually this will be the first (remaining) segment, but not always, if segment A contains
                // writes to a CF that is unflushed but is followed by segment B whose CFs are all flushed.
                iter.remove();
            }
            else
            {
                segment.writeHeader(); // 因为header已经更新了，所以需要重新写入到disk中
            }
        }

}

```
AbstractCommitLogExecutorService.java
BatchCommitLogExecutorService.java
BatchCommitLogExecutorServiceMBean.java
ICommitLogExecutorService.java
PeriodicCommitLogExecutorService.java
PeriodicCommitLogExecutorServiceMBean.java

需要先了解一下上层的实现，特别是Memtable和数据结构Column families!