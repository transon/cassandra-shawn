Cassandra is a hybrid non-relational database in the same class as Google's BigTable. It is more featureful than a key/value store like Riak, but supports fewer query types than a document store like MongoDB.

Cassandra was started by Facebook and later transferred to the open-source community. It is an ideal runtime database for web-scale domains like social networks.

This post is both a tutorial and a "getting started" overview. You will learn about Cassandra's features, data model, API, and operational requirements—everything you need to know to deploy a Cassandra-backed service.

May 11, 2010: post updated for Cassandra gem 0.8 and Cassandra version 0.6.
features

There are a number of reasons to choose Cassandra for your website. Compared to other databases, three big features stand out:

  * Flexible schema: with Cassandra, like a document store, you don't have to decide what fields you need in your records ahead of time. You can add and remove arbitrary fields on the fly. This is an incredible productivity boost, especially in large deployments.
  * True scalability: Cassandra scales horizontally in the purest sense. To add more capacity to a cluster, turn on another machine. You don't have restart any processes, change your application queries, or manually relocate any data.
  * Multi-datacenter awareness: you can adjust your node layout to ensure that if one datacenter burns in a fire, an alternative datacenter will have at least one full copy of every record.

Some other features that help put Cassandra above the competition :

  * Range queries: unlike most key/value stores, you can query for ordered ranges of keys.
  * List datastructures: super columns add a 5th dimension to the hybrid model, turning columns into lists. This is very handy for things like per-user indexes.
  * Distributed writes: you can read and write any data to anywhere in the cluster at any time. There is never any single point of failure.

### installation ###

You need a Unix system. If you are using Mac OS 10.5, all you need is Git. Otherwise, you need to install Java 1.6, Git 1.6, Ruby, and Rubygems in some reasonable way.

Start a terminal and run:

sudo gem install cassandra

If you are using Mac OS, you need to export the following environment variables:

export JAVA\_HOME="/System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home"
export PATH="/System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home/bin:$PATH"

Now you can build and start a test server with cassandra\_helper:

cassandra\_helper cassandra

It runs!
live demo

The above script boots the server with a schema that we can interact with. Open another terminal window and start irb, the Ruby shell:

irb

In the irb prompt, require the library:
```
require 'rubygems'
require 'cassandra'
include SimpleUUID

Now instantiate a client object:

twitter = Cassandra.new('Twitter')

Let's insert a few things:

user = {'screen_name' => 'buttonscat'}
twitter.insert(:Users, '5', user)  

tweet1 = {'text' => 'Nom nom nom nom nom.', 'user_id' => '5'}
twitter.insert(:Statuses, '1', tweet1)

tweet2 = {'text' => '@evan Zzzz....', 'user_id' => '5', 'reply_to_id' => '8'}
twitter.insert(:Statuses, '2', tweet2)
```
Notice that the two status records do not have all the same columns. Let's go ahead and connect them to our user record:
```
twitter.insert(:UserRelationships, '5', {'user_timeline' => {UUID.new => '1'}})
twitter.insert(:UserRelationships, '5', {'user_timeline' => {UUID.new => '2'}})
```
The UUID.new call creates a collation key based on the current time; our tweet ids are stored in the values.

Now we can query our user's tweets:
```
timeline = twitter.get(:UserRelationships, '5', 'user_timeline', :reversed => true)
timeline.map { |time, id| twitter.get(:Statuses, id, 'text') }
# => ["@evan Zzzz....", "Nom nom nom nom nom."]
```
Two tweet bodies, returned in recency order—not bad at all. In a similar fashion, each time a user tweets, we could loop through their followers and insert the status key into their follower's home\_timeline relationship, for handling general status delivery.

### the data model ###

Cassandra is best thought of as a 4 or 5 dimensional hash. The usual way to refer to a piece of data is as follows: a keyspace, a column family, a key, an optional super column, and a column. At the end of that chain lies a single, lonely value.

Let's break down what these layers mean.

  * Keyspace (also confusingly called "table"): the outer-most level of organization. This is usually the name of the application. For example, 'Twitter' and 'Wordpress' are both good keyspaces. Keyspaces must be defined at startup in the storage-conf.xml file.
  * Column family: a slice of data corresponding to a particular key. Each column family is stored in a separate file on disk, so it can be useful to put frequently accessed data in one column family, and rarely accessed data in another. Some good column family names might be :Posts, :Users and :UserAudits. Column families must be defined at startup.
  * Key: the permanent name of the record. You can query over ranges of keys in a column family, like :start => '10050', :finish => '10070'—this is the only index Cassandra provides for free. Keys are defined on the fly.

After the column family level, the organization can diverge—this is a feature unique to Cassandra. You can choose either:

  * A column: this is a tuple with a name and a value. Good columns might be 'screen\_name' => 'lisa4718' or 'Google' => 'http://google.com'.

> It is common to not specify a particular column name when requesting a key; the response will then be an ordered hash of all columns. For example, querying for (:Users, '174927') might return:
```
      {'name' => 'Lisa Jones', 
       'gender' => 'f', 
       'screen_name' => 'lisa4718'}
```
> In this case, name, gender, and screen\_name are all column names. Columns are defined on the fly, and different records can have different sets of column names, even in the same keyspace and column family. This lets you use the column name itself as either structure or data. Columns can be stored in recency order, or alphabetical by name, and all columns keep a timestamp.
  * A super column: this is a named list. It contains standard columns, stored in recency order.

> Say Lisa Jones has bookmarks in several categories. Querying (:UserBookmarks, '174927') might return:
```
      {'work' => {
          'Google' => 'http://google.com', 
          'IBM' => 'http://ibm.com'}, 
       'todo': {...}, 
       'cooking': {...}}
```
> Here, work, todo, and cooking are all super column names. They are defined on the fly, and there can be any number of them per row. :UserBookmarks is the name of the super column family. Super columns are stored in alphabetical order, with their sub columns physically adjacent on the disk.

Super columns and standard columns cannot be mixed at the same (4th) level of dimensionality. You must define at startup which column families contain standard columns, and which contain super columns with standard columns inside them.

Super columns are a great way to store one-to-many indexes to other records: make the sub column names TimeUUIDs (or whatever you'd like to use to sort the index), and have the values be the foreign key. We saw an example of this strategy in the demo, above.

If this is confusing, don't worry. We'll now look at two example schemas in depth.
twitter schema

Here is the schema definition we used for the demo, above. It is based on Eric Florenzano's Twissandra:
```
<Keyspace Name="Twitter">
  <ColumnFamily CompareWith="UTF8Type" Name="Statuses" />
  <ColumnFamily CompareWith="UTF8Type" Name="StatusAudits" />
  <ColumnFamily CompareWith="UTF8Type" Name="StatusRelationships"
    CompareSubcolumnsWith="TimeUUIDType" ColumnType="Super" />  
  <ColumnFamily CompareWith="UTF8Type" Name="Users" />
  <ColumnFamily CompareWith="UTF8Type" Name="UserRelationships"
    CompareSubcolumnsWith="TimeUUIDType" ColumnType="Super" />
</Keyspace>
```
What could be in StatusRelationships? Maybe a list of users who favorited the tweet? Having a super column family for both record types lets us index each direction of whatever many-to-many relationships we come up with.

Here's how the data is organized:

![http://cassandra-shawn.googlecode.com/files/twitter.jpg](http://cassandra-shawn.googlecode.com/files/twitter.jpg)
Cassandra lets you distribute the keys across the cluster either randomly, or in order, via the Partitioner option in the storage-conf.xml file.

For the Twitter application, if we were using the order-preserving partitioner, all recent statuses would be stored on the same node. This would cause hotspots. Instead, we should use the random partitioner.

Alternatively, we could preface the status keys with the user key, which has less temporal locality. If we used user\_id:status\_id as the status key, we could do range queries on the user fragment to get tweets-by-user, avoiding the need for a user\_timeline super column.
multi-blog schema

Here's a another schema, suggested to me by Jonathan Ellis, the primary Cassandra maintainer. It's for a multi-tenancy blog platform:
```
<Keyspace Name="Multiblog">      
  <ColumnFamily CompareWith="TimeUUIDType" Name="Blogs" />
  <ColumnFamily CompareWith="TimeUUIDType" Name="Comments"/>
</Keyspace>
```
Imagine we have a blog named 'The Cutest Kittens'. We will insert a row when the first post is made as follows:
```
require 'rubygems'
require 'cassandra'
include SimpleUUID

multiblog = Cassandra.new('Multiblog')

multiblog.insert(:Blogs, 'The Cutest Kittens',
  { UUID.new => 
    '{"title":"Say Hello to Buttons Cat","body":"Buttons is a cute cat."}' })
```
UUID.new generates a unique, sortable column name, and the JSON hash contains the post details. Let's insert another:
```
multiblog.insert(:Blogs, 'The Cutest Kittens',
  { UUID.new => 
    '{"title":"Introducing Commie Cat","body":"Commie is also a cute cat"}' })
```
Now we can find the latest post with the following query:
```
post = multiblog.get(:Blogs, 'The Cutest Kittens', :reversed => true).to_a.first
```
On our website, we can build links based on the readable representation of the UUID:
```
guid = post.first.to_guid
# => "b06e80b0-8c61-11de-8287-c1fa647fd821"
```
If the user clicks this string in a permalink, our app can find the post directly via:
```
multiblog.get(:Blogs, 'The Cutest Kittens', :start => UUID.new(guid), :count => 1)
```
For comments, we'll use the post UUID as the outermost key:
```
multiblog.insert(:Comments, guid,
  {UUID.new => 'I like this cat. - Evan'})
multiblog.insert(:Comments, guid, 
  {UUID.new => 'I am cuter. - Buttons'})
```
Now we can get all comments (oldest first) for a post by calling:
```
multiblog.get(:Comments, guid)
```
We could paginate them by passing :start with a UUID. See this presentation to learn more about token-based pagination.

We have sidestepped two problems with this data model: we don't have to maintain separate indexes for any lookups, and the posts and comments are stored in separate files, where they don't cause as much write contention. Note that we didn't need to use any super columns, either.

### storage layout and api comparison ###

The storage strategy for Cassandra's standard model is the same as BigTable's. Here's a comparison chart:

http://cassandra-shawn.googlecode.com/files/diff.JPG

Column families are stored in column-major order, which is why people call BigTable a column-oriented database. This is not the same as a column-oriented OLAP database like Sybase IQ—it depends on whether your data model considers keys to span column families or not.

![http://cassandra-shawn.googlecode.com/files/row_oriented.jpg](http://cassandra-shawn.googlecode.com/files/row_oriented.jpg)

In row-orientation, the column names are the structure, and you think of the column families as containing keys. This is the convention in relational databases.

![http://cassandra-shawn.googlecode.com/files/column_oriented.jpg](http://cassandra-shawn.googlecode.com/files/column_oriented.jpg)

In column-orientation, the column names are the data, and the column families are the structure. You think of the key as containing the column family, which is the convention in BigTable. (In Cassandra, super columns are also stored in column-major order—all the sub columns are together.)

In Cassandra's Ruby API, parameters are expressed in storage order, for clarity:

http://cassandra-shawn.googlecode.com/files/diff2.JPG

Note that Cassandra's internal Thrift interface mimics BigTable in some ways, but this is being changed.
going to production

Cassandra is an alpha product and could, theoretically, lose your data. In particular, if you change the schema specified in the storage-conf.xml file, you must follow these instructions carefully, or corruption will occur (this is going to be fixed). Also, the on-disk storage format is subject to change, making upgrading a bit difficult.

The biggest deployment is at Facebook, where hundreds of terabytes of token indexes are kept in about a hundred Cassandra nodes. However, their use case allows the data to be rebuilt if something goes wrong. Proceed carefully, keep a backup in an unrelated storage engine...and submit patches if things go wrong. (Some other production deployments are listed here.)

That aside, here is a guide for deploying a production cluster:

  * Hardware: get a handful of commodity Linux servers. 16GB memory is good; Cassandra likes a big filesystem buffer. You don't need RAID. If you put the commitlog file and the data files on separate physical disks, things will go faster. Don't use EC2 or friends without being aware that the virtualized I/O can be slow, especially on the small instances.
  * Configuration: in the storage-conf.xml schema file, set the replication factor to 3. List the IP address of one of the nodes as the seed. Set the listen address to the empty string, so the hosts will resolve their own IPs. Now, adjust the contents of cassandra.in.sh for your various paths and JVM options—for a 16GB node, set the JVM heap to 4GB.
  * Deployment: build a package of Cassandra itself and your configuration files, and deliver it to all your servers (I use Capistrano for this). Start the servers by setting CASSANDRA\_INCLUDE in the environment to point to your cassandra.in.sh file, and run bin/cassandra. At this point, you should see join notices in the Cassandra logs:

> Cassandra starting up...
> Node 10.224.17.13:7001 has now joined.
> Node 10.224.17.14:7001 has now joined.

> Congratulations! You have a cluster. Don't forget to turn off debug logging in the log4j.properties file.
  * Visibility: you can get a little more information about your cluster via the tool bin/nodeprobe, included:

> $ bin/nodeprobe --host 10.224.17.13 ring
> Token(124007023942663924846758258675932114665)  3 10.224.17.13  |<--|
> Token(106858063638814585506848525974047690568)  3 10.224.17.19  |   ^
> Token(141130545721235451315477340120224986045)  3 10.224.17.14  |-->|

> Cassandra also exposes various statistics over JMX.

Note that your client machines (not servers!) must have accurate clocks for Cassandra to resolve write conflicts properly. Use NTP.
conclusion

There is a misperception that if someone advocates a non-relational database, they either don't understand SQL optimization, or they are generally a hater. This is not the case.

It is reasonable to seek a new tool for a new problem, and database problems have changed with the rise of web-scale distributed systems. This does not mean that SQL as a general-purpose runtime and reporting tool is going away. However, at web-scale, it is more flexible to separate the concerns. Runtime object lookups can be handled by a low-latency, strict, self-managed system like Cassandra. Asynchronous analytics and reporting can be handled by a high-latency, flexible, un-managed system like Hadoop. And in neither case does SQL lend itself to sharding.

I think that Cassandra is the most promising current implementation of a runtime distributed database, but much work remains to be done. We're beginning to use Cassandra at Twitter, and here's what I would like to happen real-soon-now:

  * Interface cleanup: the Thrift API for Cassandra is incomplete and inconsistent, which makes writing clients very irritating.
> > Done!
  * Online migrations: restarting the cluster 3 times to add a column family is silly.
  * ActiveModel or DataMapper adapter: for interaction with business objects in Ruby.
> > Done! Michael Koziarski on the Rails core team wrote an ActiveModel adapter.
  * Scala client: for interoperability with JVM middleware.

Go ahead and jump on any of those projects—it's a chance to get in on the ground floor.

Cassandra has excellent performance. There some benchmark results for version 0.5 at the end of the Yahoo performance study.
further resources

  * Cassandra wiki
  * Presentation by Avinash Lakshman about Cassandra: slides, video
  * The cassandra-user and cassandra-dev mailing lists
  * The #cassandra IRC channel on irc.freenode.net
  * Cassandra's bug tracker
  * Twitter's Ruby client: docs, source

July 02, 2009