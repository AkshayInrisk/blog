# Building a Data Pipeline on Google Cloud Using Flask and Parquet

In data-driven industries, the quality and reliability of your data pipeline often determine the accuracy of every downstream insight — whether it’s a business dashboard, an ML model, or a disaster risk product.

This article walks through how a cloud-based data pipeline can be built using Flask, Google Cloud Storage, and Apache Parquet, with deployment on Google Cloud Run. It’s designed for engineers who want to understand the architecture, design choices, and performance considerations of a modern ingestion and processing system.

To keep things concrete, the example centers around ingesting and processing rainfall data from the Indian Meteorological Department (IMD).

---

## Understanding the Core Components

Before diving into the architecture, it’s important to establish what each component actually does.

### Flask

Flask is a lightweight web framework written in Python. It’s commonly used to build APIs — small web services that accept incoming data, process it, and return results. Flask is perfect for this scenario because it’s simple to deploy, fast to start up, and integrates easily with Python’s data libraries such as pandas.

### Google Cloud Storage (GCS)

Google Cloud Storage (GCS) is a scalable cloud storage system used to store any type of file or object — from images to large datasets. In this pipeline, GCS serves as the centralized data lake. Raw rainfall data and processed results are both stored in separate folders within a single GCS bucket.

### Apache Parquet

Apache Parquet is an open-source, columnar file format optimized for analytical workloads. Traditional CSV files store data row by row; Parquet stores it column by column, which drastically reduces file size and improves read performance for analytical queries. Parquet also supports compression (Snappy, GZIP) and schema evolution, making it ideal for large-scale data pipelines.

### Docker

Docker is a containerization technology that packages an application and all its dependencies into a lightweight unit called a container. This ensures that the same code runs identically in development, testing, and production. Containers are especially important in cloud environments, where services like Cloud Run or Kubernetes deploy and scale them automatically.

### Google Cloud Run

Google Cloud Run is a fully managed platform that runs containers. It automatically handles scaling, load balancing, and version management. Developers only need to provide a container image; Cloud Run takes care of the rest. For this pipeline, Cloud Run hosts the Flask service that receives data, processes it, and writes Parquet files to GCS.

---

## Why Build a Data Pipeline?

A data pipeline is an automated system that moves data from its source to its destination in a clean, reliable, and structured way.  
For rainfall or climate-based applications, data is collected from multiple sources — CSVs, JSON APIs, or daily measurements — which often have missing values, inconsistent timestamps, or incorrect coordinates.

Without a structured ingestion and processing pipeline:

- Analysts and data scientists spend most of their time cleaning data.  
- Data versions become difficult to track.  
- Scalability and reproducibility become major challenges.

By building a pipeline, the process becomes standardized:

1. Data arrives at a single entry point (the API).  
2. It’s validated, cleaned, and normalized automatically.  
3. The final data is stored in a compressed, query-ready format (Parquet).

---

## System Architecture

Below is a simplified view of the system architecture.

```mermaid
flowchart LR
  A[Data Source: IMD Rainfall CSV/API] --> B[Flask API (Cloud Run)]
  B --> C[Processing: Pandas + PyArrow]
  C --> D[Write to Parquet]
  D --> E[GCS Bucket (raw/ and processed/ folders)]
  E --> F[Analytics Tools: BigQuery / Dashboards / ML]
```

The process begins with the data source (for example, a CSV file from IMD).  
The data is sent to the Flask API hosted on Cloud Run. Flask processes the data using pandas, converts it to Parquet with pyarrow, and stores the resulting file in a Google Cloud Storage bucket.

This architecture supports both real-time ingestion (small payloads) and batch ingestion (larger files uploaded via signed URLs).

---

## Designing the Flask Application

The ingestion layer is a small Flask service that exposes HTTP endpoints. It performs three primary tasks:

1. Receives incoming rainfall data (either as uploaded files or JSON payloads).  
2. Processes and validates the data.  
3. Saves both raw and processed outputs in GCS.

### Directory structure

```
weather-pipeline/
├─ app/
│  ├─ main.py
│  ├─ processor.py
│  └─ config.py
├─ requirements.txt
├─ Dockerfile
└─ README.md
```

### The main Flask service

```python
# app/main.py
from flask import Flask, request, jsonify
from processor import process_dataframe, process_gcs_object
import os

app = Flask(__name__)
BUCKET = os.environ.get("GCS_BUCKET")

@app.route("/health")
def health():
    return "ok", 200

@app.route("/upload", methods=["POST"])
def upload():
    if 'file' not in request.files:
        return jsonify({"error": "file not provided"}), 400
    file = request.files['file']
    content = file.read()
    result = process_dataframe(content, file.content_type)
    return jsonify({"stored": result}), 201
```

The `/upload` endpoint handles small CSV or JSON files directly. For large datasets, it’s more efficient to use a signed upload URL — a temporary, secure URL that allows direct upload to GCS. Once the client uploads a file, the system receives a notification through a `/notify` endpoint and processes it asynchronously.

---

## Processing and Writing to Parquet

The processing logic uses pandas for data manipulation and pyarrow for writing Parquet files.

```python
# app/processor.py
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from google.cloud import storage
import io, os
from datetime import datetime

BUCKET = os.environ.get("GCS_BUCKET")

def normalize(df):
    df['date'] = pd.to_datetime(df['date'], errors='coerce', utc=True)
    df['rain'] = pd.to_numeric(df['rain'], errors='coerce').fillna(0)
    df = df.dropna(subset=['date', 'lat', 'lon'])
    return df

def write_parquet(df, path):
    table = pa.Table.from_pandas(df)
    buffer = io.BytesIO()
    pq.write_table(table, buffer, compression='snappy')
    buffer.seek(0)
    storage.Client().bucket(BUCKET).blob(path).upload_from_file(buffer)
    return f"gs://{BUCKET}/{path}"

def process_dataframe(content, content_type):
    if 'json' in content_type:
        df = pd.read_json(io.BytesIO(content), lines=True)
    else:
        df = pd.read_csv(io.BytesIO(content))
    df = normalize(df)
    df['year'] = df['date'].dt.year
    df['month'] = df['date'].dt.month
    for (year, month), group in df.groupby(['year', 'month']):
        path = f"processed/year={year}/month={month}/data-{datetime.utcnow().timestamp()}.parquet"
        write_parquet(group, path)
    return path
```

Here, data is grouped by year and month before being written to Parquet files. This approach makes future queries more efficient because analytical tools can scan only the relevant partitions instead of entire datasets.

---

## Why Parquet Is a Game-Changer

CSV files are great for simple storage and human readability, but they quickly become inefficient as datasets grow. Every query must read the entire file, even if it needs only one column.

Parquet solves this problem by storing data column-wise. This means:

- Only the required columns are read during queries.  
- Compression is applied more effectively.  
- Schema and metadata are stored within the file, preserving data types.

For example, a 1 GB rainfall CSV dataset typically compresses to around 200–300 MB when stored in Parquet format, while remaining fully queryable in analytics tools such as BigQuery or Apache Spark.

---

## Containerization with Docker

To deploy the pipeline reliably, the application is packaged into a Docker container. This ensures that all dependencies — Flask, pandas, pyarrow, and Google Cloud libraries — are included.

```Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app /app
CMD ["gunicorn", "main:app", "--bind", "0.0.0.0:8080", "--workers", "2"]
```

Once built, this container can be run locally or deployed directly to Cloud Run.

---

## Deploying on Google Cloud Run

Google Cloud Run simplifies deployment by running the Docker container as a serverless service. It automatically scales based on incoming requests, handles HTTPS, and integrates with IAM for secure access to GCS.

```bash
gcloud builds submit --tag gcr.io/<PROJECT_ID>/weather-pipeline
gcloud run deploy weather-pipeline   --image gcr.io/<PROJECT_ID>/weather-pipeline   --region us-central1   --allow-unauthenticated   --set-env-vars GCS_BUCKET=<BUCKET_NAME>
```

The service account attached to Cloud Run must have permission to write to the storage bucket, typically via the role `roles/storage.objectAdmin`.

---

## Handling Large Files and Memory Efficiency

Rainfall datasets, especially when aggregated across years, can become quite large. Processing large files in-memory is inefficient and can easily exceed available RAM. To handle this, chunk-based processing is used.

By reading data in chunks using `pd.read_csv(chunksize=...)`, only a subset of rows is processed at a time, normalized, and written to Parquet in parts. This approach keeps memory usage constant regardless of total file size.

It’s also recommended to batch uploads and avoid creating too many small Parquet files, as each object in cloud storage adds metadata overhead.

---

## Automation and Continuous Deployment

Once the pipeline is stable, automation ensures consistency and reliability.  
Cloud Build or GitHub Actions can be used to automatically:

- Build the Docker image on every commit.  
- Run tests and lint checks.  
- Deploy to Cloud Run.

Terraform can be used to define GCS buckets, IAM permissions, and Cloud Run configurations as infrastructure as code, making the setup reproducible across environments.

---

## Observability and Monitoring

A production-grade pipeline must be observable.

**Key metrics to monitor:**

- Number of files ingested per day  
- Average processing latency  
- Error rate during parsing or uploads  
- Size of output Parquet files

Google Cloud’s Operations Suite (formerly Stackdriver) integrates directly with Cloud Run logs, allowing easy monitoring and alerting.

Structured logs in JSON format make it possible to filter and analyze specific events, such as failed uploads or schema mismatches.

---

## Common Challenges and Their Solutions

- **Authentication issues** – Cloud Run uses the runtime service account for authentication. If GCS writes fail, it usually indicates missing IAM permissions.  
- **Timeouts for large files** – Cloud Run has a 60-minute request timeout limit. For larger datasets, use asynchronous processing with Cloud Tasks or Pub/Sub.  
- **Schema drift** – Input data may change over time (for example, new columns added). Schema validation before processing helps maintain consistency.  
- **Small-file problem** – Writing many small Parquet files increases overhead. Combining chunks or running a nightly merge job can improve performance.  
- **Data integrity** – Always retain the raw unprocessed data in a `raw/` folder in GCS. It provides an immutable source for reprocessing or audit.

---

## Performance and Cost Considerations

Using Parquet significantly reduces both storage and query costs:

- Parquet compression (Snappy) typically reduces data size by 60–80%.  
- Analytical queries scan fewer bytes, reducing BigQuery costs.  
- Cloud Run’s pay-per-use pricing means the service incurs zero cost when idle.

Batching uploads and minimizing small files further reduces GCS metadata and network overhead.

---

## The End Result

After deployment, the entire system works autonomously:

1. New rainfall files are uploaded through the API or signed URL.  
2. Flask processes and stores cleaned Parquet files in GCS.  
3. Downstream systems such as BigQuery or dashboards can directly query the processed data.

### Final storage structure

```
gs://weather-pipeline-bucket/
├─ raw/
│  ├─ 2025-10-01-imd.csv
├─ processed/
│  ├─ year=2025/
│  │  ├─ month=10/
│  │  │  ├─ data-1730000000.parquet
```

This approach ensures data is versioned, auditable, and analytics-ready.

---

## Lessons from Implementation

Building production pipelines reveals important patterns:

- Always store raw data before processing. It’s invaluable for debugging and reprocessing.  
- Partition data by access patterns (such as time and region) instead of arbitrary categories.  
- Schema validation prevents downstream corruption.  
- Use signed URLs for client uploads to avoid exposing service account credentials.  
- Test with real-size data to uncover scaling or timeout issues early.  
- Keep processing stateless so Cloud Run can scale horizontally under load.

---

## Closing Thoughts

This pipeline represents a modern, serverless approach to data engineering. It’s lightweight but scalable, secure, and cost-efficient.  
Using Flask for ingestion, Parquet for efficient storage, and Cloud Run for deployment forms a clean separation of responsibilities — each tool doing what it does best.

As datasets continue to grow, these architectural patterns — stateless services, schema validation, partitioned storage, and containerized deployment — become essential foundations for reliable data platforms.
