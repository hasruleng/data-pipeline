Real-Time Sales Analytics Platform
A complete real-time analytics platform combining ClickHouse, Kafka, and Streamlit for processing and visualizing sales data streams.
🏗️ Architecture Overview
This platform provides:

ClickHouse Database with Kafka integration for real-time data ingestion
Streamlit Dashboard for live analytics visualization
Apache Superset for advanced business intelligence
Docker Compose orchestration for easy deployment

📋 Prerequisites

Docker and Docker Compose
Python 3.8+
Access to Kafka broker (configured for stack-kafka-kafka-1:9092)

🚀 Quick Start
1. Clone and Setup
bashgit clone <your-repo-url>
cd <your-repo-name>
2. Environment Configuration
Create a .env file in the docker/ directory:
bash# Create docker directory if it doesn't exist
mkdir -p docker

# Copy environment template (you'll need to create this)
cp docker/.env-example docker/.env
3. Launch Services
bash# Start all services
docker-compose up -d

# Check service status
docker-compose ps
4. Access Applications

Superset: http://localhost:8088
Streamlit Dashboard: Run separately (see below)
Nginx Proxy: http://localhost:80

🗄️ ClickHouse Database Setup
Database Schema
The platform uses three main ClickHouse tables:
1. Base Sales Data Table
sqlCREATE TABLE default.sales_data
(
    `date` Datetime,
    `name` String,
    `market_area` String,
    `number_of_sales` UInt32,
    `pricing_unit` Float64,
    `insert_time_clickhouse` SimpleAggregateFunction(max, DateTime('Asia/Jakarta')) DEFAULT now()
)
ENGINE = ReplacingMergeTree
PARTITION BY name
PRIMARY KEY (date)
ORDER BY (date, name, market_area, number_of_sales, pricing_unit);
2. Kafka Queue Table
sqlCREATE TABLE default.sales_data_queue
(
    `date` UInt64,
    `name` String,
    `market_area` String,
    `number_of_sales` Int32,
    `pricing_unit` Float64
)
ENGINE = Kafka
SETTINGS 
    kafka_broker_list = 'stack-kafka-kafka-1:9092',
    kafka_topic_list = 'etl.public.sales_data',
    kafka_group_name = 'test-1',
    kafka_format = 'JSONEachRow';
3. Materialized View
sqlCREATE MATERIALIZED VIEW default.sales_data_queue_mv TO default.sales_data
AS
SELECT
    toDateTime(`date` / 1000000) as date,
    name,
    market_area,
    number_of_sales,
    pricing_unit
FROM default.sales_data_queue;
4. Analytics View
sqlCREATE VIEW sales_data_view
AS
SELECT
    `date`,
    name,
    market_area,
    toInt32(number_of_sales) as number_of_sales,
    pricing_unit,
    number_of_sales * pricing_unit as revenue,
    toDateTime64(insert_time_clickhouse, 3, 'Asia/Jakarta') as insert_time_clickhouse,
    toDateTime64(now(), 3 , 'Asia/Jakarta') as source_collected
FROM `default`.sales_data;
Setting up ClickHouse

Connect to ClickHouse container:

bashdocker-compose exec superset_db bash
# Or connect directly to ClickHouse if you have it running separately

Execute the SQL scripts:
Copy the SQL commands from the notebook and execute them in your ClickHouse client.

📊 Streamlit Dashboard
Installation
bash# Install required packages
pip install streamlit clickhouse-sqlalchemy pandas
Running the Dashboard

Save the dashboard code as app.py:

pythonimport streamlit as st
import pandas as pd
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from datetime import datetime
import time

# Database connection
try:
    conn_str = 'clickhouse://default:password@localhost:18123/default'
    engine = create_engine(conn_str)
    session = sessionmaker(bind=engine)()
except Exception as e:
    print("Error while connecting to ClickHouse: " + str(e))

# Streamlit configuration
st.set_page_config(layout="wide")
st.title("Sales Data Analytics Real-Time ▊ ▋ ▌ ▍ ▎ ▏ █")

# Dashboard implementation continues...

Launch the dashboard:

bashstreamlit run app.py

Access the dashboard:
Open http://localhost:8501 in your browser.

Dashboard Features

Real-time data refresh (configurable intervals)
Time period selection (5 minutes to 1 hour)
Live metrics (Total deals, Revenue)
Interactive charts (Line charts, data tables)
Market area analysis (Deals by region)
Member performance (Deals by sales person)

🐳 Docker Services
Service Overview
ServicePortDescriptionnginx80Reverse proxy and load balancersuperset8088Apache Superset BI platformsuperset-websocket8080WebSocket support for Supersetsuperset-node9000Frontend development serverdb (PostgreSQL)5432Superset metadata databaseredis6379Caching and session storage
Useful Docker Commands
bash# View logs
docker-compose logs -f superset

# Restart specific service
docker-compose restart superset

# Scale services
docker-compose up -d --scale superset-worker=3

# Stop all services
docker-compose down

# Remove volumes (caution: data loss)
docker-compose down -v
🔧 Configuration
Kafka Configuration
Update the Kafka settings in your ClickHouse queue table:
sql-- Modify these settings based on your Kafka setup
kafka_broker_list = 'your-kafka-broker:9092'
kafka_topic_list = 'your-topic-name'
kafka_group_name = 'your-consumer-group'
ClickHouse Connection
Update connection string in app.py:
python# Modify based on your ClickHouse setup
conn_str = 'clickhouse://username:password@host:port/database'
Environment Variables
Key environment variables to configure in docker/.env:
bash# Superset Configuration
SUPERSET_SECRET_KEY=your-secret-key
SUPERSET_LOAD_EXAMPLES=yes

# Database Configuration  
DATABASE_DB=superset
DATABASE_HOST=db
DATABASE_PASSWORD=superset
DATABASE_USER=superset

# Redis Configuration
REDIS_HOST=redis
REDIS_PORT=6379
📈 Data Flow

Data Ingestion: Kafka streams sales data to ClickHouse
Real-time Processing: Materialized views process incoming data
Analytics Layer: Views aggregate data for dashboard consumption
Visualization: Streamlit dashboard displays real-time metrics

🛠️ Development
Adding New Metrics

Extend the ClickHouse view:

sql-- Add new calculated fields to sales_data_view
ALTER VIEW sales_data_view AS
SELECT 
    *,
    -- Add your new metrics here
    revenue / number_of_sales as avg_deal_size
FROM `default`.sales_data;

Update Streamlit dashboard:

python# Add new visualization components
st.metric(
    label="Average Deal Size",
    value=df['avg_deal_size'].mean()
)
Testing
bash# Test ClickHouse connection
docker-compose exec superset clickhouse-client --query "SELECT version()"

# Test Streamlit app
streamlit run app.py --server.headless true
🚨 Troubleshooting
Common Issues

ClickHouse Connection Failed

Check if ClickHouse is running and accessible
Verify connection string credentials
Ensure network connectivity between services


Kafka Integration Issues

Verify Kafka broker accessibility
Check topic name and permissions
Review consumer group configuration


Streamlit Dashboard Not Loading

Check Python dependencies installation
Verify database connection string
Review browser console for JavaScript errors


Docker Services Not Starting

Check Docker daemon status
Verify port availability
Review service logs: docker-compose logs [service-name]



Performance Optimization
sql-- Optimize ClickHouse queries with proper indexing
ALTER TABLE sales_data ADD INDEX idx_date_name date, name TYPE minmax GRANULARITY 4;

-- Monitor query performance
SELECT query, query_duration_ms FROM system.query_log ORDER BY query_duration_ms DESC LIMIT 10;
📚 Additional Resources

ClickHouse Documentation
Streamlit Documentation
Apache Superset Documentation
Docker Compose Documentation

🤝 Contributing

Fork the repository
Create a feature branch (git checkout -b feature/amazing-feature)
Commit your changes (git commit -m 'Add amazing feature')
Push to the branch (git push origin feature/amazing-feature)
Open a Pull Request

📄 License
This project is licensed under the MIT License - see the LICENSE file for details.
📞 Support
For support and questions:

Create an issue in this repository
Check the troubleshooting section above
Review service logs using Docker Compose commands


Note: This platform is designed for development and testing. For production deployment, ensure proper security configurations, environment variables, and resource allocation.