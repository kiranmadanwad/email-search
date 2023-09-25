# Email Search Application README

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Component Descriptions](#component-descriptions)
   - [Load Balancer](#load-balancer)
   - [Cassandra Cluster](#cassandra-cluster)
   - [Redis Cache](#redis-cache)
   - [Kafka Queue](#kafka-queue)
   - [Application Servers](#application-servers)
4. [Backup and Disaster Recovery](#backup-and-disaster-recovery)
5. [Conclusion](#conclusion)

## Introduction
The email search application aims to provide fast email search functionality with a target response time of less than 300ms.

## System Architecture
The architecture consists of several key components:
![HLD](https://github.com/kiranmadanwad/email-search/assets/29003308/83dfbc98-5d86-4f56-a26f-c3be44a1eeca)
### Load Balancer
- Kubernetes can be an excellent choice for handling load balancing in a scalable and containerized environment. Kubernetes provides a built-in solution for load balancing through its service abstraction. Here's how we can use Kubernetes for load balancing in this email/domain search system:

- Kubernetes Service:
Kubernetes Services are used to expose applications running inside the cluster to the external world or to other applications within the cluster. Services provide load balancing, service discovery, and routing traffic to appropriate pods based on labels and selectors.

- To use Kubernetes for load balancing, we can:

  a. Deploy Application Pods:
  Deploy Spring Boot-based or python flask application servers as pods within your Kubernetes cluster. Ensure that your application is containerized, and you have a Docker image for it.
  
  b. Create a Kubernetes Service:
  Define a Kubernetes Service that targets your application pods. You can create a service of type LoadBalancer to enable external access to your application.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```
In this example:

selector specifies which pods the service should target. Ensure that application pods have the appropriate labels (app: my-app in this case).
ports define the port mapping between the service and the pods.
type is set to LoadBalancer, which instructs Kubernetes to create a cloud load balancer or use an internal load balancer to distribute traffic to your pods.

  c. Load Balancing Configuration:
  The load balancer created by Kubernetes will distribute incoming traffic to the pods behind it. The specific load balancing algorithm used depends on the underlying cloud provider's capabilities.
  
  d. Expose the Load Balancer:
  Once the service is created with type LoadBalancer, Kubernetes will interact with the cloud provider's load balancing service to create an external IP or DNS name. Clients can then access application using this IP or DNS.

### Cassandra Cluster
- Stores email data with efficient data modeling and partitioning.
- Cassandra Configuration:
Set the cassandra.yaml file for cluster settings, including node configuration, data directory, and heap size.
Data Replication:
Define replication strategies and factors to ensure data durability and high availability.
Example (Cassandra CQL):
```sql
CREATE KEYSPACE IF NOT EXISTS my_keyspace
    WITH replication = {
        'class': 'NetworkTopologyStrategy',
        'datacenter1': 3
    };
```
Designing a data model for email storage in Cassandra involves understanding the query patterns and access patterns expected in this application. Based on problem statement, I have designed a simplified Cassandra data model for storing emails by domain. I'll also consider some optimization techniques.

- Data Model:
In this example, we will store email records with the following attributes:

```
email_id (TEXT): A unique identifier for each email.
domain (TEXT): The domain associated with the email address.
content (TEXT): The content of the email.
timestamp (TIMESTAMP): The timestamp when the email was received or sent.
```
The primary goal is to optimize the system for searching emails by domain.

Table: email_records
```
sql
CREATE TABLE IF NOT EXISTS email_records (
    domain TEXT,
    email_id TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    PRIMARY KEY (domain, email_id)
);
```
In this data model:

We use the domain as the partition key to distribute data across nodes evenly. This allows for efficient retrieval of emails by domain.
email_id is used as the clustering key to ensure uniqueness within a domain.
Secondary indexes can be created on other fields (e.g., timestamp) if you have specific query requirements.
Optimization Techniques:
- Let's look at some optimization techniques:

  - Data Modeling: The data model is designed to efficiently support email/domain searches.

  -  Indexing Strategies: We can consider adding secondary indexes on fields like timestamp if we frequently query emails within a specific time range.

  -  Query Optimization: Design queries to avoid full table scans and use appropriate consistency levels. For example, to retrieve emails within a specific domain:

```
sql
SELECT * FROM email_records WHERE domain = 'example.com' LIMIT 10;
```
Partitioning Strategy: domain-based partitioning. Each unique email domain will be a partition key, and all email records for that domain will be stored together within that partition.

Data Model:
```
sql
CREATE TABLE IF NOT EXISTS email_records (
    domain TEXT,
    email_id TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    PRIMARY KEY (domain, email_id)
);
```
domain is the partition key.
email_id is the clustering key to ensure uniqueness within a domain.
With this strategy:

Each unique domain will have its own partition.
All email records for a specific domain will be stored together within that partition.
Cassandra will automatically distribute these partitions across the nodes in the cluster.

Compression and Compaction: Configure data compression and compaction settings for efficient storage and query performance.


### Redis Cache
- Using Redis for Caching in Email Search:
- Cache Strategy: Determine which email records should be cached. For example, you might want to cache frequently searched emails or frequently accessed domains.

- Cache Key Design: Define a cache key structure that uniquely identifies the data you want to cache. In this case, it might be based on email IDs or domains. For example:

- Cache email records by email ID: email:id:<email_id>
- Cache email records by domain: email:domain:<domain>
- Cache Population: When performing a search query, first check if the data is available in the Redis cache using the cache key. If it's present, retrieve the data from Redis and return it. If not, fetch the data from Cassandra, store it in Redis, and return it to the client.

- Cache Expiration: Consider setting an expiration time for cached data based on how frequently your data changes. For email data that rarely changes, you can cache it for a longer duration to maximize cache hits.
Specify eviction policies (e.g., LRU, LFU) and set the maximum cache size.

- Cache Invalidation: Implement cache invalidation strategies to ensure that cached data is updated when it changes in the Cassandra database. This can be done by deleting or updating the corresponding cache keys when data is modified.

By using Redis as a caching mechanism in this way, we can reduce the load on your Cassandra cluster and significantly improve the speed of email search queries, especially for frequently accessed data. Redis' in-memory storage and efficient caching mechanisms make it well-suited for this purpose.

### Kafka Queue
- To make data ingestion from Kafka parallel and optimize the entire process, we can implement parallelism at various levels in our data pipeline. Parallel processing allows us to handle larger volumes of data more efficiently, which can help maintain sub-300ms response times for email search. Here are some strategies for achieving parallelism in data ingestion from Kafka:

- Kafka Consumer Groups:

- Use Kafka Consumer Groups: Kafka allows you to create consumer groups with multiple consumer instances. Each instance in a consumer group can process data in parallel from different Kafka partitions.

- Increase the number of consumer instances: By adding more consumer instances to a consumer group, you can parallelize the processing of data from multiple partitions. Ensure that the number of consumer instances matches the number of partitions in your Kafka topic for optimal parallelism.

- Parallel Message Processing:

Implement parallel processing within each consumer instance: Each Kafka message can be processed in parallel using multi-threading or multiprocessing within your consumer code. This can be especially useful if processing a single message involves complex operations.

- Use worker pools: Consider using worker pools to distribute Kafka message processing across multiple worker threads or processes.

- Batch Processing:

- Process messages in batches: Instead of processing one message at a time, collect a batch of messages and process them together. This reduces the overhead of message handling and can improve throughput.

Adjust the batch size: Experiment with different batch sizes to find the optimal trade-off between throughput and processing latency.
### Application Servers
- Develop Spring Boot microservices responsible for handling search queries.
- Implement parallel search processing by dividing the search workload into tasks handled by multiple threads or processes.
- Use connection pooling to efficiently manage connections to the Cassandra database.
- Implement error handling and retries for resilience.


## Backup and Disaster Recovery
- Handling disaster recovery for your email search application, especially when using managed services like Cassandra and Redis in a Kubernetes environment, is essential to ensure the continuity of your service. Here are steps and strategies to implement a robust disaster recovery plan:
- Data Backup and Replication:
  
  a. Cassandra: Set up regular backups of your Cassandra data. Most managed Cassandra services provide automated backup and restore functionality. Ensure backups are taken at frequent intervals and stored securely. Implement Cassandra's built-in replication features, such as replication factor and data center replication, to ensure data redundancy across multiple nodes and data centers. This helps prevent data loss in the event of a node or data center failure.

  b. Redis Cache: Configure Redis to take snapshots of your data at regular intervals. You can use Redis' persistence options like RDB snapshots and AOF logs.Consider using Redis' replication feature to replicate data to multiple Redis instances or clusters. This provides data redundancy and high availability.

- Data Recovery and Failover:
  
  a. Cassandra: Set up a disaster recovery site with a secondary Cassandra cluster in a different region or data center. Use asynchronous replication to replicate data to the secondary cluster.
In the event of a primary cluster failure, switch traffic to the secondary cluster. Ensure that your application can automatically detect the primary cluster's unavailability and fail over to the secondary cluster.

  b. Redis Cache: Implement Redis Sentinel or Redis Cluster for high availability and automatic failover. These features monitor Redis instances and promote a replica to the primary role if the primary instance fails.
Consider implementing a read-only replica in a different region to improve read performance and minimize latency in case of a failover.

## Conclusion
The email search application's architecture and strategies, including efficient data modeling, caching, monitoring, and disaster recovery planning, collectively contribute to achieving the 300ms response time target, ensuring a responsive and reliable user experience.
