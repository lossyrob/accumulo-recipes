#What is the Event Store Recipe?

We tend to define events today in enterprise systems as an immutable object that occurs at some well-defined time. Events in the Calrissian stack are composed of tuples- that is, a set of keys and values that also contain visiblity information for cell-level security. The event store is a document store. But it's not just any document store, it's a document store that can scale to extreme sizes. It supplies a rich query interface for selecting events from the store. Future versions will include MapReduce input formats that can query events directly.

##What does an event look like?

Many of the stores in the Calrissian stack deal with objects modeled like events. A "StoreEntry" object is provided for such a purpose. Each store entry object must have an id, a timestamp, and at least one tuple to define it.

```java
// create our event to contain the keys/values we plan to add to give it state
StoreEntry event = new StoreEntry(UUID.randomUUID().toString(), System.currentTimeMillis());

// add the state to the event
event.put(new Tuple("systemName", "system1", "USER&ADMIN"));
event.put(new Tuple("eventType", "status", "USER&ADMIN"));
event.put(new Tuple("healthOK", true, "USER&ADMIN"));
event.put(new Tuple("location", "Maryland", "ADMIN"));
```

Here we've constructed an event to model the health status of a system. In this scenario, assume we have systems at several locations throughout an enterprise fabric. Perhaps we have a couple clouds located throughout the country but only users with ADMIN privileges can see those locations. 

Also notice that tuple values aren't just limited to being strings. Most standard java primitives are supported (with the exception of arrays). When this event gets stored in Accumulo, the value of the tuple will keep datatype information with it so that queries that are performed will know exactly the datatype of each value in which to search. Ths is important as well because of the lexicographically sorted nature of Accumulo keys. When the value of a tuple is supposed to be representing a number, it's important that I don't put that number in a string because 10 will sort before 2 in the lexicographic order of the bytes. This may not seem like a big deal until I want to search for numbers 1-10 and find that only 1 and 10 were returned and not 2-9.

##How do I store events?

Like many of the other recipes, the first thing you need is a Connector object to an Accumulo instance
```java
Instance instance = new MockInstance();
Connector connector = instance.getConnector("root", "secret".getBytes());
```

Next you will need to create an EventStore instance and save your event
```java
AccumuloEventStore store = new AccumuloEventStore(connector);
store.save(event);
```

Almost seemed too easy. What's really happening underneath is that a discrete shard has been created for the RowID of the underlying Accumulo table. The shard is created to guarantee even distribution of the event over the nodes in the Accumulo cluster. Aside from the shard, the key and value have been normalized so that they can be searched using the built-in query mechanism.

It's important, however, that you pre-split the Accumulo table. 

##Pre-splitting the Accumulo table

If the event table isn't pre-split, all shards will end up on the same physical tablet until it gets big enough and Accumulo decides to split it into two. This won't allow the store distribute its data randomly across the cluster. Let's go ahead and split the table. There's a class provided in org.calrissian.eventstore.cli called ShardSplitter. It's an executable class but you'll need to pass in as arguments your Accumulo configuration. Since events are all time-based, a time-range for which to the split the table needs to be passed in as well. This allows shards to be split every morning, for instance, or every hour.

```java
java -cp accumulo-recipes-<version>.jar org.calrissian.eventstore.cli.ShardSplitter <zookeepers> <instance> <username> <password> <tableName> <start day: yyyy-mm-dd> <stop day: yyyy-mm-dd>
```