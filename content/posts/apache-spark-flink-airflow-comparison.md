+++
title = "Apache Spark vs Flink vs Airflow: A Comprehensive Comparison with Real-World Examples"
description = "An in-depth comparison of Apache Spark, Flink, and Airflow with practical real-world examples for data processing and orchestration."
date = 2026-01-20
updated = 2026-01-20

[taxonomies]
tags = ["big-data", "apache-spark", "apache-flink", "apache-airflow", "data-engineering", "streaming", "batch-processing"]

[extra]
toc = true
comment = false
+++

In the modern data engineering landscape, choosing the right tool for your data processing needs can be overwhelming. Apache Spark, Apache Flink, and Apache Airflow are three powerful frameworks that often come up in discussions, but they serve different purposes. In this comprehensive guide, we'll compare these three tools and provide real-world examples to help you understand when to use each one.

## Understanding the Core Difference

Before diving into the details, it's crucial to understand that these tools serve different primary purposes:

- **Apache Spark**: A unified analytics engine for large-scale data processing (both batch and streaming)
- **Apache Flink**: A stream processing framework with true real-time capabilities
- **Apache Airflow**: A workflow orchestration platform for scheduling and monitoring data pipelines

Think of it this way: Spark and Flink are the **workers** that process your data, while Airflow is the **manager** that schedules and coordinates when and how these workers operate.

## Apache Spark: The Unified Analytics Engine

### What is Apache Spark?

Apache Spark is a distributed computing framework designed for large-scale data processing. It provides high-level APIs in Java, Scala, Python, and R, and supports batch processing, SQL queries, streaming data, machine learning, and graph processing.

### Key Features

- **In-memory processing**: Processes data in RAM for faster computation
- **Lazy evaluation**: Optimizes execution plans before running
- **RDD, DataFrame, and Dataset APIs**: Multiple abstraction levels for different use cases
- **Spark Streaming**: Micro-batch processing for near real-time analytics
- **MLlib**: Built-in machine learning library
- **Spark SQL**: SQL interface for structured data processing

### Real-World Example: E-commerce User Behavior Analysis

Let's build a real-world example analyzing e-commerce user behavior to generate product recommendations.

**Scenario**: You have clickstream data from your e-commerce website stored in S3/HDFS, and you want to:
1. Clean and transform the data
2. Identify user purchase patterns
3. Generate product recommendations
4. Store results in a data warehouse

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, desc, collect_list, explode, lit, rank, sum, avg, when, countDistinct
from pyspark.sql.window import Window
from pyspark.ml.fpm import FPGrowth
from datetime import datetime

# Initialize Spark Session
spark = SparkSession.builder \
    .appName("EcommerceUserBehaviorAnalysis") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()

# Read clickstream data from S3
clickstream_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .parquet("s3a://your-bucket/clickstream-data/")

# Sample data structure:
# user_id, session_id, event_type, product_id, category, timestamp, price

# 1. Data Cleaning and Transformation
cleaned_df = clickstream_df \
    .filter(col("event_type").isin(["view", "add_to_cart", "purchase"])) \
    .filter(col("product_id").isNotNull()) \
    .withColumn("date", col("timestamp").cast("date"))

# 2. User Purchase Patterns Analysis
user_purchases = cleaned_df \
    .filter(col("event_type") == "purchase") \
    .groupBy("user_id", "session_id") \
    .agg(
        collect_list("product_id").alias("products"),
        count("product_id").alias("item_count"),
        sum("price").alias("total_amount")
    )

# 3. Find Popular Product Combinations using FP-Growth Algorithm
fp_growth = FPGrowth(itemsCol="products", minSupport=0.01, minConfidence=0.3)
model = fp_growth.fit(user_purchases)

# Get frequent itemsets (products bought together)
frequent_itemsets = model.freqItemsets
frequent_itemsets.orderBy(desc("freq")).show(20)

# Get association rules (if user buys A, they might buy B)
association_rules = model.associationRules
recommendations_df = association_rules \
    .withColumn("confidence_pct", (col("confidence") * 100).cast("int")) \
    .filter(col("confidence") > 0.5) \
    .orderBy(desc("confidence"))

# 4. Calculate Product Popularity by Category
product_popularity = cleaned_df \
    .filter(col("event_type") == "purchase") \
    .groupBy("category", "product_id") \
    .agg(
        count("*").alias("purchase_count"),
        avg("price").alias("avg_price")
    )

# Add popularity rank within each category
window_spec = Window.partitionBy("category").orderBy(desc("purchase_count"))
product_popularity_ranked = product_popularity \
    .withColumn("rank", rank().over(window_spec)) \
    .filter(col("rank") <= 10)

# 5. User Segmentation based on behavior
user_segments = cleaned_df \
    .groupBy("user_id") \
    .agg(
        count(when(col("event_type") == "view", 1)).alias("views"),
        count(when(col("event_type") == "add_to_cart", 1)).alias("cart_adds"),
        count(when(col("event_type") == "purchase", 1)).alias("purchases"),
        sum(when(col("event_type") == "purchase", col("price"))).alias("total_spent")
    ) \
    .withColumn("conversion_rate", 
                (col("purchases") / col("views") * 100).cast("decimal(5,2)")) \
    .withColumn("segment",
                when(col("total_spent") > 1000, "premium")
                .when(col("total_spent") > 500, "regular")
                .otherwise("casual"))

# 6. Save results to data warehouse (e.g., Redshift, Snowflake)
recommendations_df.write \
    .mode("overwrite") \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://your-redshift-cluster:5439/database") \
    .option("dbtable", "product_recommendations") \
    .option("user", "username") \
    .option("password", "password") \
    .save()

product_popularity_ranked.write \
    .mode("overwrite") \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://your-redshift-cluster:5439/database") \
    .option("dbtable", "popular_products") \
    .option("user", "username") \
    .option("password", "password") \
    .save()

user_segments.write \
    .mode("overwrite") \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://your-redshift-cluster:5439/database") \
    .option("dbtable", "user_segments") \
    .option("user", "username") \
    .option("password", "password") \
    .save()

# 7. Generate daily report
daily_stats = cleaned_df \
    .groupBy("date", "event_type") \
    .agg(
        count("*").alias("event_count"),
        countDistinct("user_id").alias("unique_users"),
        sum(when(col("event_type") == "purchase", col("price"))).alias("revenue")
    )

daily_stats.write \
    .mode("append") \
    .partitionBy("date") \
    .parquet("s3a://your-bucket/daily-stats/")

print("Analysis completed successfully!")
spark.stop()
```

**Why Spark for this use case?**
- Handles large volumes of historical clickstream data efficiently
- Built-in machine learning library (FPGrowth) for pattern discovery
- Can process data from multiple sources (S3, HDFS, databases)
- Optimized for batch processing with in-memory computation

## Apache Flink: True Stream Processing

### What is Apache Flink?

Apache Flink is a distributed stream processing framework that excels at processing unbounded data streams with low latency and high throughput. Unlike Spark's micro-batch approach, Flink provides true stream processing with event time semantics.

### Key Features

- **True streaming**: Processes events one at a time, not in micro-batches
- **Event time processing**: Handles out-of-order events correctly
- **Stateful computations**: Maintains state across events
- **Exactly-once semantics**: Guarantees no data loss or duplication
- **Low latency**: Millisecond-level latency for real-time applications
- **Complex Event Processing (CEP)**: Pattern detection in streams

### Real-World Example: Real-Time Fraud Detection System

Let's build a real-time fraud detection system for credit card transactions.

**Scenario**: Process credit card transactions in real-time to detect potential fraud patterns:
1. Track transaction velocity per user
2. Detect unusual spending patterns
3. Flag transactions from new locations
4. Identify suspicious transaction sequences
5. Alert in real-time when fraud is detected

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.api.common.state.*;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
import org.apache.flink.util.Collector;

import java.time.Duration;
import java.util.*;

// Transaction POJO
public class Transaction {
    public String transactionId;
    public String userId;
    public String cardNumber;
    public double amount;
    public String merchantId;
    public String merchantCategory;
    public String location;
    public double latitude;
    public double longitude;
    public long timestamp;
    
    // Constructor, getters, setters
}

// Fraud Alert POJO
public class FraudAlert {
    public String transactionId;
    public String userId;
    public String alertType;
    public String reason;
    public double riskScore;
    public long timestamp;
    
    // Constructor, getters, setters
}

public class RealTimeFraudDetection {
    
    public static void main(String[] args) throws Exception {
        
        // Set up the streaming execution environment
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Enable checkpointing for fault tolerance (every 10 seconds)
        env.enableCheckpointing(10000);
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        
        // Configure Kafka consumer
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "localhost:9092");
        kafkaProps.setProperty("group.id", "fraud-detection");
        
        // Create Kafka source for transactions
        FlinkKafkaConsumer<Transaction> kafkaConsumer = new FlinkKafkaConsumer<>(
            "transactions",
            new TransactionDeserializationSchema(),
            kafkaProps
        );
        
        // Assign watermarks for event time processing
        WatermarkStrategy<Transaction> watermarkStrategy = WatermarkStrategy
            .<Transaction>forBoundedOutOfOrderness(Duration.ofSeconds(5))
            .withTimestampAssigner((transaction, timestamp) -> transaction.timestamp);
        
        // Read transaction stream
        DataStream<Transaction> transactions = env
            .addSource(kafkaConsumer)
            .assignTimestampsAndWatermarks(watermarkStrategy);
        
        // Rule 1: Detect high-velocity transactions (>5 transactions in 1 minute)
        DataStream<FraudAlert> velocityAlerts = transactions
            .keyBy(t -> t.userId)
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))
            .process(new VelocityDetector());
        
        // Rule 2: Detect unusual amount patterns
        DataStream<FraudAlert> amountAlerts = transactions
            .keyBy(t -> t.userId)
            .process(new UnusualAmountDetector());
        
        // Rule 3: Detect rapid location changes
        DataStream<FraudAlert> locationAlerts = transactions
            .keyBy(t -> t.userId)
            .process(new LocationAnomalyDetector());
        
        // Rule 4: Detect suspicious transaction sequences
        DataStream<FraudAlert> sequenceAlerts = transactions
            .keyBy(t -> t.userId)
            .process(new SuspiciousSequenceDetector());
        
        // Combine all alerts
        DataStream<FraudAlert> allAlerts = velocityAlerts
            .union(amountAlerts)
            .union(locationAlerts)
            .union(sequenceAlerts);
        
        // Deduplicate and enrich alerts
        DataStream<FraudAlert> enrichedAlerts = allAlerts
            .keyBy(alert -> alert.transactionId)
            .window(TumblingEventTimeWindows.of(Time.seconds(5)))
            .process(new AlertAggregator());
        
        // Send high-risk alerts to Kafka
        FlinkKafkaProducer<FraudAlert> kafkaProducer = new FlinkKafkaProducer<>(
            "fraud-alerts",
            new FraudAlertSerializationSchema(),
            kafkaProps,
            FlinkKafkaProducer.Semantic.EXACTLY_ONCE
        );
        
        enrichedAlerts
            .filter(alert -> alert.riskScore > 0.7)
            .addSink(kafkaProducer);
        
        // Store all alerts in database for analysis
        enrichedAlerts.addSink(new JdbcSink());
        
        // Execute the Flink job
        env.execute("Real-Time Fraud Detection");
    }
    
    // Velocity Detector: Flags if user makes >5 transactions in 1 minute
    public static class VelocityDetector 
            extends ProcessWindowFunction<Transaction, FraudAlert, String, TimeWindow> {
        
        @Override
        public void process(String userId, Context context, 
                          Iterable<Transaction> transactions, 
                          Collector<FraudAlert> out) {
            
            int count = 0;
            Transaction lastTxn = null;
            
            for (Transaction txn : transactions) {
                count++;
                lastTxn = txn;
            }
            
            // Alert if more than 5 transactions in 1 minute
            if (count > 5) {
                FraudAlert alert = new FraudAlert();
                alert.transactionId = lastTxn.transactionId;
                alert.userId = userId;
                alert.alertType = "HIGH_VELOCITY";
                alert.reason = String.format("%d transactions in 1 minute", count);
                alert.riskScore = Math.min(0.5 + (count - 5) * 0.1, 1.0);
                alert.timestamp = System.currentTimeMillis();
                out.collect(alert);
            }
        }
    }
    
    // Unusual Amount Detector: Compares transaction with user's historical patterns
    public static class UnusualAmountDetector 
            extends KeyedProcessFunction<String, Transaction, FraudAlert> {
        
        // State to store user's transaction history
        private transient ValueState<TransactionStats> statsState;
        
        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<TransactionStats> descriptor = 
                new ValueStateDescriptor<>("transaction-stats", TransactionStats.class);
            statsState = getRuntimeContext().getState(descriptor);
        }
        
        @Override
        public void processElement(Transaction txn, Context ctx, 
                                  Collector<FraudAlert> out) throws Exception {
            
            TransactionStats stats = statsState.value();
            if (stats == null) {
                stats = new TransactionStats();
            }
            
            // Check if amount is significantly higher than average
            if (stats.count > 10) {
                double avgAmount = stats.totalAmount / stats.count;
                double stdDev = Math.sqrt(stats.sumOfSquares / stats.count - avgAmount * avgAmount);
                
                // Flag if amount is > 3 standard deviations from mean
                if (txn.amount > avgAmount + (3 * stdDev)) {
                    FraudAlert alert = new FraudAlert();
                    alert.transactionId = txn.transactionId;
                    alert.userId = txn.userId;
                    alert.alertType = "UNUSUAL_AMOUNT";
                    alert.reason = String.format("Amount $%.2f is %.1fx higher than average $%.2f",
                        txn.amount, txn.amount / avgAmount, avgAmount);
                    alert.riskScore = Math.min((txn.amount / avgAmount - 1) * 0.2, 0.9);
                    alert.timestamp = System.currentTimeMillis();
                    out.collect(alert);
                }
            }
            
            // Update statistics
            stats.count++;
            stats.totalAmount += txn.amount;
            stats.sumOfSquares += txn.amount * txn.amount;
            statsState.update(stats);
        }
    }
    
    // Location Anomaly Detector: Flags transactions from impossible locations
    public static class LocationAnomalyDetector 
            extends KeyedProcessFunction<String, Transaction, FraudAlert> {
        
        private transient ValueState<LastLocation> lastLocationState;
        
        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<LastLocation> descriptor = 
                new ValueStateDescriptor<>("last-location", LastLocation.class);
            lastLocationState = getRuntimeContext().getState(descriptor);
        }
        
        @Override
        public void processElement(Transaction txn, Context ctx, 
                                  Collector<FraudAlert> out) throws Exception {
            
            LastLocation lastLoc = lastLocationState.value();
            
            if (lastLoc != null) {
                // Calculate distance between locations (using Haversine formula)
                double distance = calculateDistance(
                    lastLoc.latitude, lastLoc.longitude,
                    txn.latitude, txn.longitude
                );
                
                // Calculate time difference in hours
                double timeDiffHours = (txn.timestamp - lastLoc.timestamp) / (1000.0 * 3600);
                
                // Calculate required speed (km/h)
                double requiredSpeed = distance / timeDiffHours;
                
                // Flag if speed > 1000 km/h (impossible without flying)
                if (timeDiffHours < 2 && requiredSpeed > 1000) {
                    FraudAlert alert = new FraudAlert();
                    alert.transactionId = txn.transactionId;
                    alert.userId = txn.userId;
                    alert.alertType = "IMPOSSIBLE_TRAVEL";
                    alert.reason = String.format("%.0f km traveled in %.1f hours (%.0f km/h)",
                        distance, timeDiffHours, requiredSpeed);
                    alert.riskScore = 0.95;
                    alert.timestamp = System.currentTimeMillis();
                    out.collect(alert);
                }
            }
            
            // Update last location
            LastLocation newLoc = new LastLocation();
            newLoc.latitude = txn.latitude;
            newLoc.longitude = txn.longitude;
            newLoc.timestamp = txn.timestamp;
            lastLocationState.update(newLoc);
        }
        
        private double calculateDistance(double lat1, double lon1, double lat2, double lon2) {
            // Haversine formula implementation
            double R = 6371; // Earth radius in km
            double dLat = Math.toRadians(lat2 - lat1);
            double dLon = Math.toRadians(lon2 - lon1);
            double a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                      Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2)) *
                      Math.sin(dLon/2) * Math.sin(dLon/2);
            double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
            return R * c;
        }
    }
    
    // Suspicious Sequence Detector: Detects patterns like test transactions
    public static class SuspiciousSequenceDetector 
            extends KeyedProcessFunction<String, Transaction, FraudAlert> {
        
        private transient ListState<Transaction> recentTransactionsState;
        
        @Override
        public void open(Configuration parameters) {
            ListStateDescriptor<Transaction> descriptor = 
                new ListStateDescriptor<>("recent-transactions", Transaction.class);
            recentTransactionsState = getRuntimeContext().getListState(descriptor);
        }
        
        @Override
        public void processElement(Transaction txn, Context ctx, 
                                  Collector<FraudAlert> out) throws Exception {
            
            List<Transaction> recentTxns = new ArrayList<>();
            recentTransactionsState.get().forEach(recentTxns::add);
            
            // Pattern 1: Multiple small transactions followed by large transaction
            if (recentTxns.size() >= 3) {
                int smallTxnCount = 0;
                for (Transaction t : recentTxns) {
                    if (t.amount < 5.0) smallTxnCount++;
                }
                
                if (smallTxnCount >= 3 && txn.amount > 500) {
                    FraudAlert alert = new FraudAlert();
                    alert.transactionId = txn.transactionId;
                    alert.userId = txn.userId;
                    alert.alertType = "SUSPICIOUS_SEQUENCE";
                    alert.reason = "Multiple small test transactions followed by large amount";
                    alert.riskScore = 0.85;
                    alert.timestamp = System.currentTimeMillis();
                    out.collect(alert);
                }
            }
            
            // Pattern 2: Transactions at suspicious merchant categories
            String[] suspiciousMerchants = {"WIRE_TRANSFER", "CRYPTO", "GIFT_CARDS"};
            if (Arrays.asList(suspiciousMerchants).contains(txn.merchantCategory)) {
                FraudAlert alert = new FraudAlert();
                alert.transactionId = txn.transactionId;
                alert.userId = txn.userId;
                alert.alertType = "SUSPICIOUS_MERCHANT";
                alert.reason = "Transaction at high-risk merchant category: " + txn.merchantCategory;
                alert.riskScore = 0.6;
                alert.timestamp = System.currentTimeMillis();
                out.collect(alert);
            }
            
            // Keep only last 5 transactions
            recentTxns.add(txn);
            if (recentTxns.size() > 5) {
                recentTxns.remove(0);
            }
            
            recentTransactionsState.update(recentTxns);
        }
    }
    
    // Helper classes
    public static class TransactionStats {
        public int count = 0;
        public double totalAmount = 0.0;
        public double sumOfSquares = 0.0;
    }
    
    public static class LastLocation {
        public double latitude;
        public double longitude;
        public long timestamp;
    }
}
```

**Why Flink for this use case?**
- **True real-time processing**: Detects fraud within milliseconds
- **Stateful computations**: Maintains user transaction history in memory
- **Event time processing**: Handles late-arriving transactions correctly
- **Exactly-once guarantees**: Ensures no alert is missed or duplicated
- **Complex event processing**: Detects sophisticated fraud patterns across multiple transactions

## Apache Airflow: The Workflow Orchestrator

### What is Apache Airflow?

Apache Airflow is a platform to programmatically author, schedule, and monitor workflows. It's not a data processing engine itself but rather orchestrates data pipelines that may use Spark, Flink, or other tools.

### Key Features

- **DAG-based workflows**: Define workflows as Directed Acyclic Graphs
- **Extensible**: Rich library of operators for various systems
- **Dynamic pipelines**: Generate pipelines dynamically using Python
- **Rich UI**: Web interface to monitor and manage workflows
- **Scheduling**: Cron-based scheduling with dependency management
- **Task retries and alerts**: Automatic retry and notification on failures

### Real-World Example: End-to-End ML Pipeline Orchestration

Let's build a complete ML pipeline that orchestrates data extraction, transformation, model training, and deployment.

**Scenario**: Build a daily ML pipeline that:
1. Extracts data from multiple sources
2. Runs data quality checks
3. Processes data with Spark
4. Trains a machine learning model
5. Validates model performance
6. Deploys model if metrics are acceptable
7. Sends notifications on success/failure

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.dummy import DummyOperator
from airflow.providers.amazon.aws.operators.s3 import S3CreateBucketOperator
from airflow.providers.amazon.aws.operators.emr import EmrCreateJobFlowOperator, EmrTerminateJobFlowOperator
from airflow.providers.amazon.aws.sensors.emr import EmrJobFlowSensor
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
from airflow.models import Variable
from airflow.utils.trigger_rule import TriggerRule
import json
import boto3
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score
import joblib

# Default arguments for the DAG
default_args = {
    'owner': 'data-science-team',
    'depends_on_past': False,
    'start_date': datetime(2026, 1, 1),
    'email': ['datascience@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=4),
}

# Create the DAG
dag = DAG(
    'ml_pipeline_customer_churn',
    default_args=default_args,
    description='End-to-end ML pipeline for customer churn prediction',
    schedule_interval='0 2 * * *',  # Run daily at 2 AM
    catchup=False,
    max_active_runs=1,
    tags=['ml', 'production', 'churn'],
)

# Task 1: Check data availability in source systems
def check_data_availability(**context):
    """Check if all required data sources have data for execution date"""
    execution_date = context['execution_date']
    date_str = execution_date.strftime('%Y-%m-%d')
    
    s3 = boto3.client('s3')
    bucket = 'company-data-lake'
    
    # Check multiple data sources
    sources = [
        f'raw/customer_data/{date_str}/',
        f'raw/transactions/{date_str}/',
        f'raw/support_tickets/{date_str}/',
        f'raw/product_usage/{date_str}/'
    ]
    
    missing_sources = []
    for source in sources:
        try:
            response = s3.list_objects_v2(Bucket=bucket, Prefix=source)
            if 'Contents' not in response or len(response['Contents']) == 0:
                missing_sources.append(source)
        except Exception as e:
            missing_sources.append(source)
    
    if missing_sources:
        raise ValueError(f"Missing data sources: {missing_sources}")
    
    print(f"All data sources available for {date_str}")
    return True

check_data_task = PythonOperator(
    task_id='check_data_availability',
    python_callable=check_data_availability,
    provide_context=True,
    dag=dag,
)

# Task 2: Run data quality checks
def run_data_quality_checks(**context):
    """Run data quality checks on raw data"""
    execution_date = context['execution_date']
    date_str = execution_date.strftime('%Y-%m-%d')
    
    # Read sample of data from S3
    s3 = boto3.client('s3')
    
    quality_issues = []
    
    # Check 1: Null value percentage
    # (In production, you'd read actual data)
    null_percentage = 0.05  # 5% nulls
    if null_percentage > 0.10:
        quality_issues.append(f"High null percentage: {null_percentage*100}%")
    
    # Check 2: Data freshness
    # Ensure data is not older than expected
    
    # Check 3: Schema validation
    # Validate that all expected columns exist
    
    # Check 4: Value ranges
    # Check that numeric values are within expected ranges
    
    if quality_issues:
        raise ValueError(f"Data quality issues found: {quality_issues}")
    
    print(f"Data quality checks passed for {date_str}")
    return True

data_quality_task = PythonOperator(
    task_id='data_quality_checks',
    python_callable=run_data_quality_checks,
    provide_context=True,
    dag=dag,
)

# Task 3: Submit Spark job to EMR for data processing
EMR_CONFIG = {
    'Name': 'ChurnDataProcessing',
    'ReleaseLabel': 'emr-6.10.0',
    'Applications': [{'Name': 'Spark'}, {'Name': 'Hadoop'}],
    'Instances': {
        'InstanceGroups': [
            {
                'Name': 'Master',
                'Market': 'ON_DEMAND',
                'InstanceRole': 'MASTER',
                'InstanceType': 'r5.xlarge',
                'InstanceCount': 1,
            },
            {
                'Name': 'Worker',
                'Market': 'SPOT',
                'InstanceRole': 'CORE',
                'InstanceType': 'r5.2xlarge',
                'InstanceCount': 3,
            }
        ],
        'KeepJobFlowAliveWhenNoSteps': True,
        'TerminationProtected': False,
    },
    'JobFlowRole': 'EMR_EC2_DefaultRole',
    'ServiceRole': 'EMR_DefaultRole',
    'LogUri': 's3://company-emr-logs/',
    'Steps': [
        {
            'Name': 'ProcessChurnData',
            'ActionOnFailure': 'CONTINUE',
            'HadoopJarStep': {
                'Jar': 'command-runner.jar',
                'Args': [
                    'spark-submit',
                    '--deploy-mode', 'cluster',
                    '--master', 'yarn',
                    '--conf', 'spark.sql.adaptive.enabled=true',
                    's3://company-spark-jobs/churn_data_processing.py',
                    '--input-date', '{{ ds }}',
                    '--output-path', 's3://company-processed-data/churn_features/{{ ds }}/'
                ]
            }
        }
    ],
}

create_emr_cluster = EmrCreateJobFlowOperator(
    task_id='create_emr_cluster',
    job_flow_overrides=EMR_CONFIG,
    aws_conn_id='aws_default',
    dag=dag,
)

wait_for_spark_job = EmrJobFlowSensor(
    task_id='wait_for_spark_job',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    aws_conn_id='aws_default',
    dag=dag,
)

terminate_emr_cluster = EmrTerminateJobFlowOperator(
    task_id='terminate_emr_cluster',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    aws_conn_id='aws_default',
    trigger_rule=TriggerRule.ALL_DONE,
    dag=dag,
)

# Task 4: Train ML model
def train_churn_model(**context):
    """Train customer churn prediction model"""
    execution_date = context['execution_date']
    date_str = execution_date.strftime('%Y-%m-%d')
    
    # Load processed features from S3
    s3 = boto3.client('s3')
    feature_path = f's3://company-processed-data/churn_features/{date_str}/features.parquet'
    
    # In production, you'd read actual data
    # df = pd.read_parquet(feature_path)
    
    # Simulated training for demonstration
    print("Loading training data...")
    # X_train, y_train = df.drop('churned', axis=1), df['churned']
    
    print("Training Random Forest model...")
    model = RandomForestClassifier(
        n_estimators=100,
        max_depth=10,
        min_samples_split=5,
        random_state=42,
        n_jobs=-1
    )
    
    # model.fit(X_train, y_train)
    
    # Save model to S3
    model_path = f'/tmp/churn_model_{date_str}.pkl'
    joblib.dump(model, model_path)
    
    s3.upload_file(
        model_path,
        'company-ml-models',
        f'churn_prediction/{date_str}/model.pkl'
    )
    
    print(f"Model trained and saved to S3")
    
    # Store model path in XCom for next task
    context['task_instance'].xcom_push(key='model_path', 
                                       value=f'churn_prediction/{date_str}/model.pkl')
    
    return True

train_model_task = PythonOperator(
    task_id='train_model',
    python_callable=train_churn_model,
    provide_context=True,
    dag=dag,
)

# Task 5: Validate model performance
def validate_model(**context):
    """Validate model performance on test set"""
    execution_date = context['execution_date']
    date_str = execution_date.strftime('%Y-%m-%d')
    
    # Load model from S3
    s3 = boto3.client('s3')
    model_path = context['task_instance'].xcom_pull(
        task_ids='train_model', 
        key='model_path'
    )
    
    # Download and load model
    s3.download_file(
        'company-ml-models',
        model_path,
        '/tmp/model.pkl'
    )
    model = joblib.load('/tmp/model.pkl')
    
    # Load test data
    # In production, load actual test data
    # X_test, y_test = load_test_data()
    
    # Make predictions (simulated)
    # y_pred = model.predict(X_test)
    
    # Calculate metrics (simulated values)
    metrics = {
        'accuracy': 0.87,
        'precision': 0.82,
        'recall': 0.79,
        'f1_score': 0.80,
        'date': date_str
    }
    
    print(f"Model Metrics: {metrics}")
    
    # Store metrics in database
    # In production, write to actual database
    
    # Store metrics in XCom
    context['task_instance'].xcom_push(key='metrics', value=metrics)
    
    # Return branch decision based on performance
    if metrics['f1_score'] >= 0.75:
        return 'deploy_model'
    else:
        return 'alert_poor_performance'

validate_model_task = BranchPythonOperator(
    task_id='validate_model',
    python_callable=validate_model,
    provide_context=True,
    dag=dag,
)

# Task 6a: Deploy model if performance is good
def deploy_model(**context):
    """Deploy model to production serving endpoint"""
    model_path = context['task_instance'].xcom_pull(
        task_ids='train_model',
        key='model_path'
    )
    
    # Deploy to SageMaker/Kubernetes/Model serving platform
    print(f"Deploying model from {model_path} to production...")
    
    # Update model registry
    # Update feature store
    # Update model serving endpoint
    
    print("Model successfully deployed to production")
    return True

deploy_model_task = PythonOperator(
    task_id='deploy_model',
    python_callable=deploy_model,
    provide_context=True,
    dag=dag,
)

# Task 6b: Alert if performance is poor
alert_poor_performance = SlackWebhookOperator(
    task_id='alert_poor_performance',
    http_conn_id='slack_webhook',
    message="""
    :warning: *Model Performance Alert*
    
    The churn prediction model for {{ ds }} did not meet performance thresholds.
    
    *Metrics:*
    {{ task_instance.xcom_pull(task_ids='validate_model', key='metrics') }}
    
    Please review the model and data quality.
    """,
    dag=dag,
)

# Task 7: Update model registry
update_registry = PostgresOperator(
    task_id='update_model_registry',
    postgres_conn_id='postgres_ml_metadata',
    sql="""
    INSERT INTO model_registry (
        model_name, 
        version, 
        metrics, 
        model_path, 
        created_at,
        status
    ) VALUES (
        'churn_prediction',
        '{{ ds }}',
        '{{ task_instance.xcom_pull(task_ids="validate_model", key="metrics") | tojson }}',
        '{{ task_instance.xcom_pull(task_ids="train_model", key="model_path") }}',
        '{{ ts }}',
        'deployed'
    );
    """,
    trigger_rule=TriggerRule.ONE_SUCCESS,
    dag=dag,
)

# Task 8: Send success notification
send_success_notification = SlackWebhookOperator(
    task_id='send_success_notification',
    http_conn_id='slack_webhook',
    message="""
    :white_check_mark: *ML Pipeline Success*
    
    The churn prediction pipeline for {{ ds }} completed successfully!
    
    *Metrics:*
    {{ task_instance.xcom_pull(task_ids='validate_model', key='metrics') }}
    
    Model deployed and ready for inference.
    """,
    trigger_rule=TriggerRule.ONE_SUCCESS,
    dag=dag,
)

# Task 9: Cleanup temporary files
cleanup = BashOperator(
    task_id='cleanup',
    bash_command='rm -rf /tmp/churn_* /tmp/model.pkl',
    trigger_rule=TriggerRule.ALL_DONE,
    dag=dag,
)

# Task 10: Generate daily report
generate_report = PythonOperator(
    task_id='generate_daily_report',
    python_callable=lambda **context: print("Generating daily report..."),
    provide_context=True,
    trigger_rule=TriggerRule.ALL_DONE,
    dag=dag,
)

# Define task dependencies
check_data_task >> data_quality_task >> create_emr_cluster
create_emr_cluster >> wait_for_spark_job >> terminate_emr_cluster
terminate_emr_cluster >> train_model_task >> validate_model_task

# Branching based on model performance
validate_model_task >> [deploy_model_task, alert_poor_performance]

# Both branches converge to update registry
[deploy_model_task, alert_poor_performance] >> update_registry
update_registry >> send_success_notification >> cleanup >> generate_report
```

**Why Airflow for this use case?**
- **Orchestrates complex workflows**: Manages dependencies between multiple tasks
- **Handles failures gracefully**: Automatic retries and alerting
- **Dynamic pipelines**: Can generate tasks based on runtime conditions
- **Monitors progress**: Rich UI to track pipeline execution
- **Integrations**: Connects with Spark, databases, cloud services, etc.
- **Scheduling**: Runs pipelines on schedule with dependency management

## Comparison Table

| Feature | Apache Spark | Apache Flink | Apache Airflow |
|---------|-------------|--------------|----------------|
| **Primary Purpose** | Batch & Streaming Processing | Stream Processing | Workflow Orchestration |
| **Processing Model** | Micro-batch | True streaming | N/A (orchestrator) |
| **Latency** | Seconds | Milliseconds | N/A |
| **State Management** | Limited | Excellent | N/A |
| **Fault Tolerance** | RDD lineage, checkpointing | Checkpointing, savepoints | Task retries |
| **APIs** | Java, Scala, Python, R, SQL | Java, Scala, Python, SQL | Python |
| **Learning Curve** | Moderate | Steep | Easy to Moderate |
| **Maturity** | Very mature | Mature | Very mature |
| **Community** | Very large | Large | Very large |
| **Use Cases** | Batch analytics, ML, ETL | Real-time analytics, CEP | Pipeline orchestration |
| **Memory Usage** | High (in-memory) | Moderate | Low |
| **Exactly-Once** | Structured Streaming | Native support | Via operators |
| **Event Time** | Watermarks | Native support | N/A |
| **SQL Support** | Excellent (Spark SQL) | Good (Flink SQL) | Via operators |
| **ML Libraries** | MLlib (built-in) | FlinkML (limited) | Integrates external |
| **Deployment** | Standalone, YARN, K8s, Mesos | Standalone, YARN, K8s, Mesos | Standalone, K8s, Docker |
| **Monitoring** | Spark UI, metrics | Flink UI, metrics | Rich web UI |

## When to Use Which Tool?

### Use Apache Spark When:

1. **Batch processing of large datasets**: Historical data analysis, ETL jobs
2. **Complex analytics**: Machine learning, graph processing, SQL queries
3. **Near real-time is acceptable**: Micro-batch processing with seconds latency
4. **You need mature ML libraries**: MLlib provides comprehensive algorithms
5. **Flexibility in languages**: Need support for multiple programming languages
6. **Cost optimization**: Can use spot instances for batch jobs

**Examples:**
- Daily report generation
- Training machine learning models
- Data warehouse ETL
- Log analysis and aggregation
- Feature engineering for ML

### Use Apache Flink When:

1. **True real-time processing**: Millisecond-level latency required
2. **Complex event processing**: Detecting patterns across multiple events
3. **Stateful computations**: Need to maintain state across events
4. **Event time processing**: Handling out-of-order events correctly
5. **Exactly-once guarantees**: Critical for financial or compliance applications
6. **Continuous queries**: Long-running streaming queries

**Examples:**
- Fraud detection
- Real-time recommendations
- Network monitoring and anomaly detection
- Real-time dashboards
- IoT event processing
- Stock trading systems

### Use Apache Airflow When:

1. **Orchestrating complex workflows**: Multiple dependent tasks
2. **Scheduling data pipelines**: Run pipelines on a schedule
3. **Managing dependencies**: Tasks that depend on other tasks or external systems
4. **Monitoring and alerting**: Need visibility into pipeline execution
5. **Mixing multiple tools**: Coordinating Spark, Flink, databases, APIs, etc.
6. **Dynamic pipelines**: Generate tasks based on runtime conditions

**Examples:**
- ETL pipelines
- ML model training pipelines
- Data quality monitoring
- Report generation workflows
- Multi-stage data processing
- Backup and maintenance tasks

## Can You Use Them Together?

**Absolutely!** In fact, many production data platforms use all three together:

**Example Architecture:**
1. **Airflow** orchestrates the entire data platform
2. **Flink** processes real-time streaming data
3. **Spark** runs batch jobs for historical analysis and ML training
4. **Airflow** schedules when Spark jobs run
5. **Airflow** monitors Flink streaming jobs
6. **Airflow** triggers alerts if either fails

**Real-world scenario:**
```
Daily at 2 AM (Airflow scheduled):
  → Extract data from APIs (Airflow task)
  → Run data quality checks (Airflow task)
  → Process with Spark (Airflow triggers Spark job)
  → Train ML model (Airflow triggers Spark ML job)
  → Deploy model (Airflow task)
  
Continuous (Flink streaming):
  → Read events from Kafka
  → Apply ML model for real-time predictions
  → Write results to database
  → Send alerts on anomalies
  
Hourly (Airflow scheduled):
  → Check Flink job health (Airflow sensor)
  → Restart Flink job if failed (Airflow task)
  → Update monitoring dashboard (Airflow task)
```

## Conclusion

Choosing between Apache Spark, Flink, and Airflow depends on your specific use case:

- **Spark** is your go-to for batch processing and complex analytics with a rich ecosystem
- **Flink** excels at true real-time stream processing with low latency and stateful computations
- **Airflow** is essential for orchestrating and managing complex data workflows

In modern data platforms, you'll often use all three:
- **Airflow** as the orchestrator managing the entire data platform
- **Flink** handling real-time event streams
- **Spark** processing large-scale batch jobs and machine learning workloads

The key is understanding that these tools complement each other rather than compete. By using them together strategically, you can build robust, scalable data platforms that handle both real-time and batch processing needs while maintaining clear visibility and control over your workflows.

## Further Reading

- [Apache Spark Documentation](https://spark.apache.org/docs/latest/)
- [Apache Flink Documentation](https://flink.apache.org/docs/stable/)
- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Stream Processing with Apache Flink (O'Reilly)](https://www.oreilly.com/library/view/stream-processing-with/9781491974285/)
- [Learning Spark, 2nd Edition (O'Reilly)](https://www.oreilly.com/library/view/learning-spark-2nd/9781492050032/)
- [Data Pipelines with Apache Airflow (Manning)](https://www.manning.com/books/data-pipelines-with-apache-airflow)
