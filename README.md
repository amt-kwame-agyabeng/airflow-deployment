# Airflow Deployment

## Overview

This project deploys **Apache Airflow 3.1.0** locally using Docker Compose.

The deployment provides the core Airflow components required to schedule, monitor, and execute workflows. It uses **CeleryExecutor** for distributed task execution and **GitSync** for synchronizing workflow/DAG files from a Git repository.

The long-term purpose of this deployment is to provide the Airflow platform required to orchestrate data movement from **Kafka into a Vector Database**.

---

## Architecture

The deployment consists of the following components:

| Component | Purpose |
|---|---|
| Airflow API Server | Provides the Airflow Web UI and API |
| Airflow Scheduler | Schedules and manages DAG tasks |
| Airflow Worker | Executes tasks through Celery |
| Airflow Triggerer | Handles deferred/asynchronous tasks |
| Airflow GitSync | Synchronizes DAGs/workflows from Git |
| PostgreSQL | Stores Airflow metadata |
| Redis | Celery broker and result backend |

### Architecture Flow

```text
                         ┌─────────────────────┐
                         │  Airflow API Server │
                         │       :8080         │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Airflow Scheduler   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Redis / Celery    │
                         │       Broker        │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Airflow Worker     │
                         └─────────────────────┘

              ┌───────────────────────────────┐
              │          PostgreSQL           │
              │      Airflow Metadata DB      │
              └───────────────────────────────┘

              ┌───────────────────────────────┐
              │            GitSync            │
              │       DAG Repository          │
              └───────────────────────────────┘
```

---

## Project Structure

```text
airflow-deployment/
├── config/
├── dags/
├── logs/
├── plugins/
├── .env
├── docker-compose.yml
└── README.md
```

### Directory Description

- `config/` - Airflow configuration files.
- `dags/` - Airflow DAG files.
- `logs/` - Airflow logs.
- `plugins/` - Custom Airflow plugins.
- `.env` - Environment variables used by Docker Compose.
- `docker-compose.yml` - Defines the Airflow deployment.
- `README.md` - Deployment documentation.

---

## Technologies

- Apache Airflow `3.1.0`
- Docker
- Docker Compose
- PostgreSQL `16`
- Redis `7`
- Celery
- GitSync `4.3.0`

---

## Configuration

The deployment uses the following core configuration:

```dotenv
AIRFLOW_UID=50000
AIRFLOW_GID=0
AIRFLOW_IMAGE_NAME=apache/airflow:3.1.0
AIRFLOW__CORE__EXECUTOR=CeleryExecutor
AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres/airflow
AIRFLOW__CELERY__RESULT_BACKEND=db+postgresql://airflow:airflow@postgres/airflow
AIRFLOW__CELERY__BROKER_URL=redis://:@redis:6379/0
AIRFLOW__CORE__LOAD_EXAMPLES=false
```

> Authentication should follow the authentication manager configured for the Airflow 3 deployment. Airflow 3 does not use the legacy `airflow users` CLI command.

---

## Deployment

### 1. Navigate to the project

```bash
cd ~/airflow-deployment
```

### 2. Validate the Docker Compose configuration

```bash
docker compose config --services
```

Expected services:

```text
airflow-gitsync
redis
postgres
airflow-init
airflow-scheduler
airflow-triggerer
airflow-worker
airflow-api-server
```

### 3. Initialize Airflow

Run:

```bash
docker compose up airflow-init
```

The initialization service prepares the Airflow metadata database.

### 4. Start the Airflow stack

```bash
docker compose up -d
```

### 5. Verify containers

```bash
docker compose ps
```

The following services should be running:

```text
airflow-api-server
airflow-scheduler
airflow-worker
airflow-triggerer
airflow-gitsync
airflow-postgres
airflow-redis
```

---

## Accessing Airflow

The Airflow Web UI is available at:

```text
http://localhost:8080
```

Airflow 3 uses the **API Server** for the Web UI and API.

Therefore, the deployment uses:

```text
airflow-api-server
```

rather than the older:

```text
airflow-webserver
```

---

## Authentication

This deployment uses the authentication mechanism provided by Airflow 3.

The legacy command:

```bash
airflow users list
```

is not available in Airflow 3.1.0.

Likewise, the older:

```bash
airflow users create
```

command should not be used with this deployment.

To determine the active authentication manager:

```bash
docker compose exec airflow-api-server \
  airflow config get-value core auth_manager
```

If the Simple Auth Manager is being used, Airflow generates authentication credentials automatically.

The generated credentials can be inspected inside the container with:

```bash
docker compose exec airflow-api-server \
  cat /opt/airflow/simple_auth_manager_passwords.json.generated
```

Do not commit generated credentials or passwords to Git.

---

## Health Verification

Check the Airflow health endpoint:

```bash
curl -s http://localhost:8080/api/v2/monitor/health
```

A healthy deployment should report healthy statuses for the Airflow metadata database, scheduler, and triggerer.

The deployment was successfully verified with:

- PostgreSQL metadata database healthy
- Airflow Scheduler healthy
- Airflow Triggerer healthy
- Airflow Worker connected to Redis
- Redis healthy
- Airflow API Server running
- GitSync successfully synchronizing

---

## Scheduler Verification

View scheduler logs:

```bash
docker compose logs --tail=50 airflow-scheduler
```

Expected indicators include:

```text
Starting the scheduler
Loaded executor: :CeleryExecutor:
```

The scheduler should remain running without repeated crashes or restart loops.

---

## Worker Verification

View worker logs:

```bash
docker compose logs --tail=50 airflow-worker
```

The worker should show a successful Redis connection and a ready state.

Example:

```text
Connected to redis://redis:6379/0
celery@... ready.
```

The worker uses Celery to receive and execute tasks scheduled by Airflow.

---

## Redis Verification

Check Redis:

```bash
docker compose ps redis
```

Redis should show:

```text
healthy
```

Redis is used as the Celery message broker.

---

## PostgreSQL Verification

Check PostgreSQL:

```bash
docker compose ps postgres
```

PostgreSQL should show:

```text
healthy
```

PostgreSQL stores Airflow metadata such as DAG runs, task instances, and scheduler state.

---

## GitSync

GitSync synchronizes workflow/DAG files from a Git repository.

View GitSync logs:

```bash
docker compose logs --tail=50 airflow-gitsync
```

Successful synchronization produces messages similar to:

```text
initializing repo directory
update required ref:"main"
updated successfully
```

### Current GitSync Repository

For initial deployment validation, GitSync is configured to synchronize:

```text
https://github.com/apache/airflow.git
```

This repository is being used to verify that GitSync itself is operational.

When the actual DAG repository for the Kafka → Vector Database workflow becomes available, update:

```yaml
GITSYNC_REPO
```

in `docker-compose.yml` to point to the appropriate repository.

---

## Managing the Deployment

### View all services

```bash
docker compose ps
```

### View all logs

```bash
docker compose logs
```

### Follow scheduler logs

```bash
docker compose logs -f airflow-scheduler
```

### Follow worker logs

```bash
docker compose logs -f airflow-worker
```

### Follow API Server logs

```bash
docker compose logs -f airflow-api-server
```

### Follow GitSync logs

```bash
docker compose logs -f airflow-gitsync
```

### Stop the deployment

```bash
docker compose down
```

### Start the deployment

```bash
docker compose up -d
```

---

## Kafka → Vector Database Integration

The intended workflow is:

```text
┌─────────┐
│  Kafka  │
└────┬────┘
     │
     ▼
┌───────────────┐
│ Airflow DAG   │
└───────┬───────┘
        │
        ▼
┌──────────────────┐
│ Data Processing  │
│ / Transformation │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Vector Database │
└──────────────────┘
```

The current task deploys the Airflow platform required to support this workflow.

The actual Kafka-to-Vector-Database DAG and integrations are **not included yet**, because the specific Kafka and Vector Database configuration has not been provided as part of this deployment task.

---

## Acceptance Criteria

### Airflow up and running

**Status: PASSED**

The deployment successfully provides:

- [x] Airflow API Server
- [x] Airflow Web UI on port `8080`
- [x] Airflow Scheduler
- [x] Airflow Celery Worker
- [x] Airflow Triggerer
- [x] PostgreSQL metadata database
- [x] Redis broker
- [x] GitSync
- [x] Celery worker connected to Redis
- [x] GitSync successfully synchronized

Therefore, the acceptance criterion:

> **Airflow up and running**

has been met.

---

## Future Work

The next stage of the project can include:

1. Configure GitSync with the actual DAG repository.
2. Create the Kafka-to-Vector-Database Airflow DAG.
3. Configure Kafka connectivity.
4. Configure Vector Database connectivity.
5. Test the complete data pipeline.
6. Add monitoring and alerting for DAG failures.

---

## Conclusion

Apache Airflow 3.1.0 has been successfully deployed locally using Docker Compose with **CeleryExecutor**.

The deployment includes the required Airflow components and supporting services and is ready for the next stage of development: implementing the workflow that forwards data from Kafka into the Vector Database.
