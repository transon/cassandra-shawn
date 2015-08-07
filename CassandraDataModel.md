## Introduction ##

Cassandra has a data model that can most easily be thought of as a four or five dimensional hash.

The basic concepts are:
  * Cluster: the machines (nodes) in a logical Cassandra instance. Clusters can contain multiple keyspaces.

  * Keyspace: a namespace for ColumnFamilies, typically one per application.

  * ColumnFamilies: contain multiple columns, each of which has a name, value, and a timestamp, and which are referenced by row keys.

  * SuperColumns can be thought of as columns that themselves have subcolumns.

We'll start from the bottom up, moving from the leaves of Cassandra's data structure (columns) up to the root of the tree (the cluster).

![http://cassandra-shawn.googlecode.com/files/Cassandra%20%E7%9A%84%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B%E5%9B%BE.gif](http://cassandra-shawn.googlecode.com/files/Cassandra%20%E7%9A%84%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B%E5%9B%BE.gif)

## Columns ##

The column is the lowest/smallest increment of data. It's a tuple (triplet) that contains a name, a value and a timestamp.

Here's the thrift interface definition of a Column
```
struct Column {
  1: binary                        name,
  2: binary                        value,
  3: i64                           timestamp,
}
```
And here's a column represented in JSON-ish notation:
```
{
  "name": "emailAddress",
  "value": "foo@bar.com",
  "timestamp": 123456789
}
```
All values are supplied by the client, including the 'timestamp'. This means that clocks on the clients should be synchronized (in the Cassandra server environment is useful also), as these timestamps are used for conflict resolution. In many cases the 'timestamp' is not used in client applications, and it becomes convenient to think of a column as a name/value pair. For the remainder of this document, 'timestamps' will be elided for readability. It is also worth noting the name and value are binary values, although in many applications they are UTF8 serialized strings.

Timestamps can be anything you like, but microseconds since 1970 is a convention. Whatever you use, it must be consistent across the application otherwise earlier changes may overwrite newer ones.

## Column Families ##

A column family is a container for columns, analogous to the table in a relational system. You define column families in your storage-conf.xml file, and cannot modify them (or add new column families) without restarting your Cassandra process. A column family holds an ordered list of columns, which you can reference by the column name.

Column families have a configurable ordering applied to the columns within each row, which affects the behavior of the get\_slice call in the thrift API. Out of the box ordering implementations include ASCII, UTF-8, Long, and UUID (lexical or time).

## Rows ##

In Cassandra, each column family is stored in a separate file, and the file is sorted in row (i.e. key) major order. Related columns, those that you'll access together, should be kept within the same column family.

The row key is what determines what machine data is stored on. Thus, for each key you can have data from multiple column families associated with it. However, these are logically distinct, which is why the Thrift interface is oriented around accessing one ColumnFamily per key at a time. (TODO given this, is the following JSON more confusing than helpful?)

A JSON representation of the key -> column families -> column structure is
```
{
   "mccv":{
      "Users":{
         "emailAddress":{"name":"emailAddress", "value":"foo@bar.com"},
         "webSite":{"name":"webSite", "value":"http://bar.com"}
      },
      "Stats":{
         "visits":{"name":"visits", "value":"243"}
      }
   },
   "user2":{
      "Users":{
         "emailAddress":{"name":"emailAddress", "value":"user2@bar.com"},
         "twitter":{"name":"twitter", "value":"user2"}
      }
   }
}
```
Note that the key "mccv" identifies data in two different column families, "Users" and "Stats". This does not imply that data from these column families is related. The semantics of having data for the same key in two different column families is entirely up to the application. Also note that within the "Users" column family, "mccv" and "user2" have different column names defined. This is perfectly valid in Cassandra. In fact there may be a virtually unlimited set of column names defined, which leads to fairly common use of the column name as a piece of runtime populated data. This is unusual in storage systems, particularly if you're coming from the RDBMS world.

## Keyspaces ##

A keyspace is the first dimension of the Cassandra hash, and is the container for column families. Keyspaces are of roughly the same granularity as a schema or database (i.e. a logical collection of tables) in the RDBMS world. They are the configuration and management point for column families, and is also the structure on which batch inserts are applied.

## Super Columns ##

So far we've covered "normal" columns and rows. Cassandra also supports super columns: columns whose values are super columns; that is, a super column is a (sorted) associative array of columns.

One can thus think of columns and super columns in terms of maps: A row in a regular column family is basically a sorted map of column names to column values; a row in a super column family is a sorted map of super column names to maps of column names to column values.

A JSON description of this layout:
```
{
  "mccv": {
    "Tags": {
      "cassandra": {
        "incubator": {"incubator": "http://incubator.apache.org/cassandra/"},
        "jira": {"jira": "http://issues.apache.org/jira/browse/CASSANDRA"}
      },
      "thrift": {
        "jira": {"jira": "http://issues.apache.org/jira/browse/THRIFT"}
      }
    }  
  }
}
```
Here my column family is "Tags". I have two super columns defined here, "cassandra" and "thrift". Within these I have specific named bookmarks, each of which is a column.

Just like normal columns, super columns are sparse: each row may contain as many or as few as it likes; Cassandra imposes no restrictions.