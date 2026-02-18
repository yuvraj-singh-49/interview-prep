# AWS Data & Analytics: Kinesis, Athena, Glue

## Overview

```
Data Analytics Landscape on AWS:

┌─────────────────────────────────────────────────────────────────────────────┐
│                          Data Flow                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Sources              Ingestion          Processing         Analytics     │
│                                                                             │
│  ┌─────────┐       ┌─────────────┐     ┌─────────────┐    ┌─────────────┐  │
│  │ Apps    │──────►│ Kinesis     │────►│ Kinesis     │───►│ Athena      │  │
│  │ IoT     │       │ Data Streams│     │ Analytics   │    │ (SQL)       │  │
│  │ Logs    │       └─────────────┘     └─────────────┘    └─────────────┘  │
│  │ Events  │                                                    │          │
│  └─────────┘       ┌─────────────┐                              │          │
│       │            │ Kinesis     │                              ▼          │
│       └───────────►│ Firehose    │────────────────────────►┌─────────┐    │
│                    └─────────────┘                          │   S3    │    │
│                                                             │  (Lake) │    │
│  ┌─────────┐       ┌─────────────┐     ┌─────────────┐     └─────────┘    │
│  │ Batch   │──────►│ Glue        │────►│ Glue ETL    │          │          │
│  │ Data    │       │ Crawlers    │     │ Jobs        │          │          │
│  │ (S3)    │       └─────────────┘     └─────────────┘          ▼          │
│  └─────────┘                                              ┌─────────────┐  │
│                                                           │ QuickSight  │  │
│                                                           │ (Dashboard) │  │
│                                                           └─────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Amazon Kinesis

### Kinesis Data Streams

```
Architecture:

Producers ────► Kinesis Data Stream ────► Consumers

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kinesis Data Stream                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        Data Stream                                   │  │
│   ├─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬────────┤  │
│   │ Shard 1 │ Shard 2 │ Shard 3 │ Shard 4 │ Shard 5 │ Shard 6 │  ...   │  │
│   │         │         │         │         │         │         │        │  │
│   │ 1 MB/s  │ 1 MB/s  │ 1 MB/s  │ 1 MB/s  │ 1 MB/s  │ 1 MB/s  │        │  │
│   │ write   │ write   │ write   │ write   │ write   │ write   │        │  │
│   │ 2 MB/s  │ 2 MB/s  │ 2 MB/s  │ 2 MB/s  │ 2 MB/s  │ 2 MB/s  │        │  │
│   │ read    │ read    │ read    │ read    │ read    │ read    │        │  │
│   └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴────────┘  │
│                                                                             │
│   Partition Key → Hash → Shard Assignment                                   │
│   Records with same partition key → same shard → ordering guaranteed       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Key Concepts:
├── Shard: Base throughput unit (1MB/s in, 2MB/s out)
├── Partition Key: Determines which shard receives record
├── Sequence Number: Unique ID per record within shard
├── Retention: 24 hours (default) to 365 days
└── Data Record: Max 1MB (partition key + data + sequence number)
```

### Capacity Modes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Provisioned vs On-Demand                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provisioned Mode:                    On-Demand Mode:                       │
│  ├── You specify shard count          ├── Auto-scales                      │
│  ├── Manual scaling                   ├── No capacity planning             │
│  ├── Pay per shard-hour               ├── Pay per GB ingested              │
│  ├── Predictable cost                 ├── Variable cost                    │
│  └── Good for steady workloads        └── Good for unpredictable traffic   │
│                                                                             │
│  Provisioned Pricing:                 On-Demand Pricing:                    │
│  ~$0.015/shard-hour                   ~$0.08/GB ingested                    │
│  + $0.014/1M PUT units                + $0.04/GB retrieved                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Capacity Planning (Provisioned):

Write capacity needed = (Records/second × Record size) / 1 MB/s per shard
Read capacity needed = (Consumers × Records/second × Record size) / 2 MB/s per shard

Example:
├── 10,000 records/second
├── Average record: 500 bytes (0.5 KB)
├── 3 consumer applications

Write shards = (10,000 × 0.0005) / 1 = 5 shards
Read shards = (3 × 10,000 × 0.0005) / 2 = 7.5 = 8 shards

Total shards needed = max(5, 8) = 8 shards
```

### Producer and Consumer Patterns

```python
# Producer - Using AWS SDK
import boto3
import json

kinesis = boto3.client('kinesis')

def produce_event(stream_name, event):
    response = kinesis.put_record(
        StreamName=stream_name,
        Data=json.dumps(event),
        PartitionKey=event['user_id']  # Same user → same shard → ordering
    )
    return response['SequenceNumber']

# Batch producer for higher throughput
def produce_batch(stream_name, events):
    records = [
        {
            'Data': json.dumps(event),
            'PartitionKey': event['user_id']
        }
        for event in events
    ]

    response = kinesis.put_records(
        StreamName=stream_name,
        Records=records  # Max 500 records or 5MB per batch
    )

    # Handle partial failures
    if response['FailedRecordCount'] > 0:
        for i, record in enumerate(response['Records']):
            if 'ErrorCode' in record:
                # Retry failed records with exponential backoff
                print(f"Failed record {i}: {record['ErrorCode']}")
```

```python
# Consumer - Kinesis Client Library (KCL) Pattern
from amazon_kclpy import kcl

class RecordProcessor(kcl.RecordProcessorBase):
    def __init__(self):
        self.checkpoint_freq = 60  # seconds

    def initialize(self, initialize_input):
        self.shard_id = initialize_input.shard_id
        self.largest_seq = None
        self.last_checkpoint = time.time()

    def process_records(self, process_records_input):
        for record in process_records_input.records:
            # Process the record
            data = json.loads(record.data)
            self.process_event(data)
            self.largest_seq = record.sequence_number

        # Checkpoint periodically
        if time.time() - self.last_checkpoint > self.checkpoint_freq:
            process_records_input.checkpointer.checkpoint(self.largest_seq)
            self.last_checkpoint = time.time()

    def process_event(self, event):
        # Business logic here
        print(f"Processing event: {event}")

# KCL handles:
# - Shard discovery and assignment
# - Load balancing across workers
# - Checkpointing
# - Fault tolerance
```

### Enhanced Fan-Out

```
Standard Consumers vs Enhanced Fan-Out:

Standard (Shared Throughput):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Shard (2 MB/s read)                                                       │
│         │                                                                   │
│         ├──────► Consumer A ◄─── Polling (GetRecords)                      │
│         ├──────► Consumer B     (Share 2 MB/s)                             │
│         └──────► Consumer C                                                │
│                                                                             │
│   With 5 consumers: each gets ~400 KB/s                                    │
│   Latency: ~200ms per poll                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Enhanced Fan-Out (Dedicated Throughput):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Shard ──┬──► Consumer A (2 MB/s dedicated) ◄─── Push (SubscribeToShard)  │
│           ├──► Consumer B (2 MB/s dedicated)                                │
│           └──► Consumer C (2 MB/s dedicated)                                │
│                                                                             │
│   Each consumer gets full 2 MB/s                                            │
│   Latency: ~70ms (push-based)                                               │
│   Max 20 consumers per stream                                               │
│   Additional cost: $0.015/consumer-shard-hour + $0.013/GB                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Kinesis Data Firehose

```
Firehose = Fully managed delivery stream (no shards, auto-scaling)

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kinesis Data Firehose                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Sources:                           Destinations:                          │
│   ├── Direct PUT                     ├── Amazon S3                         │
│   ├── Kinesis Data Streams           ├── Amazon Redshift (via S3)          │
│   ├── CloudWatch Logs                ├── Amazon OpenSearch                  │
│   ├── IoT                            ├── Splunk                            │
│   └── MSK                            ├── HTTP Endpoints                     │
│                                      └── Datadog, New Relic, etc.          │
│                                                                             │
│   Transformation (optional):                                                │
│   ├── Lambda function                                                       │
│   ├── Format conversion (JSON → Parquet/ORC)                               │
│   └── Compression (GZIP, Snappy, etc.)                                     │
│                                                                             │
│   Buffering:                                                                │
│   ├── Size: 1-128 MB                                                       │
│   └── Time: 60-900 seconds                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

# CloudFormation Example
FirehoseDeliveryStream:
  Type: AWS::KinesisFirehose::DeliveryStream
  Properties:
    DeliveryStreamType: DirectPut
    ExtendedS3DestinationConfiguration:
      BucketARN: !GetAtt DataLakeBucket.Arn
      RoleARN: !GetAtt FirehoseRole.Arn
      BufferingHints:
        IntervalInSeconds: 300
        SizeInMBs: 64
      CompressionFormat: GZIP
      Prefix: 'data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/'
      ErrorOutputPrefix: 'errors/'
      DataFormatConversionConfiguration:
        Enabled: true
        InputFormatConfiguration:
          Deserializer:
            JsonSerDe: {}
        OutputFormatConfiguration:
          Serializer:
            ParquetSerDe:
              Compression: SNAPPY
        SchemaConfiguration:
          RoleARN: !GetAtt GlueRole.Arn
          DatabaseName: !Ref GlueDatabase
          TableName: !Ref GlueTable
```

### Kinesis Data Analytics

```
Real-time analytics with SQL or Apache Flink:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kinesis Data Analytics                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SQL Mode (Legacy):                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   CREATE STREAM output_stream (                                     │  │
│   │     device_id VARCHAR(32),                                          │  │
│   │     avg_temperature DOUBLE,                                          │  │
│   │     reading_time TIMESTAMP                                          │  │
│   │   );                                                                │  │
│   │                                                                     │  │
│   │   CREATE PUMP output_pump AS                                        │  │
│   │   INSERT INTO output_stream                                         │  │
│   │   SELECT STREAM                                                     │  │
│   │     device_id,                                                      │  │
│   │     AVG(temperature) as avg_temperature,                            │  │
│   │     ROWTIME as reading_time                                         │  │
│   │   FROM source_stream                                                │  │
│   │   GROUP BY device_id,                                               │  │
│   │     STEP(source_stream.ROWTIME BY INTERVAL '1' MINUTE);            │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Apache Flink Mode (Current):                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   // Java/Scala Flink application                                   │  │
│   │   DataStream<Event> events = env                                    │  │
│   │       .addSource(new FlinkKinesisConsumer<>(...))                   │  │
│   │       .keyBy(event -> event.getDeviceId())                          │  │
│   │       .window(TumblingEventTimeWindows.of(Time.minutes(1)))         │  │
│   │       .aggregate(new AverageAggregate());                           │  │
│   │                                                                     │  │
│   │   events.addSink(new FlinkKinesisProducer<>(...));                  │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Amazon Athena

### Athena Overview

```
Athena = Serverless SQL query service for S3 data

┌─────────────────────────────────────────────────────────────────────────────┐
│                              Athena Architecture                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Query: SELECT * FROM logs WHERE date = '2024-01-15'                       │
│                           │                                                  │
│                           ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        Athena Engine                                 │  │
│   │              (Presto/Trino based, serverless)                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                           │                                                  │
│                           ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     AWS Glue Data Catalog                            │  │
│   │                                                                     │  │
│   │  Database: analytics                                                │  │
│   │  └── Table: logs                                                    │  │
│   │      ├── Schema: (timestamp, user_id, action, ...)                 │  │
│   │      ├── Location: s3://bucket/logs/                               │  │
│   │      ├── Format: Parquet                                           │  │
│   │      └── Partitions: year, month, day                              │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                           │                                                  │
│                           ▼                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         Amazon S3                                    │  │
│   │                                                                     │  │
│   │  s3://bucket/logs/                                                  │  │
│   │  └── year=2024/                                                     │  │
│   │      └── month=01/                                                  │  │
│   │          └── day=15/                                                │  │
│   │              ├── data-001.parquet                                   │  │
│   │              ├── data-002.parquet                                   │  │
│   │              └── ...                                                │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Pricing: $5 per TB scanned (with data savings from partitioning/Parquet) │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Athena Table Creation

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS analytics;

-- Create external table with partitions
CREATE EXTERNAL TABLE analytics.user_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,
    properties MAP<STRING, STRING>,
    timestamp TIMESTAMP
)
PARTITIONED BY (
    year STRING,
    month STRING,
    day STRING
)
STORED AS PARQUET
LOCATION 's3://my-bucket/events/'
TBLPROPERTIES (
    'parquet.compression' = 'SNAPPY',
    'projection.enabled' = 'true',
    'projection.year.type' = 'integer',
    'projection.year.range' = '2020,2030',
    'projection.month.type' = 'integer',
    'projection.month.range' = '1,12',
    'projection.month.digits' = '2',
    'projection.day.type' = 'integer',
    'projection.day.range' = '1,31',
    'projection.day.digits' = '2',
    'storage.location.template' = 's3://my-bucket/events/year=${year}/month=${month}/day=${day}/'
);

-- Partition projection eliminates need for MSCK REPAIR TABLE

-- Query with partition pruning
SELECT
    event_type,
    COUNT(*) as event_count
FROM analytics.user_events
WHERE year = '2024'
    AND month = '01'
    AND day BETWEEN '01' AND '15'
GROUP BY event_type
ORDER BY event_count DESC
LIMIT 10;
```

### Athena Performance Optimization

```
Cost and Performance Optimization:

1. Use Columnar Formats (Parquet/ORC)
   ┌────────────────────────────────────────────────────────────┐
   │  CSV:     Scan entire row    → 100 GB scanned → $0.50     │
   │  Parquet: Scan only columns  → 10 GB scanned  → $0.05     │
   │           (90% cost savings!)                              │
   └────────────────────────────────────────────────────────────┘

2. Partition Data
   Bad:  s3://bucket/logs/data.parquet (1 TB, full scan)
   Good: s3://bucket/logs/year=2024/month=01/day=15/ (10 GB scan)

3. Use Compression
   SNAPPY: Fast, moderate compression (recommended for Parquet)
   GZIP:   Better compression, slower
   ZSTD:   Best of both (newer)

4. Optimize File Sizes
   ├── Too small (< 128 MB): High overhead, slow
   ├── Too large (> 512 MB): Less parallelism
   └── Optimal: 128-256 MB per file

5. Use Partition Projection (avoid MSCK REPAIR)

6. CTAS for Optimizing Existing Data
   CREATE TABLE analytics.events_optimized
   WITH (
       format = 'PARQUET',
       parquet_compression = 'SNAPPY',
       partitioned_by = ARRAY['year', 'month', 'day'],
       bucketed_by = ARRAY['user_id'],
       bucket_count = 100
   ) AS
   SELECT
       event_id, user_id, event_type, properties, timestamp,
       year(timestamp) as year,
       month(timestamp) as month,
       day(timestamp) as day
   FROM analytics.events_raw;
```

### Athena Federated Queries

```
Query across multiple data sources:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Athena Federated Queries                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SELECT                                                                    │
│       s3_data.user_id,                                                      │
│       dynamo_data.preferences,                                              │
│       rds_data.account_status                                               │
│   FROM s3_data                                                              │
│   JOIN dynamo_data ON s3_data.user_id = dynamo_data.user_id                │
│   JOIN rds_data ON s3_data.user_id = rds_data.user_id                      │
│                                                                             │
│                           │                                                  │
│           ┌───────────────┼───────────────┐                                 │
│           ▼               ▼               ▼                                 │
│    ┌───────────┐   ┌───────────┐   ┌───────────┐                           │
│    │ Lambda    │   │ Lambda    │   │ Lambda    │                           │
│    │ Connector │   │ Connector │   │ Connector │                           │
│    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘                           │
│          │               │               │                                  │
│          ▼               ▼               ▼                                  │
│    ┌───────────┐   ┌───────────┐   ┌───────────┐                           │
│    │    S3     │   │ DynamoDB  │   │  RDS/     │                           │
│    │           │   │           │   │  Aurora   │                           │
│    └───────────┘   └───────────┘   └───────────┘                           │
│                                                                             │
│   Available Connectors: DynamoDB, RDS, Redshift, CloudWatch, Redis,        │
│                         HBase, DocumentDB, Neptune, and more               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## AWS Glue

### Glue Components

```
AWS Glue = Serverless ETL and Data Catalog

┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS Glue Components                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Data Catalog:                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Central metadata repository (Hive-compatible)                       │  │
│   │  ├── Databases                                                      │  │
│   │  │   └── Tables                                                     │  │
│   │  │       ├── Schema (columns, data types)                          │  │
│   │  │       ├── Location (S3, JDBC, etc.)                             │  │
│   │  │       ├── Partition keys                                        │  │
│   │  │       └── SerDe info                                            │  │
│   │  └── Connections (JDBC, Kafka, etc.)                               │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Crawlers:                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Automatically discover schema from data                             │  │
│   │  ├── Scans data sources (S3, databases)                            │  │
│   │  ├── Infers schema                                                  │  │
│   │  ├── Creates/updates tables in Data Catalog                        │  │
│   │  └── Detects partitions                                            │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ETL Jobs:                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Apache Spark or Python Shell                                        │  │
│   │  ├── Visual editor or code                                         │  │
│   │  ├── Auto-scaling workers                                          │  │
│   │  ├── Job bookmarks (incremental processing)                        │  │
│   │  └── Built-in transforms                                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   DataBrew (No-code):                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  Visual data preparation                                             │  │
│   │  ├── 250+ pre-built transforms                                     │  │
│   │  ├── Profile data quality                                          │  │
│   │  └── Recipe-based transformations                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Glue Crawler Configuration

```yaml
# CloudFormation - Glue Crawler
GlueCrawler:
  Type: AWS::Glue::Crawler
  Properties:
    Name: events-crawler
    Role: !GetAtt GlueCrawlerRole.Arn
    DatabaseName: !Ref GlueDatabase
    Schedule:
      ScheduleExpression: 'cron(0 * * * ? *)'  # Hourly
    Targets:
      S3Targets:
        - Path: s3://my-bucket/events/
          Exclusions:
            - '**/_temporary/**'
            - '**/.spark-staging/**'
    SchemaChangePolicy:
      UpdateBehavior: UPDATE_IN_DATABASE
      DeleteBehavior: LOG
    Configuration: |
      {
        "Version": 1.0,
        "Grouping": {
          "TableGroupingPolicy": "CombineCompatibleSchemas"
        },
        "CrawlerOutput": {
          "Partitions": {
            "AddOrUpdateBehavior": "InheritFromTable"
          }
        }
      }
```

### Glue ETL Job

```python
# Glue ETL Job (PySpark)
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame

# Initialize
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_database', 'source_table'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read from Data Catalog
source_dyf = glueContext.create_dynamic_frame.from_catalog(
    database=args['source_database'],
    table_name=args['source_table'],
    transformation_ctx="source_dyf",
    push_down_predicate="year='2024' and month='01'"  # Partition filtering
)

# Transform - Apply mappings
mapped_dyf = ApplyMapping.apply(
    frame=source_dyf,
    mappings=[
        ("event_id", "string", "event_id", "string"),
        ("user_id", "string", "user_id", "string"),
        ("event_type", "string", "event_type", "string"),
        ("timestamp", "string", "event_timestamp", "timestamp"),
        ("properties", "struct", "event_properties", "string")
    ],
    transformation_ctx="mapped_dyf"
)

# Transform - Filter
filtered_dyf = Filter.apply(
    frame=mapped_dyf,
    f=lambda x: x["event_type"] in ["purchase", "add_to_cart", "view_item"],
    transformation_ctx="filtered_dyf"
)

# Convert to DataFrame for complex transformations
df = filtered_dyf.toDF()

# Add derived columns
from pyspark.sql.functions import col, date_format, when

df_enriched = df.withColumn(
    "event_date", date_format(col("event_timestamp"), "yyyy-MM-dd")
).withColumn(
    "is_conversion", when(col("event_type") == "purchase", 1).otherwise(0)
)

# Convert back to DynamicFrame
enriched_dyf = DynamicFrame.fromDF(df_enriched, glueContext, "enriched_dyf")

# Write to S3 in Parquet format with partitioning
glueContext.write_dynamic_frame.from_options(
    frame=enriched_dyf,
    connection_type="s3",
    connection_options={
        "path": "s3://my-bucket/processed/events/",
        "partitionKeys": ["event_date"]
    },
    format="parquet",
    format_options={
        "compression": "snappy"
    },
    transformation_ctx="write_dyf"
)

# Update Data Catalog
glueContext.write_dynamic_frame.from_catalog(
    frame=enriched_dyf,
    database="analytics",
    table_name="processed_events",
    transformation_ctx="catalog_write"
)

job.commit()
```

### Glue Job Bookmarks

```
Job Bookmarks = Incremental Processing

Without Bookmarks:
┌─────────────────────────────────────────────────────────────────────────────┐
│  Job Run 1: Process files 1-100                                             │
│  Job Run 2: Process files 1-150 (re-processes 1-100)                       │
│  Job Run 3: Process files 1-200 (re-processes 1-150)                       │
│  → Wasteful! Processing same data repeatedly                                │
└─────────────────────────────────────────────────────────────────────────────┘

With Bookmarks:
┌─────────────────────────────────────────────────────────────────────────────┐
│  Job Run 1: Process files 1-100 (bookmark saved)                           │
│  Job Run 2: Process files 101-150 only (new data)                          │
│  Job Run 3: Process files 151-200 only (new data)                          │
│  → Efficient! Only new data processed                                       │
└─────────────────────────────────────────────────────────────────────────────┘

# Enable in job:
job.init(args['JOB_NAME'], args)
# ... transformations ...
job.commit()  # Saves bookmark

# CloudFormation
GlueJob:
  Type: AWS::Glue::Job
  Properties:
    Name: incremental-etl
    Role: !GetAtt GlueRole.Arn
    Command:
      Name: glueetl
      ScriptLocation: s3://scripts/etl.py
    DefaultArguments:
      '--job-bookmark-option': 'job-bookmark-enable'
      '--enable-continuous-cloudwatch-log': 'true'
    GlueVersion: '4.0'
    NumberOfWorkers: 10
    WorkerType: G.1X
```

### Glue Workflows

```yaml
# Orchestrate multiple crawlers and jobs
GlueWorkflow:
  Type: AWS::Glue::Workflow
  Properties:
    Name: daily-etl-workflow
    Description: Daily ETL pipeline

CrawlerTrigger:
  Type: AWS::Glue::Trigger
  Properties:
    Name: start-crawler
    Type: SCHEDULED
    Schedule: 'cron(0 6 * * ? *)'  # 6 AM daily
    WorkflowName: !Ref GlueWorkflow
    Actions:
      - CrawlerName: !Ref SourceCrawler

JobTrigger:
  Type: AWS::Glue::Trigger
  Properties:
    Name: start-etl-job
    Type: CONDITIONAL
    WorkflowName: !Ref GlueWorkflow
    Predicate:
      Conditions:
        - CrawlerName: !Ref SourceCrawler
          CrawlState: SUCCEEDED
          LogicalOperator: EQUALS
    Actions:
      - JobName: !Ref TransformJob
        Arguments:
          '--source_database': 'raw'
          '--source_table': 'events'

OutputCrawlerTrigger:
  Type: AWS::Glue::Trigger
  Properties:
    Name: update-output-catalog
    Type: CONDITIONAL
    WorkflowName: !Ref GlueWorkflow
    Predicate:
      Conditions:
        - JobName: !Ref TransformJob
          State: SUCCEEDED
    Actions:
      - CrawlerName: !Ref OutputCrawler
```

---

## Data Lake Architecture

### Lake Formation

```
AWS Lake Formation = Simplified data lake management

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Lake Formation Architecture                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Data Sources                    Lake Formation                            │
│   ┌─────────────┐               ┌─────────────────────────────────────┐    │
│   │ S3          │──────────────►│                                     │    │
│   │ RDS         │               │  ┌─────────────────────────────┐   │    │
│   │ On-premises │               │  │     Data Catalog            │   │    │
│   └─────────────┘               │  │  (Central Metadata Store)   │   │    │
│                                 │  └─────────────────────────────┘   │    │
│                                 │                                     │    │
│                                 │  ┌─────────────────────────────┐   │    │
│                                 │  │   Security & Access         │   │    │
│                                 │  │  ├── Column-level security  │   │    │
│                                 │  │  ├── Row-level security     │   │    │
│                                 │  │  └── Cell-level security    │   │    │
│                                 │  └─────────────────────────────┘   │    │
│                                 │                                     │    │
│                                 │  ┌─────────────────────────────┐   │    │
│                                 │  │      Governed Tables        │   │    │
│                                 │  │  (ACID transactions)        │   │    │
│                                 │  └─────────────────────────────┘   │    │
│                                 └─────────────────────────────────────┘    │
│                                               │                             │
│                                               ▼                             │
│   Consumers: Athena, Redshift Spectrum, EMR, Glue                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Permissions Model (Column/Row Level):

GRANT SELECT (column1, column2)
ON TABLE database.table
TO PRINCIPAL 'arn:aws:iam::account:role/analyst-role'
WITH GRANT OPTION;

-- Row-level security
GRANT SELECT
ON TABLE database.sales
TO PRINCIPAL 'arn:aws:iam::account:role/regional-manager'
WHERE region = 'us-west-2';
```

---

## Interview Discussion Points

### How would you design a real-time analytics pipeline?

```
Requirements-Based Design:

1. For IoT/Clickstream (High Volume, Low Latency)

   IoT Devices → Kinesis Data Streams → Kinesis Analytics (Flink)
                                              │
                        ┌─────────────────────┼─────────────────────┐
                        │                     │                     │
                        ▼                     ▼                     ▼
                 DynamoDB              S3 (archive)          CloudWatch
               (real-time lookup)    via Firehose          (alerts)

   Why this design:
   ├── Data Streams: Sub-second latency, ordering per partition key
   ├── Flink: Complex event processing, windowed aggregations
   ├── DynamoDB: Fast reads for real-time dashboards
   └── Firehose: Cost-effective archival to S3

2. For Log Analytics (Cost Optimized)

   Applications → CloudWatch Logs → Subscription Filter → Firehose
                                                              │
                                                              ▼
                                            S3 (Parquet, partitioned)
                                                              │
                                                              ▼
                                                           Athena
                                                              │
                                                              ▼
                                                          QuickSight

   Why this design:
   ├── Firehose: Handles batching, format conversion automatically
   ├── Parquet: 90% cost reduction in Athena queries
   ├── Partitions: Query only relevant time ranges
   └── Athena: Pay per query, no infrastructure to manage
```

### Kinesis vs Kafka (MSK)?

```
Decision Matrix:

┌────────────────────────────────────────────────────────────────────────────┐
│ Factor              │ Kinesis                │ MSK (Kafka)                 │
├────────────────────────────────────────────────────────────────────────────┤
│ Management          │ Fully managed          │ Managed brokers, you       │
│                     │                        │ manage topics/partitions   │
├────────────────────────────────────────────────────────────────────────────┤
│ Retention           │ 24 hrs - 365 days      │ Unlimited (storage-based)  │
├────────────────────────────────────────────────────────────────────────────┤
│ Throughput          │ Limited by shards      │ Higher per partition       │
│                     │ (1 MB/s write)         │ (configurable)             │
├────────────────────────────────────────────────────────────────────────────┤
│ Ecosystem           │ AWS-native             │ Rich Kafka ecosystem       │
│                     │ integrations           │ (Connect, Streams, KSQL)   │
├────────────────────────────────────────────────────────────────────────────┤
│ Pricing             │ Per shard-hour +       │ Per broker-hour            │
│                     │ data                   │ (higher base cost)         │
├────────────────────────────────────────────────────────────────────────────┤
│ Best for            │ AWS-native apps,       │ Kafka expertise exists,    │
│                     │ simpler use cases      │ need Kafka ecosystem       │
└────────────────────────────────────────────────────────────────────────────┘

Choose Kinesis when:
├── Building AWS-native applications
├── Team doesn't have Kafka expertise
├── Simpler streaming requirements
└── Want fully managed with no broker management

Choose MSK when:
├── Already using Kafka on-premises
├── Need Kafka Connect connectors
├── Higher throughput requirements
└── Need unlimited retention
```

### How do you optimize Athena query costs?

```
Cost Optimization Strategies:

1. Data Format (Biggest Impact)
   ├── Use Parquet or ORC
   ├── Enable compression (SNAPPY for Parquet)
   └── Typical savings: 80-90%

2. Partitioning
   ├── Partition by query patterns (date, region, etc.)
   ├── Use partition projection for dynamic partitions
   └── Avoid over-partitioning (small files)

3. Query Optimization
   ├── SELECT only needed columns (never SELECT *)
   ├── Use LIMIT for exploration
   ├── Filter on partition columns first
   └── Use approximate functions (approx_distinct vs COUNT DISTINCT)

4. Data Organization
   ├── Compact small files (Glue job, S3 Batch)
   ├── Target 128-256 MB file sizes
   └── Use bucketing for frequent JOIN columns

5. Caching with Athena Workgroups
   ├── Enable query result reuse
   ├── Set result retention period
   └── Multiple workgroups for cost allocation

Example Optimization:
Before: SELECT * FROM logs → 1 TB scanned → $5.00
After:  SELECT user_id, event_type
        FROM logs
        WHERE year='2024' AND month='01'
        → 10 GB scanned → $0.05 (100x savings)
```
