# Apache Airflow DAG Tutorial with Docker Compose

This comprehensive tutorial will guide you through setting up Apache Airflow with the latest version using Docker Compose and creating your first DAGs (Directed Acyclic Graphs).

## Prerequisites

- Docker installed on your system
- Docker Compose installed
- Basic understanding of Python
- At least 4GB of RAM available for Docker

## Table of Contents

1. [Setting up Apache Airflow with Docker Compose](#setup)
2. [Understanding the Docker Compose Configuration](#docker-config)
3. [Creating Your First DAG](#first-dag)
4. [Advanced DAG Examples](#advanced-examples)
5. [Best Practices](#best-practices)
6. [Troubleshooting](#troubleshooting)

## 1. Setting up Apache Airflow with Docker Compose {#setup}

### Step 1: Create Project Directory

```bash
mkdir airflow-tutorial
cd airflow-tutorial
```

### Step 2: Download the Official Docker Compose File

```bash
# Download the latest official docker-compose.yaml
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'
```

### Step 3: Create Required Directories

```bash
mkdir -p ./dags ./logs ./plugins ./config
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

### Step 4: Initialize the Database

```bash
docker-compose up airflow-init
```

### Step 5: Start Airflow Services

```bash
docker-compose up -d
```

The services will be available at:
- **Airflow Webserver**: http://localhost:8080
- **Default credentials**: Username: `airflow`, Password: `airflow`

## 2. Understanding the Docker Compose Configuration {#docker-config}

The official docker-compose.yaml includes these key services:

- **airflow-webserver**: The web UI
- **airflow-scheduler**: Schedules and triggers tasks
- **airflow-worker**: Executes tasks (in CeleryExecutor mode)
- **airflow-triggerer**: Handles deferrable tasks
- **postgres**: Database backend
- **redis**: Message broker for Celery

### Key Environment Variables

```yaml
environment:
  AIRFLOW__CORE__EXECUTOR: CeleryExecutor
  AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
  AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
  AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
```

## 3. Creating Your First DAG {#first-dag}

### Basic DAG Structure

Create a file `dags/hello_world_dag.py`:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator

# Default arguments for the DAG
default_args = {
    'owner': 'tutorial',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

# Define the DAG
dag = DAG(
    'hello_world_dag',
    default_args=default_args,
    description='A simple hello world DAG',
    schedule='@daily',  # Run daily
    catchup=False,  # Don't run for past dates
    tags=['tutorial', 'beginner'],
)

def print_hello():
    """Simple Python function to print hello"""
    print("Hello from Airflow!")
    return "Hello task completed"

def print_world():
    """Another Python function"""
    print("World from Airflow!")
    return "World task completed"

# Define tasks
hello_task = PythonOperator(
    task_id='hello_task',
    python_callable=print_hello,
    dag=dag,
)

world_task = PythonOperator(
    task_id='world_task',
    python_callable=print_world,
    dag=dag,
)

bash_task = BashOperator(
    task_id='bash_task',
    bash_command='echo "Hello from Bash!" && date',
    dag=dag,
)

# Define task dependencies
hello_task >> world_task >> bash_task
```

### TaskFlow API (Modern Approach)

Create `dags/taskflow_example.py`:

```python
from datetime import datetime, timedelta
from airflow.decorators import dag, task

@dag(
    schedule='@daily',
    start_date=datetime(2025, 1, 1),
    catchup=False,
    tags=['tutorial', 'taskflow'],
    default_args={
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
    }
)
def taskflow_example():
    """
    Example DAG using TaskFlow API
    """
    
    @task
    def extract():
        """Extract data"""
        data = {"users": [1, 2, 3, 4, 5]}
        return data
    
    @task
    def transform(data: dict):
        """Transform data"""
        transformed = []
        for user_id in data["users"]:
            transformed.append({
                "user_id": user_id,
                "processed": True,
                "timestamp": datetime.now().isoformat()
            })
        return {"processed_users": transformed}
    
    @task
    def load(data: dict):
        """Load data"""
        print(f"Loading {len(data['processed_users'])} users")
        for user in data["processed_users"]:
            print(f"User {user['user_id']} processed at {user['timestamp']}")
        return "Data loaded successfully"
    
    # Define the workflow
    extracted_data = extract()
    transformed_data = transform(extracted_data)
    load(transformed_data)

# Instantiate the DAG
taskflow_dag = taskflow_example()
```

## 4. Advanced DAG Examples {#advanced-examples}

### Data Pipeline with Sensors and Branching

Create `dags/advanced_pipeline.py`:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.dummy import DummyOperator
from airflow.sensors.filesystem import FileSensor
from airflow.operators.email import EmailOperator
import random

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'advanced_data_pipeline',
    default_args=default_args,
    description='Advanced data pipeline with sensors and branching',
    schedule='0 6 * * *',  # Run at 6 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['production', 'data-pipeline'],
)

def check_data_quality(**context):
    """Simulate data quality check"""
    quality_score = random.uniform(0.7, 1.0)
    print(f"Data quality score: {quality_score}")
    
    if quality_score > 0.9:
        return "high_quality_processing"
    elif quality_score > 0.7:
        return "standard_processing"
    else:
        return "data_quality_failed"

def process_high_quality_data(**context):
    """Process high quality data"""
    print("Processing high quality data with advanced algorithms")
    return "high_quality_complete"

def process_standard_data(**context):
    """Process standard quality data"""
    print("Processing standard quality data")
    return "standard_complete"

def handle_quality_failure(**context):
    """Handle data quality failure"""
    print("Data quality check failed - sending alert")
    return "quality_failure_handled"

# File sensor to wait for input data
wait_for_file = FileSensor(
    task_id='wait_for_input_file',
    filepath='/tmp/input_data.txt',
    fs_conn_id='fs_default',
    poke_interval=30,
    timeout=300,
    dag=dag,
)

# Branch based on data quality
quality_check = BranchPythonOperator(
    task_id='data_quality_check',
    python_callable=check_data_quality,
    dag=dag,
)

# Processing paths
high_quality_task = PythonOperator(
    task_id='high_quality_processing',
    python_callable=process_high_quality_data,
    dag=dag,
)

standard_quality_task = PythonOperator(
    task_id='standard_processing',
    python_callable=process_standard_data,
    dag=dag,
)

quality_failure_task = PythonOperator(
    task_id='data_quality_failed',
    python_callable=handle_quality_failure,
    dag=dag,
)

# Convergence point
join_task = DummyOperator(
    task_id='join_processing',
    trigger_rule='none_failed_min_one_success',
    dag=dag,
)

# Final notification
notify_completion = DummyOperator(
    task_id='notify_completion',
    dag=dag,
)

# Define workflow
wait_for_file >> quality_check
quality_check >> [high_quality_task, standard_quality_task, quality_failure_task]
[high_quality_task, standard_quality_task, quality_failure_task] >> join_task >> notify_completion
```

### Dynamic DAG Generation

Create `dags/dynamic_dag_example.py`:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

# Configuration for dynamic tasks
DATABASES = ['users_db', 'products_db', 'orders_db', 'analytics_db']

def create_processing_dag(db_name):
    """Create a DAG for processing a specific database"""
    
    def process_database(**context):
        print(f"Processing database: {db_name}")
        # Simulate processing time
        import time
        time.sleep(2)
        return f"{db_name} processed successfully"
    
    def validate_database(**context):
        print(f"Validating database: {db_name}")
        return f"{db_name} validation complete"
    
    dag = DAG(
        f'process_{db_name}',
        default_args={
            'owner': 'data-ops',
            'depends_on_past': False,
            'start_date': datetime(2025, 1, 1),
            'retries': 1,
            'retry_delay': timedelta(minutes=5),
        },
        description=f'Process {db_name} database',
        schedule='0 2 * * *',  # Run at 2 AM daily
        catchup=False,
        tags=['dynamic', 'database', db_name],
    )
    
    process_task = PythonOperator(
        task_id=f'process_{db_name}',
        python_callable=process_database,
        dag=dag,
    )
    
    validate_task = PythonOperator(
        task_id=f'validate_{db_name}',
        python_callable=validate_database,
        dag=dag,
    )
    
    process_task >> validate_task
    
    return dag

# Create DAGs dynamically
for db in DATABASES:
    globals()[f'process_{db}_dag'] = create_processing_dag(db)
```

## 5. Best Practices {#best-practices}

### DAG Design Principles

1. **Keep DAGs Simple**: Each DAG should have a single responsibility
2. **Use Descriptive Names**: Both for DAGs and tasks
3. **Set Proper Timeouts**: Avoid infinite running tasks
4. **Handle Failures Gracefully**: Use retries and proper error handling
5. **Use XComs Sparingly**: For small data exchange between tasks

### Configuration Best Practices

```python
# Good practices in DAG definition
default_args = {
    'owner': 'team-name',  # Always specify owner
    'depends_on_past': False,  # Usually False for better parallelism
    'start_date': datetime(2025, 1, 1),  # Use specific dates
    'email_on_failure': True,  # Enable notifications
    'email_on_retry': False,  # Usually not needed
    'retries': 2,  # Reasonable retry count
    'retry_delay': timedelta(minutes=5),  # Reasonable delay
    'max_active_runs': 1,  # Prevent overlapping runs
}

dag = DAG(
    'well_designed_dag',
    default_args=default_args,
    description='Clear description of what this DAG does',
    schedule='@daily',  # Use cron or presets
    catchup=False,  # Usually False for new DAGs
    tags=['team', 'environment', 'purpose'],  # Helpful for organization
    doc_md=__doc__,  # Include documentation
)
```

### Task Dependencies Patterns

```python
# Sequential
task_a >> task_b >> task_c

# Parallel branches
task_a >> [task_b, task_c] >> task_d

# Complex dependencies
task_a >> task_b
task_a >> task_c
[task_b, task_c] >> task_d

# Cross dependencies (use with caution)
from airflow.models import Variable
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

trigger_other_dag = TriggerDagRunOperator(
    task_id='trigger_downstream_dag',
    trigger_dag_id='downstream_processing',
    dag=dag,
)
```

### Error Handling and Monitoring

```python
def robust_task(**context):
    """Example of robust task with error handling"""
    try:
        # Your main logic here
        result = perform_operation()
        
        # Log success
        print(f"Task completed successfully: {result}")
        return result
        
    except SpecificException as e:
        # Handle specific errors
        print(f"Specific error occurred: {e}")
        # Maybe send to dead letter queue or retry with different params
        raise
        
    except Exception as e:
        # Handle unexpected errors
        print(f"Unexpected error: {e}")
        # Send alert to monitoring system
        send_alert(f"Task failed: {context['task_instance'].task_id}")
        raise
    
    finally:
        # Cleanup code
        cleanup_resources()
```

## 6. Troubleshooting {#troubleshooting}

### Common Issues and Solutions

#### 1. DAG Not Appearing in UI
- Check DAG syntax: `python dags/your_dag.py`
- Verify DAG is in the correct directory
- Check Airflow logs: `docker-compose logs airflow-scheduler`

#### 2. Task Failures
- Check task logs in the Airflow UI
- Verify Python dependencies are installed
- Check file permissions and paths

#### 3. Memory Issues
- Increase Docker memory allocation
- Optimize task memory usage
- Use appropriate pool configurations

#### 4. Database Connection Issues
```bash
# Check database connectivity
docker-compose exec airflow-webserver airflow db check

# Reset database if needed
docker-compose down
docker volume rm airflow-tutorial_postgres-db-volume
docker-compose up airflow-init
docker-compose up -d
```

#### 5. Performance Issues
- Monitor task execution times
- Use appropriate parallelism settings
- Consider using different executors for different workloads

### Useful Commands

```bash
# View logs
docker-compose logs -f airflow-scheduler
docker-compose logs -f airflow-webserver

# Execute commands in containers
docker-compose exec airflow-webserver airflow dags list
docker-compose exec airflow-webserver airflow tasks list your_dag_id

# Test specific tasks
docker-compose exec airflow-webserver airflow tasks test your_dag_id your_task_id 2025-01-01

# Restart services
docker-compose restart airflow-scheduler
docker-compose restart airflow-webserver
```

## Conclusion

This tutorial covered the essentials of setting up Apache Airflow with Docker Compose and creating effective DAGs. Key takeaways:

1. Use the official Docker Compose configuration as a starting point
2. Leverage the TaskFlow API for modern, clean DAG definitions
3. Follow best practices for maintainable and reliable workflows
4. Implement proper error handling and monitoring
5. Use dynamic DAG generation for scalable architectures

For production deployments, consider additional factors like security, scaling, monitoring, and backup strategies. The Docker Compose setup shown here is excellent for development and learning, but production environments may require Kubernetes or other orchestration platforms.

## Next Steps

- Explore Airflow providers for integrations (AWS, GCP, Azure, etc.)
- Learn about Airflow's REST API for programmatic control
- Implement CI/CD pipelines for DAG deployment
- Study advanced features like custom operators and hooks
- Consider Airflow's security features for production use