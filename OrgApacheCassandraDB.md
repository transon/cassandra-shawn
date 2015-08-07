  * IColumn.java
```
Column和SuperColumn的接口，里面有Column和SuperColumn都要用到的方法
```

  * Column.java
```
// 一个Column包含这三个fields，这个是Cassandra数据模型中的最小单位了
protected final byte[] name;
protected final byte[] value;
protected final IClock clock;

// 返回这个Column的field对应的值
public byte[] name();
public byte[] value();
public IClock clock();

// 一个Column的大小
public int size()
{
    /*
     * Size of a column is =
     *   size of a name (short + length of the string)
     * + 1 byte to indicate if the column has been deleted
     * + x bytes depending on IClock size
     * + 4 bytes which basically indicates the size of the byte array
     * + entire byte array.
     */
    return DBConstants.shortSize_ + name.length + DBConstants.boolSize_ + clock.size() + DBConstants.intSize_ + value.length;
}

public boolean isMarkedForDelete()
{
    return false;
}

```

  * DeletedColumn.java
```
继承了Column，唯一的差别就是传递给构造函数的value是localDeletionTime
```

  * ExpiringColumn.java
```
继承了Colomn，另外定义下面的两个变量
// (int) (System.currentTimeMillis() / 1000) + timeToLive
private final int localExpirationTime; 
private final int timeToLive;

@Override
public boolean isMarkedForDelete()
{
    return (int) (System.currentTimeMillis() / 1000 ) > localExpirationTime;
}

// Column的size + sizeof(timeToLive) + sizeof(localExpirationTime)
@Override
public int size()
{
    /*
     * An expired column adds to a Column : 
     *    4 bytes for the localExpirationTime
     *  + 4 bytes for the timeToLive
     */
    return super.size() + DBConstants.intSize_ + DBConstants.intSize_;
}
```

  * SuperColumn.java
```
// 可以看成是SuperColumn的索引，通过name_可以在cf's map快速找到它
private byte[] name_; 

// SuperColumn存储数据的地方, IColumn可以是Column，也可以是SuperColumn，其他就可以把name_ => columns_看做是(name, value)
private ConcurrentSkipListMap<byte[], IColumn> columns_;

// delete标记，决定SuperColumn是否被删除
private AtomicInteger localDeletionTime = new AtomicInteger(Integer.MIN_VALUE);
private AtomicReference<IClock> markedForDeleteAt;
private AbstractReconciler reconciler;

public boolean isMarkedForDelete()
{
    IClock _markedForDeleteAt = markedForDeleteAt.get();
    return _markedForDeleteAt.compare(_markedForDeleteAt.type().minClock()) == ClockRelationship.GREATER_THAN;
}

/**
 * This calculates the exact size of the sub columns on the fly
 */
public int size()
{
    int size = 0;
    for (IColumn subColumn : getSubColumns())
    {
        size += subColumn.serializedSize(); // serializedSize（）是Column/DColumn/EColumn的
    }
    return size;
}

/**
 * This returns the size of the super-column when serialized.
 * @see org.apache.cassandra.db.IColumn#serializedSize()
*/
public int serializedSize() //  serializedSize调用size
{
	/*
	 * We need to keep the way we are calculating the column size in sync with the
	 * way we are calculating the size for the column family serializer.
	 */
  IClock _markedForDeleteAt = markedForDeleteAt.get();
  return DBConstants.shortSize_ + name_.length + DBConstants.intSize_ + _markedForDeleteAt.size() + DBConstants.intSize_ + size();
}

public void addColumn(IColumn column)
{
    assert column instanceof Column : "A super column can only contain simple columns";

    byte[] name = column.name();
    IColumn oldColumn = columns_.putIfAbsent(name, column);
    if (oldColumn != null)
    {
        IColumn reconciledColumn = reconciler.reconcile((Column)column, (Column)oldColumn);
        while (!columns_.replace(name, oldColumn, reconciledColumn))
        {
            // if unable to replace, then get updated old (existing) col
            oldColumn = columns_.get(name);
            // re-calculate reconciled col from updated old col and original new col
            reconciledColumn = reconciler.reconcile((Column)column, (Column)oldColumn);
            // try to re-update value, again
        }
	}
}

/*
 * Go through each sub column if it exists then as it to resolve itself
 * if the column does not exist then create it.
 */  
public void putColumn(IColumn column) // column是SuperColumn!!!
{
    assert column instanceof SuperColumn;

    for (IColumn subColumn : column.getSubColumns())
    {
    	addColumn(subColumn);
    }
    FBUtilities.atomicSetMax(localDeletionTime, column.getLocalDeletionTime()); // do this first so we won't have a column that's "deleted" but has no local deletion time
    FBUtilities.atomicSetMax(markedForDeleteAt, column.getMarkedForDeleteAt());
}

```

  * ColumnFamily.java
```
private final Integer cfid; // cf id
private final ColumnFamilyType type; // cf type
private final ClockType clockType;
private final AbstractReconciler reconciler; //  how to resolve the conflict 

private transient ICompactSerializer2<IColumn> columnSerializer;
final AtomicReference<IClock> markedForDeleteAt;
final AtomicInteger localDeletionTime = new AtomicInteger(Integer.MIN_VALUE);

// cf的data都放在这里，byte[]可以看做是IColumn的索引，而IColumn可以是Column, SuperColumn
// 如果是Column，则byte[]是name， 如果是SuperColumn，则byte[]是name_
// <byte[], IColumn>中的IColumn仅仅是row_key对应的value,row_key没有存放在这里
private ConcurrentSkipListMap<byte[], IColumn> columns; 

/**
 * @return The CFMetaData for this row, or null if the column family was dropped.
 */
public CFMetaData metadata()
{
    return DatabaseDescriptor.getCFMetaData(cfid);
}

// 把这个ColumnFamily的所有IColumn添加到当前cf中去
public void addAll(ColumnFamily cf);

// 返回这个cf中有多少IColumn, Returns the number of key-value mappings in this map
int getColumnCount();

public void addColumn(QueryPath path, byte[] value, IClock clock)
{
    assert path.columnName != null : path;
    addColumn(path.superColumnName, new Column(path.columnName, value, clock));
}

public void addTombstone(QueryPath path, byte[] localDeletionTime, IClock clock)
{
    addColumn(path.superColumnName, new DeletedColumn(path.columnName, localDeletionTime, clock));
}

public void addColumn(QueryPath path, byte[] value, IClock clock, int timeToLive)
{
    assert path.columnName != null : path;
    Column column;
    if (timeToLive > 0)
        column = new ExpiringColumn(path.columnName, value, clock, timeToLive);
    else
        column = new Column(path.columnName, value, clock);
    addColumn(path.superColumnName, column);
}

public void deleteColumn(byte[] column, int localDeletionTime, IClock clock)
{
    addColumn(null, new DeletedColumn(column, localDeletionTime, clock));
}

public void deleteColumn(QueryPath path, int localDeletionTime, IClock clock)
{
    assert path.columnName != null : path;
    addColumn(path.superColumnName, new DeletedColumn(path.columnName, localDeletionTime, clock));
}

public void addColumn(byte[] superColumnName, Column column)
{
    IColumn c;
    if (superColumnName == null)
    {
        c = column;
    }
    else
    {
        assert isSuper();
        c = new SuperColumn(superColumnName, getSubComparator(), clockType, reconciler);
        c.addColumn(column); // checks subcolumn name
    }
    addColumn(c);
}

/*
 * If we find an old column that has the same name
 * the ask it to resolve itself else add the new column .
*/
public void addColumn(IColumn column)
{
    byte[] name = column.name();
    IColumn oldColumn = columns.putIfAbsent(name, column);
    if (oldColumn != null)
    {
        if (oldColumn instanceof SuperColumn)
        {
            ((SuperColumn) oldColumn).putColumn(column);
        }
        else
        {
            // calculate reconciled col from old (existing) col and new col
            IColumn reconciledColumn = reconciler.reconcile((Column)column, (Column)oldColumn);
            while (!columns.replace(name, oldColumn, reconciledColumn))
            {
                // if unable to replace, then get updated old (existing) col
                oldColumn = columns.get(name);
                // re-calculate reconciled col from updated old col and original new col
                reconciledColumn = reconciler.reconcile((Column)column, (Column)oldColumn);
                // try to re-update value, again
            }
        }
    }
}

```

  * Row.java
```
public final DecoratedKey key;
public final ColumnFamily cf; // 其实就是数据模型中的columns所处的位置
```

  * Table.java
```
// Table objects, one per keyspace.  only one instance should ever exist for any given keyspace.
// 这个Map被所有的keyspace共用
private static final Map<String, Table> instances = new NonBlockingHashMap<String, Table>();

// Table name 
public final String name;
    
// ColumnFamilyStore per column family, Integer是cfid
public final Map<Integer, ColumnFamilyStore> columnFamilyStores = new HashMap<Integer, ColumnFamilyStore>();
```

http://cassandra-shawn.googlecode.com/files/cassandra_data_model.JPG

  * IFlushable.java
```
这个接口提供一个Stage工作方式的method，sorter是cpu-bound，而write则是disk-bound

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.locks.Condition;

public interface IFlushable
{
    public void flushAndSignal(CountDownLatch condition, ExecutorService sorter, ExecutorService writer);
}
```

  * Memtable.java
```
Memtable实现了接口IFlushable，所以它会实现flushAndSignal

isFrozen  // Memtable是否正在flush
currentThroughput // Memtable目前的size
currentOperations // Memtable中有多少个Column
creationTime // Memtable的create时间

// Memtable的数据都存储在这里
ConcurrentNavigableMap<DecoratedKey, ColumnFamily> columnFamilies;

// Memtable所属的ColumnFamily对应的cfs，当Memtable满的时候，创建新的Memtable同样绑定在这个cfs上面去
ColumnFamilyStore cfs;

Memtable有两个重要的方法，一个是put，另外一个就是flushAndSignal：

/**
 * Should only be called by ColumnFamilyStore.apply.  NOT a public API.
 * (CFS handles locking to avoid submitting an op
 *  to a flushing memtable.  Any other way is unsafe.)
 */
void put(DecoratedKey key, ColumnFamily columnFamily);
该函数会调用void resolve(DecoratedKey key, ColumnFamily cf)
1) 增加计数器currentThroughput和currentOperations的值。
2) 如果在columnFamilies里不存在key -> columnFamily的映射，则添加进去。
3) 如果存在，则需要将columnFamily里的IColumn添加到已经存在的columnFamily里去。

void flushAndSignal(final CountDownLatch latch, ExecutorService sorter, final ExecutorService writer);
该函数会调用SSTableReader writeSortedContents()
1) 把当前的Memtable添加到cfs的memtablesPendingFlush中
2）将sorter和writer的工作交给ExecutorService
3）write完成以后会将Memtable从cfs的memtablesPendingFlush移除
```

  * BinaryMemtable.java
```
BinaryMemtable实现了接口IFlushable，所以它会实现flushAndSignal

currentSize  // BinaryMemtable当前大小
isFrozen // BinaryMemtable是否可用
Lock lock; // 同步
Condition condition;

// BinaryMemtable存储数据的地方，注意和Memtable的不同的是key -> byte[]的映射
Map<DecoratedKey, byte[]> columnFamilies; 
// BinaryMemtable关联的cfs
ColumnFamilyStore cfs;

BinaryMemtable同样有两个重要的方法，一个是put，另外一个是flushAndSignal

/*
 * This version is used by the external clients to put data into
 * the memtable. This version will respect the threshold and flush
 * the memtable to disk when the size exceeds the threshold.
 */
// 1) 判断isThresholdViolated是否为true，如果是
//   1.1) 上锁
//   1.2) 判断isFrozen，如果是false，则将isFrozen设置为true，调用cfs的submitFlush将
//        BinaryMemtable的数据flush到SSTable，然后调用switchBinaryMemtable将key -> byte[]
//        添加到新建的BinaryMemtable中去。
//   1.3 如果是true，则调用cfs的applyBinary将key -> byte[]添加到新建的BinaryMemtable中去     
// 2）如果不是，直接调用void resolve(DecoratedKey key, byte[] buffer)将key -> byte[]添加到
//    columnFamilies中去
void put(DecoratedKey key, byte[] buffer);


// 和Memtable的flushAndSignal差不多一样，同样调用其SSTableReader 
// writeSortedContents(List<DecoratedKey> sortedKeys)方法，但是这里不会将BinaryMemtable添加// 到cfs的memtablesPendingFlush中去。
void flushAndSignal(final CountDownLatch latch, ExecutorService sorter, final ExecutorService writer);

// 对columnFamilies的keys进行排序返回
List<DecoratedKey> getSortedKeys();
```

  * ColumnFamilyStore.java
```
/*
 * submitFlush first puts [Binary]Memtable.getSortedContents on the flushSorter executor,
 * which then puts the sorted results on the writer executor.  This is because sorting is CPU-bound,
 * and writing is disk-bound; we want to be able to do both at once.  When the write is complete,
 * we turn the writer into an SSTableReader and add it to ssTables_ where it is available for reads.
 *
 * For BinaryMemtable that's about all that happens.  For live Memtables there are two other things
 * that switchMemtable does (which should be the only caller of submitFlush in this case).
 * First, it puts the Memtable into memtablesPendingFlush, where it stays until the flush is complete
 * and it's been added as an SSTableReader to ssTables_.  Second, it adds an entry to commitLogUpdater
 * that waits for the flush to complete, then calls onMemtableFlush.  This allows multiple flushes
 * to happen simultaneously on multicore systems, while still calling onMF in the correct order,
 * which is necessary for replay in case of a restart since CommitLog assumes that when onMF is
 * called, all data up to the given context has been persisted to SSTables.
 */

// Memtable的flushAndSignal会用到这几个fields
ExecutorService flushSorter;
ExecutorService flushWriter;
private Set<Memtable> memtablesPendingFlush = new ConcurrentSkipListSet<Memtable>();

public final Table table;
public final String columnFamily;
public final IPartitioner partitioner;
private final String mbeanName;

private volatile int memtableSwitchCount = 0;

/* This is used to generate the next index for a SSTable */
private AtomicInteger fileIndexGenerator = new AtomicInteger(0);

/* active memtable associated with this ColumnFamilyStore. */
private Memtable memtable;

private final SortedMap<byte[], ColumnFamilyStore> indexedColumns;  // ??

// TODO binarymemtable ops are not threadsafe (do they need to be?)
private AtomicReference<BinaryMemtable> binaryMemtable;

/* SSTables on disk for this column family */
private SSTableTracker ssTables;

private LatencyTracker readStats = new LatencyTracker();
private LatencyTracker writeStats = new LatencyTracker();

public final CFMetaData metadata;

/* These are locally held copies to be changed from the config during runtime */
private int minCompactionThreshold;
private int maxCompactionThreshold;


---------------

√scrub directory过程：

    scrubDataDirectory

    1) Deleted compacted or temporay files
    2) Deleted zero-length files
    3) Deleted orphans files (missing Data file)

√insert/update过程：

    apply(key, value)  ------------> cfs
         |
         |
        put            ------------> memtable
         |
         |
      resolve
         |
         |
    columnFamilies.putIfAbsent(key, cf)

√flush过程：

   forceFlushIfExpired                 -------------> cfs
           |
           |
      forceFlush
           |
           |
   maybeSwitchMemtable
           |
           |
      submitFlush 
           |
           |
     flushAndSignal                     ------------> memtable
           |
           |
   writeSortedContents => getFlushPath
           |  
           |                            =========> addSSTable
           |
SSTableWriter.append(key, value)               

```

  * BinaryVerbHandler.java
  * ClockType.java

  * ColumnFamilyNotDefinedException.java
  * ColumnFamilySerializer.java

  * ColumnFamilyStoreMBean.java
  * ColumnFamilyType.java
  * ColumnIndexer.java
  * ColumnSerializer.java
  * CompactionManager.java
  * CompactionManagerMBean.java
  * DBConstants.java
  * DecoratedKey.java
  * DefinitionsAnnounceVerbHandler.java
  * DefinitionsUpdateResponseVerbHandler.java
  * DefsTable.java

  * HintedHandOffManager.java
  * IClock.java

  * IColumnContainer.java

  * IndexScanCommand.java
  * KeyspaceNotDefinedException.java

  * RangeSliceCommand.java
  * RangeSliceReply.java
  * ReadCommand.java
  * ReadRepairVerbHandler.java
  * ReadResponse.java
  * ReadVerbHandler.java

  * RowIterator.java
  * RowIteratorFactory.java
  * RowMutation.java
  * RowMutationMessage.java
  * RowMutationVerbHandler.java
  * SchemaCheckVerbHandler.java
  * SliceByNamesReadCommand.java
  * SliceFromReadCommand.java

  * SystemTable.java

  * TimestampClock.java
  * TruncateResponse.java
  * TruncateVerbHandler.java
  * Truncation.java
  * UnserializableColumnFamilyException.java
  * WriteResponse.java