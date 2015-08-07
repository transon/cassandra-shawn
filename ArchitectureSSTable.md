DRAFT. Notes on documenting how SSTables work in Cassandra (data format, indexing, serialization, searching)

SSTables have 3 separate files created, and are per column-family.
```
   1. Bloom Filter
   2. Index
   3. Data 
```
When adding a new key to an SSTable here are the steps it goes through. All keys are sorted before writing.
http://cassandra-shawn.googlecode.com/files/Sorted%20String%20Table1.JPG
http://cassandra-shawn.googlecode.com/files/Sorted%20String%20Table2.JPG


---

In "org.apache.cassandra.io.sstable":

  * BloomFilterTracker.java

  * Descriptor.java
```
/**
 * A SSTable is described by the keyspace and column family it contains data
 * for, a generation (where higher generations contain more recent data) and
 * an alphabetic version string.
 *
 * A descriptor can be marked as temporary, which influences generated filenames.
 */
public static final String LEGACY_VERSION = "a";
public static final String CURRENT_VERSION = "e";

public final File directory;
public final String version;
public final String ksname;
public final String cfname;
public final int generation;
public final boolean temporary;
private final int hashCode;

public final boolean hasStringsInBloomFilter;
public final boolean hasIntRowSize;
public final boolean hasEncodedKeys;
public final boolean isLatestVersion;

方法：
/**
 * @param suffix A component suffix, such as 'Data.db'/'Index.db'/etc
 * @return A filename for this descriptor with the given suffix.
 *
 * eg: directory/column family name-tmp-version-generation-suffix
                 ==================             ========== ======
 *     suffix such as "Data.db", "Index.db" ... 
 */
public String filenameFor(String suffix)

/**
 * Filename of the form "<ksname>/<cfname>-[tmp-][<version>-]<gen>-<component>"
 *                       ======== =============================================
 *                      directory                  name
 * @return A Descriptor for the SSTable, and the Component remainder.
 */
static Pair<Descriptor,String> fromFilename(File directory, String name)

/**
 * @return True if the given version string is not empty, and
 * contains all lowercase letters, as defined by java.lang.Character.
 */
static boolean versionValidate(String ver)
```

  * Component.java
```
/**
 * SSTables are made up of multiple components in separate files. Components are
 * identified by a type and an id, but required unique components (such as the Data
 * and Index files) may have implicit ids assigned to them.
 */

type: 
// the base data for an sstable: the remaining components can be regenerated
// based on the data component
DATA("Data.db"),
// index of the row keys with pointers to their positions in the data file
PRIMARY_INDEX("Index.db"),
// serialized bloom filter for the row keys in the sstable
FILTER("Filter.db"),
// 0-length file that is created when an sstable is ready to be deleted
COMPACTED_MARKER("Compacted"),
// statistical metadata about the content of the sstable
STATS("Statistics.db"),
// a bitmap secondary index: many of these may exist per sstable
BITMAP_INDEX("Bitidx.db");

/**
 * Filename of the form "<ksname>/<cfname>-[tmp-][<version>-]<gen>-<component>",
 * where <component> is of the form "[<id>-]<component>".
 * @return A Descriptor for the SSTable, and a Component for this particular file.
 * TODO move descriptor into Component field
 */
这个方法会调用Descriptor.java的fromFilename，取其left作为Descriptor，取其right构造Component
public static Pair<Descriptor,Component> fromFilename(File directory, String name)
```

  * IndexHelper.java
```
/**
 * Skip the bloom filter
 * @param in the data input from which the bloom filter should be skipped
 * @throws IOException
 */
public static void skipBloomFilter(DataInput in) throws IOException

/**
 * Skip the index
 * @param file the data input from which the index should be skipped
 * @throws IOException
 */
public static void skipIndex(DataInput file) throws IOException


IndexInfo:

    public final long width;
    public final byte[] lastName;
    public final byte[] firstName;
    public final long offset;

    public void serialize(DataOutput dos) throws IOException
    {
        FBUtilities.writeShortByteArray(firstName, dos);
        FBUtilities.writeShortByteArray(lastName, dos);
        dos.writeLong(offset);
        dos.writeLong(width);
    }

/**
 * Deserialize the index into a structure and return it
 * @throws IOException
 */
读取Index.DB文件构造ArrayList<IndexInfo>
public static ArrayList<IndexInfo> deserializeIndex(FileDataInput in) throws IOException

/**
 * Defreeze the bloom filter.
 *
 * @return bloom filter summarizing the column information
 * @throws java.io.IOException
 */
读取Filter.DB文件构造BloomFilter
public static BloomFilter defreezeBloomFilter(DataInput file) throws IOException
  
/**
 * the index of the IndexInfo in which @name will be found.
 * If the index is @indexList.size(), the @name appears nowhere.
 */
根据name名字在List<IndexInfo>二分查找name对应的IndexInfo
public static int indexFor(byte[] name, List<IndexInfo> indexList, AbstractType comparator, boolean reversed)
```

  * IndexSummary.java
```

/**
 * This is a simple container for the index Key and its corresponding position
 * in the index file. Binary search is performed on a list of these objects
 * to find where to start looking for the index entry containing the data position
 * (which will be turned into a PositionSize object)
 */
public static class KeyPosition implements Comparable<KeyPosition>
{
    public final DecoratedKey key;
    public final long indexPosition;

    public KeyPosition(DecoratedKey key, long indexPosition)
    {
        this.key = key;
        this.indexPosition = indexPosition;
     }

     ...................
}

private ArrayList<KeyPosition> indexPositions;
private int keysWritten = 0;
private long lastIndexPosition;

// 将key和对应的indexPosition添加到ArrayList<KeyPosition>中, 注意它不会把所有sstable中
// 的key都添加IndexSummary中，而是把sstable中间隔为128的key放入，目前定义的间隔大小为
// 128(public Integer index_interval = 128;)
// 例如：把第一个key放入IndexSummary、把第128个key放入IndexSummary、把第256个key放入
// IndexSummary .....
public void maybeAddEntry(DecoratedKey decoratedKey, long indexPosition)

     ArrayList<KeyPosition>                    index.DB                    data.DB

key -------------------------> indexPosition ---------------> data offset ----------> value
```

  * KeyIterator.java
  * ReducingKeyIterator.java

  * SSTable.java
```
/**
 * This class is built on top of the SequenceFile. It stores
 * data on disk in sorted fashion. However the sorting is upto
 * the application. This class expects keys to be handed to it
 * in sorted order.
 *
 * A separate index file is maintained as well, containing the
 * SSTable keys and the offset into the SSTable at which they are found.
 * Every 1/indexInterval key is read into memory when the SSTable is opened.
 *
 * Finally, a bloom filter file is also kept for the keys in each SSTable.
 */

```

  * SSTableWriter.java
```

IndexWriter:

    BufferedRandomAccessFile indexFile;
    Descriptor desc;
    IndexSummary summary;
    BloomFilter bf;

    1) add the key into bloom filter
    2) write the key into index file
    3) write the data position into index file
    4) add the (key, index position) into index summary
    5) add indexPosition into builder
    afterAppend(DecoratedKey key, long dataPosition)

// used to manage index file, bloom filter, index summary
private IndexWriter iwriter;                                 

// the data will be written into data file
private final BufferedRandomAccessFile dataFile;

// remember the last written key             
private DecoratedKey lastWrittenKey;                         

// check the decoratedKey with the lastWrittenKey, keys must be written in ascending order
beforeAppend(DecoratedKey decoratedKey)

// 1) assign decoratedKey to lastWrittenKey
// 2) add dataPosition into data builder
// 3) call IndexWriter's afterAppend
afterAppend(DecoratedKey decoratedKey, long dataPosition)

// 1) call beforeAppend
// 2) write data into data file (data length, data)
// 3) call afterAppend
append(AbstractCompactedRow row)
append(DecoratedKey decoratedKey, ColumnFamily cf)
append(DecoratedKey decoratedKey, byte[] value)
```

  * SSTableReader.java
```
/**
 * SSTableReaders are open()ed by Table.onStart; after that they are created by SSTableWriter.renameAndOpen.
 * Do not re-call open() on existing SSTable files; use the references kept by ColumnFamilyStore post-start instead.
 */

// 关键方法
/**
 * load the content of filter.DB into memory
 */
void loadBloomFilter();

/**
 * Loads ifile, dfile and indexSummary, and optionally recreates the bloom filter.
 */
1) create empty IndexSummary
2) estimate key count based on index length for bloom filter
3) read index file, put the key into bloom filter if recreatebloom is true
4) add the key and index position into IndexSummary
private void load(boolean recreatebloom);

/** 
 * get the position in the index file to start scanning to find the 
 * given key (at most indexInterval keys away) 
 */
private IndexSummary.KeyPosition getIndexScanPosition(DecoratedKey decoratedKey);

/**
 * @param decoratedKey The key to apply as the rhs to the given Operator.
 * @param op The Operator defining matching keys: the nearest key to the target matching the operator wins.
 * @return The position in the data file to find the key, or -1 if the key is not present
 */

// 1) check the bloom filter, test if it is exist
// 2) check the key cache
// 3) see if the sampled index says it's impossible for the key to be present using IndexSummary
// 4) scan the on-disk index, starting at the nearest sampled position
//    4.1) read key & data position from index entry
//    4.2) check if it is the expected key
//         4.2.1) yes, add the (key, data position) into cache 
public long getPosition(DecoratedKey decoratedKey, Operator op);

/**
 * Determine the minimal set of sections that can be extracted from this SSTable to cover the given ranges.
 * @return A sorted list of (offset,end) pairs that cover the given ranges in the datafile for this SSTable.
 * 
 * range => Part<Long, Long> (offset, end)
 */
public List<Pair<Long,Long>> getPositionsForRanges(Collection<Range> ranges)

```

  * SegmentedFile.java
```
/**
 * Abstracts a read-only file that has been split into segments, each of which can be represented by an independent
 * FileDataInput. Allows for iteration over the FileDataInputs, or random access to the FileDataInput for a given
 * position.
 *
 * The JVM can only map up to 2GB at a time, so each segment is at most that size when using mmap i/o. If a segment
 * would need to be longer than 2GB, that segment will not be mmap'd, and a new RandomAccessFile will be created for
 * each access to that segment.
 */

public final String path;
public final long length;

public abstract FileDataInput getSegment(long position, int bufferSize);

将一个文件变成iteration，有许多个Segment组成，而每个Segment又一个起始位置和长度来标示
这样就将一个长度为length的文件切割为了许多个长度为bufferSize的Segment，每个Segment附加
一个起始位置。
                    
                                 position
                                     \   
                                      \
                                       \ bufferSize /
------------------------------------------------------------------------------
| Segment#0 | Segment#1 | ............. | Segment#i | ........... | Segment#n|
------------------------------------------------------------------------------
                                      (FileDataInput#i)

/**
 * A lazy Iterator over segments in forward order from the given position.
 */
final class SegmentIterator implements Iterator<FileDataInput>
{
    private long nextpos;
    private final int bufferSize;
    public SegmentIterator(long position, int bufferSize)
    {
        this.nextpos = position;
        this.bufferSize = bufferSize;
    }

    public boolean hasNext()
    {
        return nextpos < length;
    }

    public FileDataInput next()
    {
        long position = nextpos;
        if (position >= length)
            throw new NoSuchElementException();

        FileDataInput segment = getSegment(nextpos, bufferSize);
        try
        {
             nextpos = nextpos + segment.bytesRemaining();
        }
        catch (IOException e)
        {
             throw new IOError(e);
        }
        return segment;
    }

    public void remove() { throw new UnsupportedOperationException(); }
}
                                         
/**
 * @return An Iterator over segments, beginning with the segment containing the given position: each segment must be closed after use.
 */
public Iterator<FileDataInput> iterator(long position, int bufferSize)
{
    return new SegmentIterator(position, bufferSize);
}

SegmentIterator实现了迭代器Iterator<FileDataInput>，而SegmentIterator由许多个Segment组成,而每个Segment由两个field组成，一个是nextPos,另外一个是bufferSize 

```

  * SSTableDeletingReference.java
  * SSTableIdentityIterator.java
  * SSTableScanner.java
  * SSTableTracker.java