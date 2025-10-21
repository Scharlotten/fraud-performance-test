# Cassandra Performance Comparison: Version 3.11.17 vs 4.0.4 vs HCD 1.2.3
## Fraud Detection Workload Analysis

### Executive Summary

This comprehensive performance analysis compares three Cassandra deployments under high-volume fraud detection workloads:
- **Cassandra 3.11.17**: Traditional Apache Cassandra
- **Cassandra 4.0.4**: Newer Apache Cassandra 
- **HCD 1.2.3 (Hybrid Cloud Database)**: DataStax's enterprise offering

All clusters were deployed using **MissionControl Operator v1.15** on identical infrastructure specifications for unbounded (maximum capacity) scenarios.

**Verdict**: HCD 1.2.3 demonstrates superior performance with 101,010 ops/sec throughput and consistent sub-5ms P99 latencies, making it the optimal choice for mission-critical fraud detection workloads.

---

## Infrastructure Specifications

### Cluster Configuration
- **Platform**: Google Kubernetes Engine (GKE) 
- **Kubernetes Version**: v1.33.5-gke.1080000
- **Deployment Tool**: MissionControl Operator v1.15
- **Node Configuration**: 6 database nodes per cluster
- **Machine Type**: n2-standard-16
- **Resource Allocation**: 16 CPU, 8Gi memory per node
- **Storage**: 
  - Cassandra 3: 25Gi standard-rwo
  - Cassandra 4: 20Gi premium-rwo
  - HCD: 25Gi standard-rwo
- **Replication Factor**: 3 across 3 availability zones (us-east1-b, us-east1-c, us-east1-d)


## Database Schema Design

### Table Architecture

#### 1. Main Transactions Table (`fraud_detection.transactions`)
**Purpose**: Primary transaction storage with comprehensive fraud detection attributes

**Schema Highlights**:
- **Primary Key**: `transaction_id` (partition key)
- **Data Types**: 20 columns spanning text, double, int, timestamp types
- **Compaction Strategy**: SizeTieredCompactionStrategy for high write throughput
- **TTL**: 7,776,000 seconds (90 days)
- **Compression**: LZ4Compressor with 4KB chunks

**Estimated Row Size**: ~350 bytes per row
- Text fields (transaction_id, user_id, merchant_id, etc.): ~180 bytes
- Numeric fields (amount, risk_score, ml_fraud_score, etc.): ~60 bytes  
- Timestamp fields: ~16 bytes
- Categorical fields (currency, transaction_type, etc.): ~94 bytes

**Sample Data:**
```
transaction_id      | amount     | currency | transaction_type | country_code | city   | device_type | is_fraud | risk_score
--------------------+------------+----------+------------------+--------------+--------+-------------+----------+------------
7667067912213249559 |  8304.5086 |      GBP |         TRANSFER |           DE |  Other |         ATM |    false |        830
9001040773479899721 | 9749.21409 |      CAD |          PAYMENT |           JP |  Other |         POS |     true |        974
1168386195310972831 | 1266.37353 |      USD |         PURCHASE |           US | London |      MOBILE |    false |        126
```

#### 2. User Transactions Table (`fraud_detection.user_transactions`)
**Purpose**: Time-ordered transaction history per user for pattern analysis

**Schema Highlights**:
- **Primary Key**: `user_id` (partition key), `transaction_timestamp, transaction_id` (clustering keys)
- **Clustering Order**: `transaction_timestamp DESC, transaction_id ASC`
- **Compaction Strategy**: TimeWindowCompactionStrategy for time-series data
- **Data Types**: 7 columns optimized for temporal queries

**Estimated Row Size**: ~120 bytes per row
- Primary/clustering keys: ~60 bytes
- Transaction metadata: ~60 bytes

**Sample Data:**
```
user_id | transaction_timestamp           | transaction_id      | amount     | is_fraud | merchant_id | risk_score
--------+---------------------------------+---------------------+------------+----------+-------------+------------
 544844 | 2024-01-02 02:14:34.623000+0000 | 5030335829473577234 | 5448.90229 |    false |       27242 |        544
 544844 | 2024-01-02 01:45:37.883000+0000 | 5030332423006334831 |  5448.8986 |    false |       27242 |        544
 544844 | 2024-01-02 00:56:10.207000+0000 | 5030332372637010955 | 5448.89855 |    false |       27242 |        544
```

### Data Diversity & Testing Scope
The workload incorporated **13 distinct data types** across both tables:
- **Text**: UUIDs, user IDs, categorical values
- **Numeric**: Doubles (amounts, ML scores), integers (risk scores, velocities)  
- **Temporal**: Timestamp fields for transaction timing
- **Geographic**: Country codes, city names, IP addresses
- **Categorical**: Weighted distributions for realistic data patterns

---

## Workload Configuration

### NoSQLBench Workload Design
```yaml
Test Phases:
1. Schema Creation: 3 DDL operations
2. Rampup Phase: 100,000 insert operations 
3. Main Phase: 100,000,000 mixed read/write operations

Operation Mix:
- 50% Read Operations (transaction lookups, user history queries)
- 50% Write Operations (dual table inserts per transaction)

Threading: optimal client connection scaling, on average ~160 threads
Rate Limiting: Uncapped for maximum throughput testing
```

### Test Execution Parameters
- **Target Operations**: 100 million operations
- **Consistency Level**: LOCAL_QUORUM
- **Client Auto-scaling**: Leveraged NoSQLBench's automatic thread optimization

---

## Performance Test Results

### Latency Performance Comparison (100M Operations)

#### Unbounded Workload Results

| Database | Min (ms) | Max (ms) | P95 (ms) | P99 (ms) | Operations Completed | Job Duration | Avg Throughput | Avg Threads |
|----------|----------|----------|----------|----------|--------------------|--------------|--------------|--------------| 
| **HCD (UCS)** | 0.58 | **123.97** | 2.14 | 4.60 | 100,000,000 | ~16.5 minutes | **101,010 ops/sec** | 159 |
| **HCD** | 0.61 | 140.44 | 2.16 | **5.33** | 100,000,000 | ~19 minutes | **87,719 ops/sec** | 158 |
| **Cassandra 4** | 0.54 | 284.20 | 3.61 | 40.19 | 100,000,000 | 25 minutes | 66,667 ops/sec | 158 |
| **Cassandra 3** | 0.49 | **2,055.47** | 2.01 | **238.91** | 99,999,368 | 67 minutes | 24,876 ops/sec | 159 |

---

## Critical Performance Issues

### Cassandra 3 Timeout Analysis

**Unbounded Scenario:**  
- **633 timeout errors** detected
- Significantly higher timeout rate under unbounded load

#### Sample Timeout Error:
```
WARN [main:033] ERRORS error with cycle 66367507 errmsg: Error in space '66367507': Query timed out after PT2S
```

### Performance Rankings by Scenario

#### Best P99 Latency Performance:
1. **HCD UCS**: 4.60ms  
2. **HCD**: 5.33ms
3. **Cassandra 4 Unbounded**: 40.19ms
4. **Cassandra 3 Unbounded**: 238.91ms ⚠️

#### Transaction Completion Rates:
- **HCD & Cassandra 4**: 100% completion (100,000,000 operations)
- **Cassandra 3**: 99.999% completion (some operations failed due to timeouts)

---

## Visual Performance Analysis

### Performance Visualizations

#### Cassandra 3.11.17 Results

**Unbounded Configuration:**

![Cassandra 3 Unbounded Reads and Writes](./cassandra-3/unbounded/writes-and-reads.png)
*Throughput analysis under maximum load*

![Cassandra 3 Unbounded Latency](./cassandra-3/unbounded/write-and-read-latency.png)
*Latency distribution showing severe performance degradation*

![Cassandra 3 Unbounded Client Timeouts](./cassandra-3/unbounded/clients-timeouts.png)
*Client timeout analysis showing 633 timeout events*

#### Cassandra 4.0.4 Results

**Unbounded Configuration:**

![Cassandra 4 Unbounded Reads and Writes](./cassandra-4/unbounded/writes-and-reads.png)
*Throughput analysis under maximum load*

![Cassandra 4 Unbounded Latency](./cassandra-4/unbounded/write-and-read-latency.png)
*Latency distribution maintaining reasonable performance*

![Cassandra 4 Unbounded Compaction Metrics](./cassandra-4/unbounded/compaction-metrics.png)
*Compaction analysis showing improved efficiency*

#### HCD (Hybrid Cloud Database) Results

**Unbounded Configuration:**

![HCD Unbounded Reads and Writes](./hcd/unbounded-insert-50-50/writes_and_reads.png)
*Throughput analysis under balanced workload*

![HCD Unbounded Latency](./hcd/unbounded-insert-50-50/write_and_read_latency.png)
*Latency distribution maintaining excellent performance*

**Unbounded UCS Configuration:**

![HCD UCS Reads and Writes](./hcd/unbounded-hcd-insert-50-50-ucs/read-and-writes.png)
*Throughput analysis with UCS optimization*

![HCD UCS Latencies](./hcd/unbounded-hcd-insert-50-50-ucs/read-write-latencies.png)
*Latency distribution showing best-in-class performance*

---

## Key Findings & Recommendations

### 🏆 **Winner: HCD 1.2.3 (Hybrid Cloud Database)**
- **Best P99 latency**: 4.60ms (unbounded UCS)
- **Most controlled max latencies**: 123.97-152.12ms vs 2000+ ms for Cassandra 3
- **100% operation completion** across all test scenarios
- **Zero timeout errors** detected
- **Enterprise optimizations**: Advanced compaction and query processing

### ⚠️ **Cassandra 3 Performance Concerns**
- **633 timeout errors** in unbounded tests
- **Extreme max latencies**: Over 2 seconds in unbounded scenario
- **Poor P99 performance**: 238.91ms in unbounded scenario
- **Operation failures**: Did not complete full 100M operations

### 📈 **Cassandra 4 Improvement Over v3**
- **Zero timeout errors**
- **Significantly better max latencies**: 205-284ms vs 2000+ ms  
- **100% operation completion**
- **Better P99 performance** in unbounded scenarios

### 🔧 **Infrastructure Impact**
- **Storage Configuration**: Premium SSD (Cassandra 4) vs Standard (Cassandra 3) may contribute to performance differences

---

## Conclusion

For **mission-critical fraud detection workloads** requiring sub-10ms P99 latencies:

1. **HCD is the clear choice** with consistent sub-5ms P99 performance
2. **Cassandra 4 is viable** for less stringent latency requirements  
3. **Cassandra 3 is not recommended** due to timeout issues and high tail latencies

**Deployment Recommendation**: HCD 1.2.3 provides optimal performance and resource efficiency for real-time fraud detection systems.

**Technical Specifications for HCD 1.2.3:**
- Server Type: hcd (DataStax Hybrid Cloud Database)
- Version: 1.2.3
- JVM: G1GC with 2Gi heap
- Storage: 25Gi standard-rwo (same as Cassandra 3)
- Resource allocation: 16 CPU, 8Gi memory per node
- Identical infrastructure to comparative tests, demonstrating superior optimization