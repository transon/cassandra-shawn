Cassandra has received a lot of attention of late, and more people are now evaluating it for their organization. As these folks work to get up to speed, the shortcomings in our documentation become all the more apparent. Easily, the worst of these is explaining the data model to those with an existing background in relational databases.

The problem is that Cassandra’s data model is different enough from that of a traditional database to readily cause confusion, and just as numerous as the misconceptions are the different ways that well intentioned people use to correct them.

Some folks will describe the model as a map of maps, or in the case of super columns, a map of maps of maps. Often, these explanations are accompanied by visual aids that use a JSON-like notation to demonstrate. Others will liken column families to sparse tables, and others still as containers that hold collections of column objects. Columns are even sometimes referred to as 3-tuples. All of these fall short in my opinion.

The problem is that it’s difficult to explain something new without using analogies, but confusing when the comparisons don’t hold up. I’m still hoping that someone will devise an elegant means of explaining this, but in the meantime I find concrete examples to be worth their weight in gold.
## Twitter ##

Despite being an actual use-case for Cassandra, Twitter is also an excellent vehicle for discussion since it is well known and easily conceptualized. We know for example that, like most sites, user information (screen name, password, email address, etc), is kept for everyone and that those entries are linked to one another to map friends and followers. And, it wouldn’t be Twitter if it weren’t storing tweets, which in addition to the 140 characters of text are also associated with meta-data like timestamp and the unique id that we see in the URLs.

Were we modelling this in a relational database the approach would be pretty straight-forward, we’d need a table to store our users.
```
CREATE TABLE user (
    id INTEGER PRIMARY KEY,
    username VARCHAR(64),
    password VARCHAR(64)
);
```
We’d need tables we could use to perform the one-to-many joins to return followers and followees.
```
CREATE TABLE followers (
    user INTEGER REFERENCES user(id),
    follower INTEGER REFERENCES user(id)
);
 
CREATE TABLE following (
    user INTEGER REFERENCES user(id),
    followed INTEGER REFERENCES user(id)
);
```
And of course we’d need a table to store the tweets themselves.
```
CREATE TABLE tweets (
    id INTEGER,
    user INTEGER REFERENCES user(id),
    body VARCHAR(140),
    timestamp TIMESTAMP
);
```
I’ve greatly oversimplified things here for the purpose of demonstration, but even with a trivial model like this, there is much to be taken for granted. For example, to accomplish data normalization like this in a practical way we need foreign-key constraints, and since we need to perform joins to combine data from multiple tables, we’ll need to be able to arbitrarily create indices on the appropriate attributes to make that efficient.

But getting distributed systems right is a real challenge, and it never comes without trade-offs. This is true of Cassandra and is why the data model above won’t work for us. For starters, there is no referential integrity, and the lack of support for secondary indexing makes it difficult to efficiently perform joins, so you must denormalize. Put another way, you’re forced to think in terms of the queries you’ll perform and the results you expect since this is likely what the model will look like.
## Twissandra ##

So how would the model above be translated to Cassandra? Fortunately we need only look as far as Twissandra, a functional albeit minimalist Twitter clone written by Eric Florenzano, specifically to serve as a sample. So lets explore data modelling in Cassandra using Twitter and Twissandra as an example.
### Schema ###

Cassandra is considered a schema-less data-store, but it is necessary to perform some configuration specific to your application. Twissandra comes with a sample configuration for Cassandra that should Just Work, but it’s worth taking some time to look at the specific aspects related to the data model.
### Keyspaces ###

Keyspaces are the upper-most namespace in Cassandra and typically you’ll see exactly one for each application. In future versions of Cassandra, keyspaces will be created dynamically similar to how you create databases on an RDBMS server, but for 0.6 and before, these are defined in the main configuration file like so:
```
 <Keyspaces>
  <Keyspace Name="Twissandra">
  ...
  </Keyspace>
</Keyspaces>
```
### Column Families ###

For each keyspace there are one or more column families. A column family is the namespace used to associate records of a similar kind. Cassandra gives you record-level atomicity within a column family when making writes, and queries against them are efficient. These qualities are important to keep in mind when designing your data model, as you’ll see in the discussion that follows.

Like keyspaces, the column families themselves are defined in the main config, though in future versions you’ll create them on the fly similar to the way you create tables in an RDBMS.
```
<Keyspaces>
  <Keyspace Name="Twissandra">
    <ColumnFamily CompareWith="UTF8Type"  Name="User"/>
    <ColumnFamily CompareWith="BytesType" Name="Username"/>
    <ColumnFamily CompareWith="BytesType" Name="Friends"/>
    <ColumnFamily CompareWith="BytesType" Name="Followers"/>
    <ColumnFamily CompareWith="UTF8Type"  Name="Tweet"/>
    <ColumnFamily CompareWith="LongType"  Name="Userline"/>
    <ColumnFamily CompareWith="LongType"  Name="Timeline"/>
  </Keyspace>
</Keyspaces>
```
One thing worth pointing out from the config snippet above is that in addition to a name, column family definitions also specify a comparator. This highlights another important distinction from traditional databases in that the order records are sorted is a design decision, and not something that can easily be changed later.
### What Are These Column Families? ###

It’s probably not immediately intuitive what all seven Twissandra column families are for, so let’s take a closer look at each.
  * User
This is where users are stored, it is analogous to the user table in the SQL schema above. Each record stored in this column family will be keyed on a UUID and contain columns for username and password.

  * Username
Looking up a user in the User column family above requires knowing that user’s key, but how do we find this UUID-based key when all we know is the username? With a relational database and the SQL schema above, we’d perform a SELECT on the users table with a predicate to match the username (WHERE username = ‘jericevans’). This won’t work with Cassandra for a couple of reasons.

First off, a relational database will scan your table sequentially when performing a SELECT like this, and since records are distributed throughout a Cassandra cluster based on key, the equivalent could mean contacting more than one node (possibly many). However, even with all of the data on a single machine, there comes a point when such an operation will become inefficient with a relational database, making it necessary to index the username attribute. As mentioned earlier, Cassandra doesn’t currently support secondary indices like this.

The answer is to create our own inverted index that maps readable usernames to the UUID-based key, and that is the purpose of this column family.

  * Friends
  * Followers
The Friends and Followers column families will answer the questions, who is user X following?(UserX是哪些User的文蜜), and who is following user X(UserX的文蜜有哪些，UserX不能老是当别人的蜜蜂啊，所以它也有自己的蜜蜂)?, respectively. Each is keyed on the unique user ID, with columns to track the corresponding relationships and the time they were created.

  * Tweet
This is where the tweets themselves are stored. This column family stores records with unique keys (UUIDs), and columns for the user id, the body, and the time the tweet was added.

  * Userline
This is where the timeline as it pertains to each user is stored. Records here consist of user ID keys, and columns to map a numeric timestamp to the unique tweet id in the Tweet column family.（自己发的tweet）

  * Timeline
Finally, this column family is similar to Userline, except that it stores the materialized view of friend tweets for each user.（我感兴趣的user发的tweet）

So, given the above column families, let’s step through some common operations and see how they would be applied.
## Tying It All Together ##
### Adding a new user ###

First off, new users will need a way to sign up for an account, and when they do we’ll need to add them to our Cassandra database. For Twissandra, that would look something like:
```
username = 'jericevans'
password = '**********'
useruuid = str(uuid())
 
columns = {'id': useruuid, 'username': username, 'password': password}
 
USER.insert(useruuid, columns)
USERNAME.insert(username, {'id': useruuid})
```
Twissandra is written in Python and uses Pycassa for client access, so the uppercase USER and USERNAME above are pycassa.ColumnFamily instances that would have been created elsewhere during initialization for “User” and “Username” respectively.

Also, this is a good time to mention that this and the code samples that follow aren’t verbatim snippets from Twissandra, I’ve changed them to be more concise and self-contained. For example, in the code above, it wouldn’t make sense to assign variables for username and password, in a web application these would be taken from the form elements on a sign-up page.

Getting back to the sample, there are two different Cassandra write (insert()) operations taking place here, one to create a new record in the User column family and one to update the inverted index that maps human readable usernames to UUID keys. In both cases, the arguments to insert() are the key that we’ll later use to look up the records, and a map containing the column names and values.
### Following a friend ###
```
frienduuid = 'a4a70900-24e1-11df-8924-001ff3591711'
 
FRIENDS.insert(useruuid, {frienduuid: time.time()})
FOLLOWERS.insert(frienduuid, {useruuid: time.time()})
```
Again we perform two different insert() operations, this time to add someone to our list of friends, and to track the inverse of that relationship, the addition of a new follower on the target user.
### Tweeting ###
```
tweetuuid = str(uuid())
body = '@ericflo thanks for Twissandra, it helps!'
timestamp = long(time.time() * 1e6)
 
columns = {'id': tweetuuid, 'user_id': useruuid, 'body': body, '_ts': timestamp}
TWEET.insert(tweetuuid, columns)
 
columns = {struct.pack('&gt;d'), timestamp: tweetuuid}
USERLINE.insert(useruuid, columns)
 
TIMELINE.insert(useruuid, columns)
for otheruuid in FOLLOWERS.get(useruuid, 5000):
    TIMELINE.insert(otheruuid, columns)
```
To store a new tweet, we create a new record in the Tweet column family using a newly created UUID as the key, with columns for the author’s user ID, the time it was created, and of course the text of the tweet itself.

Additionally, the user’s Userline is updated to map the time of the tweet to its unique ID. If this is the user’s first tweet the insert() will result in a new record, subsequent inserts will create new columns in this record.

Finally, Timeline is updated with columns that map time to tweet ID for this user and each of their followers.

One thing worth paying particular attention to here is that the timestamp used is a long (64 bit), and when it is given as a column name, it’s packed as a binary value in network byte-order. This is because the Userline and Timeline column families use a LongType comparator, allowing us to query for ranges of columns using numeric predicates, with results that are sorted numerically.
### Getting a user’s tweets ###
```
userline = USERLINE.get(useruuid, column_reversed=True)
tweets = TWEET.multiget(userline.values())
```
Here we’re retrieving the tweets for a user, first by obtaining a list of the IDs from Userline, and then fetching them from the Tweet column family with a multiget(). These results will be sorted by the numeric date/time, and in descending order since Userline uses a LongType comparator and reversed was set to True.
### Retrieving the timeline for a user ###
```
start = request.GET.get('start')
limit = NUM_PER_PAGE
 
timeline = TIMELINE.get(useruuid, column_start=start,
column_count=limit, column_reversed=True)
tweets = TWEET.multiget(timeline.values())
```
Just like the previous example, here we’re retrieving a list of tweet IDs, this time from Timeline. However, this time we’re also using start and limit to control the range of columns returned. This is handy for paging through the results.
## So What Next? ##

Hopefully this is enough to give you the general idea. Again, I took some liberties with the code samples and omitted some operations in an effort to be concise, so now might be a good time to check out the Twissandra source and take a deeper dive. There are a number of obvious features like retweet and lists, that were intentionally left unimplemented to serve as exercises for the initiated. If you’re comfortable with Python and Django, you might try your hand at one of those.

The wiki contains a growing base of information, including an up-to-date list of articles and presentations that people have given.

If IRC is your thing you can join #cassandra on irc.freenode.net where you can chat with people who have Been There and Done That and are always willing to help with questions. Or, if email is more your style there are also plenty of helpful folks on the cassandra-user list.
# Related Posts: Setting up memcached on Cloud Servers
# Rackspace’s Take on the Open Cloud Manifesto
# memcached: More Cache = Less Cash!
# Alternative Database Technology for the Cloud: There is No Silver Bullet
# How do you put a Database in the Cloud?

Ref: http://www.rackspacecloud.com/blog/2010/05/12/cassandra-by-example/