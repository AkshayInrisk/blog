# Building a Data Pipeline on Google Cloud Using Flask and Parquet

In today’s data-driven world, the ability to move, process, and store information efficiently is the backbone of any analytics or decision-making system. Behind most modern dashboards, forecasts, and models lies a silent workhorse — the data pipeline. This article dives deep into how a production-grade data pipeline can be built using Flask, Google Cloud Storage (GCS), and Parquet, with a focus on reliability, scalability, and automation.

## Why Data Pipelines Matter

A data pipeline is a structured system that automates the flow of data from one place to another — from raw collection to storage, transformation, and ultimately, analysis. Imagine hundreds of rainfall stations sending daily data or satellite feeds updating hourly. Without an automated pipeline, analysts would spend hours downloading, cleaning, and combining these files manually.

A robust pipeline ensures three key things:
1. **Consistency** — Data is processed the same way every time.
2. **Scalability** — It can handle large or growing volumes of input.
3. **Traceability** — Each step can be monitored, logged, and debugged.

In the insurance domain, where rainfall or temperature data can trigger financial payouts, accuracy and reproducibility are critical. This is exactly where cloud-based pipelines shine.

## The Tools

Before getting into implementation, let’s break down the stack:

- **Flask**: A lightweight Python web framework that helps create REST APIs. It acts as the entry point of the pipeline — users or schedulers can trigger processing via HTTP routes.
- **Pandas**: A Python library for data manipulation. It makes it easy to clean, transform, and analyze structured data.
- **Parquet**: A columnar storage file format designed for efficient querying and compression. Unlike CSVs, which store data row by row, Parquet stores data by columns — making it faster to read, smaller in size, and ideal for cloud analytics.
- **Google Cloud Storage (GCS)**: A scalable object storage solution on Google Cloud Platform (GCP). It’s where input and output data are stored — similar to AWS S3.
- **Docker**: A tool that packages applications into containers — lightweight, portable environments that include all dependencies. This makes deployment consistent across machines.
- **Google Cloud Run**: A managed service that runs containerized applications without worrying about servers. You push your container, and Cloud Run handles scaling and routing.

Each of these tools plays a distinct role: Flask handles API logic, Pandas does the processing, Parquet is the output format, GCS is the storage backend, and Docker + Cloud Run ensure smooth deployment.

## Architecture Overview

Here’s a simple breakdown of the system:

```
Data Source → Flask API → Data Processing (Pandas) → Save as Parquet → Upload to GCS → Cloud Run Deployment
```

1. The API receives an input file or trigger request.
2. Data is cleaned and processed using Pandas.
3. The processed output is stored as a Parquet file.
4. The file is uploaded to a specific bucket on GCS.
5. Everything runs inside a Docker container deployed on Cloud Run.

This design ensures modularity and scalability — any part (processing logic, storage, etc.) can evolve independently.

## Step 1: Data Ingestion

The ingestion step involves reading the raw data — typically from a CSV file. A minimal Flask endpoint looks like this:

```python
from flask import Flask, request, jsonify
import pandas as pd
from google.cloud import storage

app = Flask(__name__)

@app.route('/process', methods=['POST'])
def process_file():
    file = request.files['file']
    df = pd.read_csv(file)

    # Clean or transform data here
    df['rain'] = df['rain'].fillna(0)

    output_path = 'output.parquet'
    df.to_parquet(output_path)

    upload_to_gcs(output_path, 'your-bucket-name', 'output/output.parquet')
    return jsonify({'message': 'File processed and uploaded successfully'})
```

The route `/process` accepts a file upload, processes it, saves it as Parquet, and uploads it to GCS.

## Step 2: Uploading to Google Cloud Storage

The upload function uses the Google Cloud Storage Python client:

```python
def upload_to_gcs(local_path, bucket_name, blob_name):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    blob.upload_from_filename(local_path)
```

This ensures files are securely pushed to a defined GCS bucket. You can control access via IAM roles, ensuring only the service account of Cloud Run can write to it.

## Step 3: Understanding Parquet

Parquet deserves a closer look. It’s not just a file format — it’s a huge performance improvement over CSV for analytical workloads.

| Feature | CSV | Parquet |
|----------|-----|----------|
| Storage Type | Row-based | Columnar |
| Compression | None/Minimal | Built-in (Snappy, Gzip) |
| Read Speed | Slower | Faster for large data |
| Schema | None | Self-describing |
| File Size | Larger | Much smaller |

When dealing with years of rainfall data, this difference matters. A 1GB CSV may compress down to 200MB or less in Parquet, with faster read speeds and better query performance in tools like BigQuery or Spark.

## Step 4: Containerization with Docker

Now that the Flask app works locally, it’s time to containerize it. Docker ensures the same environment runs everywhere.

A simple `Dockerfile` for this setup:

```Dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["gunicorn", "-b", ":8080", "main:app"]
```

This image uses Gunicorn (a production-ready WSGI server) to serve Flask. Once the container is ready, it can be deployed anywhere.

To build and test locally:
```bash
docker build -t rainfall-pipeline .
docker run -p 8080:8080 rainfall-pipeline
```

Visit `http://localhost:8080/process` to check your API.

## Step 5: Deploying on Google Cloud Run

Once Docker is working, deployment is straightforward.

```bash
gcloud builds submit --tag gcr.io/<your-project-id>/rainfall-pipeline
gcloud run deploy rainfall-pipeline     --image gcr.io/<your-project-id>/rainfall-pipeline     --platform managed     --region us-central1     --allow-unauthenticated
```

Cloud Run automatically scales your service based on incoming requests — from zero to hundreds of instances if needed.

## Common Pitfalls and Fixes

Even simple pipelines can fail due to overlooked details. Here are some lessons learned:

- **Authentication Errors:** Cloud Run needs a service account with proper GCS permissions (`roles/storage.objectAdmin`).  
- **Large Files:** For very large datasets, streaming uploads or chunked processing helps avoid memory overload.  
- **Library Mismatch:** Always fix dependencies in `requirements.txt` to avoid version conflicts.  
- **Timeouts:** For longer operations, increase request timeouts or move heavy lifting to a background job (e.g., Cloud Tasks or Pub/Sub).

## Step 6: Automating the Pipeline

Automation turns this from a manual trigger into a true data pipeline. You can use:

- **Cloud Scheduler** to send HTTP requests to your Flask endpoint at set intervals.
- **Pub/Sub Triggers** to process new files automatically when they appear in GCS.
- **Workflow Orchestration Tools** like Airflow for managing dependencies between multiple stages.

A simple Cloud Scheduler job can hit your endpoint daily with:

```bash
gcloud scheduler jobs create http process-daily-data     --schedule="0 0 * * *"     --uri="https://rainfall-pipeline-abc123.a.run.app/process"     --http-method=POST
```

## Step 7: Performance Insights

Switching from CSV to Parquet alone can reduce storage by **70–80%**, and data load times by up to **5x** in downstream analytics.  
Moreover, Cloud Run’s auto-scaling and stateless nature mean your pipeline can handle spikes in traffic (like seasonal rainfall data dumps) without any manual scaling.

Logging and monitoring can be done through **Cloud Logging**, while metrics like memory, CPU usage, and request latency can be viewed in **Cloud Monitoring**.

## Final Architecture Diagram

```
┌─────────────────────┐
│  Data Source (CSV)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Flask API (GCR)   │
│  Data Cleaning +     │
│  Parquet Conversion  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Google Cloud       │
│  Storage (GCS)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Downstream Systems │
│  (Analytics, ML)    │
└─────────────────────┘
```

## Conclusion

This pipeline design balances simplicity and scalability. By combining Flask’s flexibility with the performance of Parquet and the reliability of Google Cloud, you get a system that can process large volumes of data efficiently. Containerization with Docker and deployment on Cloud Run ensure that the setup is reproducible and maintenance-free.

While this example focuses on rainfall data, the same architecture applies to any structured dataset — financial transactions, IoT sensors, or health monitoring data. In the long run, investing time in building such automated, cloud-native pipelines pays off through cleaner, faster, and more reliable data delivery.

