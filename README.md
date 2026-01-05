# BIG DATA ANALYSIS
# Complete Implementation Guide: Real-Time E-commerce Analytics

## Prerequisites

### System Requirements
- **OS**: Ubuntu 20.04+ / Windows 10+ / macOS 10.15+
- **RAM**: Minimum 8GB (Recommended 16GB+)
- **Storage**: 50GB+ free space
- **CPU**: Multi-core processor (4+ cores recommended)
- **Network**: Stable internet connection

### Software Requirements
- Docker & Docker Compose
- Python 3.8+
- Java 8 or 11
- Git (optional)

---

## STEP 1: Environment Setup

### 1.1 Install Docker and Docker Compose

**Ubuntu/Linux:**
```bash
# Update package index
sudo apt update

# Install Docker
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group
sudo usermod -aG docker $USER
```

**Windows:**
- Download Docker Desktop from https://docker.com/products/docker-desktop
- Install and restart system
- Verify installation: `docker --version`

### 1.2 Install Python Dependencies
```bash
# Install Python and pip
sudo apt install python3 python3-pip

# Install required Python packages
pip3 install kafka-python elasticsearch pandas numpy faker datetime requests
```

---

## STEP 2: Infrastructure Setup

### 2.1 Create Project Directory Structure
```bash
mkdir ecommerce-analytics
cd ecommerce-analytics

# Create directory structure
mkdir -p {data-generator,kafka-config,spark-jobs,elasticsearch-config,kibana-config,docker}
```

### 2.2 Docker Compose Configuration

Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200

  spark-master:
    image: bitnami/spark:latest
    environment:
      - SPARK_MODE=master
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    ports:
      - "8080:8080"
      - "7077:7077"

  spark-worker:
    image: bitnami/spark:latest
    depends_on:
      - spark-master
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2g
      - SPARK_WORKER_CORES=2

volumes:
  elasticsearch_data:
```

### 2.3 Start Infrastructure
```bash
# Start all services
docker-compose up -d

# Verify services are running
docker-compose ps

# Check logs if needed
docker-compose logs kafka
docker-compose logs elasticsearch
```

---

## STEP 3: Data Generation

### 3.1 Create Data Generator

Create `data-generator/ecommerce_data_generator.py`:
```python
import json
import random
import time
from datetime import datetime, timedelta
from faker import Faker
from kafka import KafkaProducer
import uuid

fake = Faker()

class EcommerceDataGenerator:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda x: json.dumps(x).encode('utf-8')
        )
        
        # Product categories and sample products
        self.categories = {
            'Electronics': ['iPhone 14', 'Samsung TV', 'MacBook Pro', 'iPad', 'AirPods'],
            'Clothing': ['Nike Shoes', 'Adidas Jacket', 'Levis Jeans', 'H&M Shirt', 'Zara Dress'],
            'Books': ['Python Guide', 'Data Science Handbook', 'Web Development', 'AI Basics'],
            'Sports': ['Tennis Racket', 'Football', 'Running Shoes', 'Gym Equipment'],
            'Home': ['Kitchen Set', 'Bedroom Decor', 'Living Room Sofa', 'Dining Table']
        }
        
        self.event_types = ['page_view', 'product_click', 'add_to_cart', 'purchase', 'search']
        self.event_weights = [40, 25, 15, 5, 15]  # Weighted probabilities
        
    def generate_customer_event(self):
        customer_id = f"cust_{random.randint(1000, 9999)}"
        session_id = str(uuid.uuid4())
        
        category = random.choice(list(self.categories.keys()))
        product = random.choice(self.categories[category])
        
        event = {
            'event_id': str(uuid.uuid4()),
            'timestamp': datetime.now().isoformat(),
            'customer_id': customer_id,
            'session_id': session_id,
            'event_type': random.choices(self.event_types, weights=self.event_weights)[0],
            'product_id': f"prod_{hash(product) % 10000}",
            'product_name': product,
            'category': category,
            'price': round(random.uniform(10, 1000), 2),
            'quantity': random.randint(1, 5),
            'user_agent': fake.user_agent(),
            'ip_address': fake.ipv4(),
            'location': {
                'city': fake.city(),
                'country': fake.country(),
                'latitude': float(fake.latitude()),
                'longitude': float(fake.longitude())
            }
        }
        
        return event
    
    def start_streaming(self, events_per_second=10):
        print(f"Starting data generation: {events_per_second} events/second")
        
        while True:
            try:
                # Generate batch of events
                for _ in range(events_per_second):
                    event = self.generate_customer_event()
                    
                    # Send to appropriate Kafka topic based on event type
                    if event['event_type'] == 'purchase':
                        topic = 'ecommerce-transactions'
                    else:
                        topic = 'ecommerce-events'
                    
                    self.producer.send(topic, value=event)
                
                # Flush and wait
                self.producer.flush()
                time.sleep(1)
                
                print(f"Generated {events_per_second} events at {datetime.now()}")
                
            except Exception as e:
                print(f"Error generating data: {e}")
                time.sleep(5)

if __name__ == "__main__":
    generator = EcommerceDataGenerator()
    generator.start_streaming(events_per_second=50)
```

### 3.2 Create Kafka Topics
```bash
# Create topics
docker exec -it $(docker-compose ps -q kafka) kafka-topics --create --topic ecommerce-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

docker exec -it $(docker-compose ps -q kafka) kafka-topics --create --topic ecommerce-transactions --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

# Verify topics
docker exec -it $(docker-compose ps -q kafka) kafka-topics --list --bootstrap-server localhost:9092
```

---

## STEP 4: Spark Streaming Job

### 4.1 Install PySpark
```bash
pip3 install pyspark findspark
```

### 4.2 Create Spark Streaming Application

Create `spark-jobs/ecommerce_stream_processor.py`:
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import json

# Initialize Spark Session
spark = SparkSession.builder \
    .appName("EcommerceStreamProcessor") \
    .config("spark.jars.packages", 
            "org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.0,"
            "org.elasticsearch:elasticsearch-spark-30_2.12:8.8.0") \
    .getOrCreate()

spark.sparkContext.setLogLevel("WARN")

# Define schema for incoming events
event_schema = StructType([
    StructField("event_id", StringType(), True),
    StructField("timestamp", TimestampType(), True),
    StructField("customer_id", StringType(), True),
    StructField("session_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("product_id", StringType(), True),
    StructField("product_name", StringType(), True),
    StructField("category", StringType(), True),
    StructField("price", DoubleType(), True),
    StructField("quantity", IntegerType(), True),
    StructField("user_agent", StringType(), True),
    StructField("ip_address", StringType(), True),
    StructField("location", StructType([
        StructField("city", StringType(), True),
        StructField("country", StringType(), True),
        StructField("latitude", DoubleType(), True),
        StructField("longitude", DoubleType(), True)
    ]), True)
])

# Read from Kafka
kafka_stream = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "ecommerce-events,ecommerce-transactions") \
    .option("startingOffsets", "latest") \
    .load()

# Parse JSON data
parsed_stream = kafka_stream.select(
    col("topic"),
    from_json(col("value").cast("string"), event_schema).alias("data")
).select("topic", "data.*")

# Add processing timestamp and derive additional fields
enriched_stream = parsed_stream.withColumn("processing_time", current_timestamp()) \
    .withColumn("hour", hour("timestamp")) \
    .withColumn("day_of_week", dayofweek("timestamp"))

# Write to Elasticsearch
def write_to_elasticsearch(df, epoch_id):
    try:
        df.write \
            .format("org.elasticsearch.spark.sql") \
            .option("es.resource", "ecommerce-events/_doc") \
            .option("es.nodes", "localhost") \
            .option("es.port", "9200") \
            .mode("append") \
            .save()
        print(f"Batch {epoch_id} written to Elasticsearch successfully")
    except Exception as e:
        print(f"Error writing batch {epoch_id}: {e}")

# Start streaming query
query = enriched_stream.writeStream \
    .trigger(processingTime='10 seconds') \
    .foreachBatch(write_to_elasticsearch) \
    .outputMode("update") \
    .start()

print("Spark Streaming job started...")
query.awaitTermination()
```

---

## STEP 5: Running the Complete System

### 5.1 Start All Services
```bash
# 1. Start infrastructure
docker-compose up -d

# 2. Wait for services to be ready (2-3 minutes)
# Check Elasticsearch health
curl -X GET "localhost:9200/_cluster/health?wait_for_status=yellow&timeout=60s"

# 3. Verify Kibana is accessible
curl http://localhost:5601/status
```

### 5.2 Start Data Generation
```bash
# In terminal 1: Start data generator
cd data-generator
python3 ecommerce_data_generator.py
```

### 5.3 Start Spark Streaming
```bash
# In terminal 2: Start Spark streaming job
cd spark-jobs
python3 ecommerce_stream_processor.py
```

### 5.4 Verify Data Flow
```bash
# Check Kafka topics have data
docker exec -it $(docker-compose ps -q kafka) kafka-console-consumer --topic ecommerce-events --from-beginning --bootstrap-server localhost:9092 --max-messages 5

# Check Elasticsearch indices
curl -X GET "localhost:9200/_cat/indices?v"

# Check document count
curl -X GET "localhost:9200/ecommerce-events/_count"
```

---

## STEP 6: Kibana Dashboard Setup

### 6.1 Access Kibana
1. Open browser: `http://localhost:5601`
2. Navigate to "Stack Management" > "Index Patterns"
3. Create index pattern: `ecommerce-events*`
4. Set time field: `timestamp`

### 6.2 Create Visualizations

**Dashboard Components:**
1. **Real-time Events Timeline**: Line chart showing events over time
2. **Event Types Distribution**: Pie chart of event types
3. **Top Products**: Bar chart of most viewed products
4. **Geographic Distribution**: Map visualization of customer locations
5. **Category Performance**: Horizontal bar chart of categories
6. **Conversion Funnel**: Funnel visualization showing customer journey

### 6.3 Import Sample Dashboard
```bash
# Create dashboard configuration (save as kibana-dashboard.json)
# Import via Kibana UI: Stack Management > Saved Objects > Import
```

---

## STEP 7: Monitoring and Troubleshooting

### 7.1 System Health Checks
```bash
# Check all services
docker-compose ps

# Check logs
docker-compose logs -f kafka
docker-compose logs -f elasticsearch
docker-compose logs -f spark-master

# Monitor resource usage
docker stats
```

### 7.2 Common Issues and Solutions

**Issue: Kafka connection failed**
```bash
# Solution: Restart Kafka
docker-compose restart kafka
```

**Issue: Elasticsearch heap size error**
```bash
# Solution: Increase memory in docker-compose.yml
environment:
  - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
```

**Issue: Spark job fails**
```bash
# Solution: Check Java version and Spark compatibility
java -version
spark-submit --version
```

---

## STEP 8: Testing Different Scenarios

### 8.1 Volume Testing
```python
# Modify data generator for high volume
generator.start_streaming(events_per_second=1000)
```

### 8.2 Variety Testing
```python
# Add more event types and data formats
new_event_types = ['wishlist_add', 'review_submit', 'share_product']
```

### 8.3 Velocity Testing
```python
# Implement burst traffic simulation
def burst_traffic():
    for _ in range(10):
        generator.start_streaming(events_per_second=2000)
        time.sleep(10)
```

---

## STEP 9: Performance Optimization

### 9.1 Kafka Optimization
```bash
# Increase partitions for better parallelism
kafka-topics --alter --topic ecommerce-events --partitions 6 --bootstrap-server localhost:9092
```

### 9.2 Spark Optimization
```python
# Increase resources
spark = SparkSession.builder \
    .appName("EcommerceStreamProcessor") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()
```

### 9.3 Elasticsearch Optimization
```bash
# Optimize index settings
curl -X PUT "localhost:9200/ecommerce-events/_settings" -H 'Content-Type: application/json' -d'
{
  "refresh_interval": "30s",
  "number_of_replicas": 0
}
'
```

---

## STEP 10: Project Submission

### 10.1 Documentation
1. Complete the project report with your results
2. Include screenshots of Kibana dashboards
3. Document any issues faced and solutions

### 10.2 Code Organization
```
ecommerce-analytics/
├── docker-compose.yml
├── data-generator/
│   └── ecommerce_data_generator.py
├── spark-jobs/
│   └── ecommerce_stream_processor.py
├── kibana-config/
│   └── dashboard-config.json
├── screenshots/
│   ├── kafka-producer.png
│   ├── spark-streaming.png
│   └── kibana-dashboard.png
└── README.md
```

### 10.3 Presentation
- Prepare demo showing real-time data flow
- Explain architecture and technology choices
- Show different types of analytics and insights
- Demonstrate system performance under load

This comprehensive guide will help you successfully implement and run your BDA project. The system demonstrates all key Big Data concepts: Volume (high throughput), Velocity (real-time processing), and Variety (different data types and sources).
