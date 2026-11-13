### Questions:

1. The credentials are temporary. What happens to a running Spark job if the service is restarted mid-lab, and why?

The running Spark job fails or loses connection to S3 because the temporary STS credentials provided by Onyxia expire or become invalid when the container/service restarts.

2. Why does a plain SimpleAWSCredentialsProvider fail here, while it worked in setups without a session token?

A SimpleAWSCredentialsProvider only handles access and secret keys, ignoring the AWS_SESSION_TOKEN. Because Onyxia uses temporary credentials that explicitly require a session token for authentication, it fails unless the TemporaryAWSCredentialsProvider is used.

3. Explain the output of the code that was just run.

It groups users by state, counts total users per state, and counts how many users have a missing (NULL) city (e.g., military addresses like AE or AP), sorted by user count in descending order.   

4. Compare the DataFrame and SQL cells above. Which one would you choose for a one-off exploration, and which for a model reused by several colleagues? Why?

One-off exploration: Spark SQL or DataFrame API, because both allow fast, interactive ad-hoc testing.   
Reused model: Spark SQL (or dbt SQL), because standard SQL is easier to read, maintain, and share across teams regardless of programming background.   

5. regexp_extract() returned an empty string instead of NULL on no match. Where else in this notebook could that same behavior silently produce a wrong result if left unhandled?

In aggregation operations like GROUP BY city or COUNT(city), where an empty string ("") would be incorrectly counted as a valid distinct city category instead of being excluded or ignored like a true NULL.

6. List the objects written under silver/users.parquet/. How many part-files are there, and why?

There is 1 part-file (part-00000-...) because the small dataset is processed within a single active partition.   

7. coalesce(1) before .write would produce a single file. What do you lose by doing that on a larger dataset?

We lose distributed parallelism, create a massive performance bottleneck, and risk out-of-memory errors because all data is funneled through a single task/executor.   

8. Compare silver/orders.parquet/ and warehouse/silver/orders/. Which files hold the data, and which ones describe the table?

The .parquet files in data/ hold the actual data. The files in metadata/ (v1.metadata.json, .avro manifests, and version-hint.text) describe the table schema and snapshots.

9. A second job overwrites the Parquet directory while a query is reading it. What does the reader see? What changes with an Iceberg table?

For a plain Parquet directory, the reader sees a FileNotFoundError or inconsistent/corrupted data mid-query. With Iceberg, the reader safely reads the unchanged current snapshot because writes create new files and commit atomically.

10. With this catalog, the pointer to the current version of a table is a file in the bucket. What can go wrong if two jobs commit at the same time, and what would a catalog service add?

Two concurrent jobs using a file-based pointer can cause race conditions or commit overwrites (lost updates). A central catalog service (like Hive Metastore, Nessie, or AWS Glue) adds atomic locking/optimistic concurrency control to prevent conflicting commits.