### Questions:

1. What decides the number of partitions textFile() picks by default, for a file this size?

Yes, the Spark UI shows a new job and execution stages. Because RDDs use lazy evaluation and are un-cached by default, calling an action (count()) forces Spark to re-evaluate the entire lineage graph from the source files every single time.

2. repartition() and coalesce() both change the partition count. Which one shuffles data, and which one doesn't?

Calling .cache() right after filtering—like orders_data = orders_lines.filter(...).cache()—would change this behavior. After the first action computes and persists the RDD in memory, subsequent actions retrieve the data directly from memory/storage instead of re-running the upstream transformations.

3. Call orders_data.count() a second time. Does the Spark UI show a new stage? Why?

Yes, the Spark UI shows a new job and execution stages. Because RDDs are evaluated lazily and are un-cached by default, every action (count()) re-evaluates and recomputes the entire lineage graph from the source files.

4. At which point in this notebook would .cache() change that answer?

Calling .cache() directly on the filter transformation (e.g., orders_data = orders_lines.filter(...).cache()) would change this behavior. Once the first action computes the RDD and persists it in memory, subsequent actions retrieve the data directly from cache rather than re-running upstream transformations.   

5. Why does spark.read.csv(multiLine=True) not have this problem, while sc.textFile() does?

spark.read.csv(multiLine=True) understands structured CSV formats and parsing rules, including quoted fields that can span across multiple physical lines. In contrast, sc.textFile() is a simple line-oriented reader that treats every single physical newline (\n) as a distinct record, ignoring CSV quote semantics and splitting multiline fields into separate rows.   

6. In general terms, what has to be true of a file format for a line-oriented reader like textFile() to be safe to use on it?

The file format must be strictly record-per-line, meaning each individual record corresponds to exactly one physical line, and newline characters never appear internally within any field or value of that record.

7. What would happen to this cell if one line in orders.csv had an extra comma inside product?

It would raise a ValueError: too many values to unpack (because line.split(",") would produce more than 5 elements, breaking the explicit 5-variable unpacking). This failure would happen at execution time when an action touches that data, rather than at read/parsing definition time.

8. Where does the DataFrame API perform the equivalent check, and when?

The DataFrame API handles schema enforcement and parsing checks during the reading phase (or upon execution of an action if parsing fails, depending on the mode like PERMISSIVE or DROPMALFORMED), automatically mapping columns based on a defined schema or inferred structure rather than fragile manual string splitting.

9. reduceByKey combines values on each partition before shuffling. What does groupByKey shuffle instead, and why is that more expensive?

groupByKey shuffles all raw key-value pairs across the network without combining or aggregating them locally first. This is significantly more expensive because it transfers a much larger volume of data over the network, increasing disk I/O and network congestion.

10. For which aggregations would groupByKey be unavoidable, reduceByKey not being able to express them?

groupByKey is unavoidable for operations where the aggregation or function requires access to the entire collection or list of grouped values at once (such as calculating the median, finding top-N items per group, collecting distinct elements into a list, or operations where the combining function is non-associative/non-commutative).

11. partitionBy() in the window function is described as a wide transformation, shuffling rows so every row of a given order_id lands on the same partition. What plays that role here?

The combination of .map(lambda o: (o["order_id"], o)) and .reduceByKey(most_recent) plays that role. The reduceByKey operation triggers a shuffle based on the key (order_id), ensuring that all rows with the same order identifier are routed to the same partition.   

12. Which version states the intent more directly: the window function, or the reduceByKey with a custom combiner?

The window function states the intent much more directly and declaratively. Using reduceByKey with a custom combiner is a lower-level, workaround approach to achieve row deduplication.   

13. join() is a wide transformation on both RDDs. What would need to be true of how orders_kv and users_kv are partitioned for Spark to skip the shuffle?

Both RDDs would need to be co-partitioned using the same partitioner (e.g., a hash partitioner on the same key) and have the same number of partitions, ensuring that records with the same key already reside on the exact same worker node.

14. Replace join() with leftOuterJoin(). What changes in the result if a user_id in orders has no matching user?

With leftOuterJoin(), orders that have no matching user_id in users_kv will still be included in the results, but their corresponding user value will appear as None (or null) instead of being dropped entirely.   

15. enriched is not yet filtered or aggregated. What is the risk of calling .collect() on it directly, on a dataset much larger than this one?

Calling .collect() brings the entire distributed dataset from all executor nodes back into the driver node's memory. On a large dataset, this creates a severe Driver Out-Of-Memory (OOM) error, crashing the Spark driver application because the driver cannot handle storing all the data locally.   