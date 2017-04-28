
Kafka Streams API
-----------------

---

What
-------

+ Real-time join of data streams based on a shared key
+ leverage the Kafka Streams API, new since ~ version 0.10

---

Why
--------

+ ODC use cases requiring joining across multiple ID spaces
+ Hydra is clunky for joining 2 data streams

---

Why Kafka
--------

+ Hydra already integrates with Kafka
+ we already use Kafka message queues

---

Features
--------

---

How
--------

---

Code
--------

+++

Init
--------

```java
        // In the subsequent lines we define the processing topology of the
        // Streams application.
        final KStreamBuilder builder = new KStreamBuilder();

        final KStream<String, String> hhidMappings = builder.stream("HHIDMappingTopic");
        final KStream<String, String> sdtFeed = builder.stream("SDTFeedTopic");
```

+++

Transform
--------

```java
        final KTable<String, String> hhidMappingsTable = hhidMappings
                .filter((key, value) -> value != null && value.length() > 0)
                .mapValues(value -> Arrays.asList(value.split("\t")))
                .filter((key, value) -> value.size() > 1 && value.get(0) != null && !value.get(0).isEmpty())
                .filter((key, value) -> Math.abs(SuperFastHash.hash(value.get(0))) % 1000 < 10)
                .map((key, value) -> KeyValue.pair(value.get(0), value.get(1))).groupByKey()
                .reduce((aggValue, newValue) -> newValue, "hhid-mapping-store2")
                ;

        final KStream<String, String> sdtFeedTransform = sdtFeed
                .filter((key, value) -> value != null && value.length() > 0)
                .mapValues(value -> Arrays.asList(value.split("\t")))
                .filter((key, value) -> value.size() > 3 && value.get(2) != null && !value.get(2).isEmpty())
                .filter((key, value) -> Math.abs(SuperFastHash.hash(value.get(2))) % 1000 < 10)
                .map((key, value) -> KeyValue.pair(value.get(2), value.get(3)))
                ;
```

+++

Join (Table Lookup semantics)
--------

```java
        final KStream<String, String> sdtJoinHhid = sdtFeedTransform.leftJoin(hhidMappingsTable,
                (left, right) -> left + "," + right);

        // Write the `KStream<String, String>` to the output topic.
        sdtJoinHhid.to(stringSerde, stringSerde, "JoinedOutputTopic");

```

+++

Start
--------

```java
        final KafkaStreams streams = new KafkaStreams(builder, streamsConfiguration);

        streams.cleanUp();
        streams.start();

```

---

Results
--------

+++

Did it work?
--------

√ yeah!

+++

Scalable?
--------

???

+++

Performant?
--------

???

---

Next Steps
--------

+ git repo
+ feature exploration
+ docker/k8s
+ hydra integration
+ scalability/performance testing
