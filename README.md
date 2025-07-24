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
docker compose up airflow-init
```

### Step 5: Start Airflow Services

```bash
docker compose up -d
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

### Basic DAG Structure : Hello World DAG

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

#### Before running:

- Ensure the DAG appears in UI (wait 30-60 seconds after creating the file)
- Check for syntax errors in the Files tab

#### Testing steps:

1. Go to Airflow UI → DAGs page
2. Find "hello_world_dag" and toggle it ON
3. Click on the DAG name → Graph view
4. Click "Trigger DAG" button (play icon)
5. Watch tasks turn from white → yellow → green
6. Click each task to view logs and verify output

#### Command line testing:

```bash
# Test individual tasks
docker compose exec airflow-webserver airflow tasks test hello_world_dag hello_task 2025-01-23
docker compose exec airflow-webserver airflow tasks test hello_world_dag world_task 2025-01-23
docker compose exec airflow-webserver airflow tasks test hello_world_dag bash_task 2025-01-23
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

#### Testing approach:

1. Enable the DAG in UI
2. Trigger manually first time
3. Verify data flow between tasks by checking logs
4. Look for the XCom data exchange in task logs

#### Validation points:

- Extract task should show: `{"users": [1, 2, 3, 4, 5]}`
- Transform task should show processed user objects
- Load task should print each user with timestamp

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

#### Prerequisites setup:

```bash
# Create the file the sensor is waiting for
docker compose exec airflow-webserver touch /tmp/input_data.txt
```

#### Testing workflow:

1. Enable DAG and trigger
2. Watch the file sensor (should complete quickly since file exists)
3. Observe branching - it will randomly pick one of three paths
4. Check which branch was taken in the Graph view
5. Re-trigger multiple times to see different branches

#### Troubleshooting the sensor:

- If sensor keeps running, verify file exists in container
- Check sensor logs for file path issues

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

#### Verification steps:

1. Look for 4 separate DAGs in the UI:

   - `process_users_db`
   - `process_products_db`
   - `process_orders_db`
   - `process_analytics_db`

2. Test each DAG individually:

   - Enable one DAG
   - Trigger it
   - Verify both tasks run (process → validate)
   - Check logs show correct database name

3. Test parallel execution by triggering multiple DAGs simultaneously

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
- Check Airflow logs: `docker compose logs airflow-scheduler`

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
docker compose exec airflow-webserver airflow db check

# Reset database if needed
docker compose down
docker volume rm airflow-tutorial_postgres-db-volume
docker compose up airflow-init
docker compose up -d
```

#### 5. Performance Issues

- Monitor task execution times
- Use appropriate parallelism settings
- Consider using different executors for different workloads

### Useful Commands

```bash
# View logs
docker compose logs -f airflow-scheduler
docker compose logs -f airflow-webserver

# Execute commands in containers
docker compose exec airflow-webserver airflow dags list
docker compose exec airflow-webserver airflow tasks list your_dag_id

# Test specific tasks
docker compose exec airflow-webserver airflow tasks test your_dag_id your_task_id 2025-01-01

# Restart services
docker compose restart airflow-scheduler
docker compose restart airflow-webserver
```

## 7. General Testing Best Practices

### Pre-Testing Checklist

1. **Syntax Check**: `docker compose exec airflow-webserver python /opt/airflow/dags/your_dag.py`
2. **DAG Validation**: Look for red error indicators in UI
3. **Import Check**: Verify no import errors in scheduler logs

### Systematic Testing Approach

#### Phase 1 - Individual Task Testing

- Test each task in isolation using `airflow tasks test`
- Verify task logic works correctly
- Check for proper error handling

#### Phase 2 - DAG Flow Testing

- Enable DAG and trigger once
- Monitor execution flow in Graph view
- Verify task dependencies execute in correct order

#### Phase 3 - Schedule Testing

- Let DAG run on schedule (or use backfill for faster testing)
- Check multiple runs don't overlap incorrectly
- Verify catchup behavior

#### Phase 4 - Failure Testing

- Intentionally cause task failures (modify code temporarily)
- Verify retry behavior works
- Test task failure notifications

### Monitoring During Tests

#### Key places to check:

1. **Graph View**: Visual task status and flow
2. **Gantt Chart**: Task timing and overlaps
3. **Task Logs**: Detailed execution information
4. **Code View**: Verify DAG structure
5. **Landing Times**: Schedule adherence

#### Log locations:

- **Task-specific logs**: Click task → View Log
- **Scheduler logs**: `docker compose logs airflow-scheduler`
- **Webserver logs**: `docker compose logs airflow-webserver`

### Performance Testing Tips

1. **Load Testing**: Trigger multiple DAG runs simultaneously
2. **Resource Monitoring**: Check Docker container resource usage
3. **Database Performance**: Monitor postgres container during heavy loads
4. **Memory Usage**: Watch for memory leaks in long-running tests

### Automated Testing Setup

#### Create test scripts:

1. Write shell scripts to trigger DAGs and check status
2. Use Airflow REST API for automated testing
3. Set up alerts for test failures
4. Create test data fixtures for consistent testing

#### Example test validation:

- Check DAG completes within expected time
- Verify all tasks reach success state
- Validate output data/logs contain expected content
- Ensure no tasks are skipped unexpectedly

### Troubleshooting Failed Tests

#### Common issues to check:

1. **Task hanging**: Check for infinite loops or missing timeouts
2. **Import errors**: Verify all dependencies are available in container
3. **Permission issues**: Check file/directory permissions
4. **Resource limits**: Ensure adequate memory/CPU allocation
5. **Network issues**: Verify external service connectivity

#### Quick debugging commands:

```bash
# Check DAG bag errors
docker-compose exec airflow-webserver airflow dags list-import-errors

# Test DAG parsing
docker-compose exec airflow-webserver airflow dags show your_dag_id

# Check task instance status
docker-compose exec airflow-webserver airflow tasks state your_dag_id your_task_id 2025-01-23
```

## 8. Testing Checklist Summary

### ✅ Before Testing Any DAG:

- [ ] Services are running (`docker compose ps`)
- [ ] DAG appears in UI without errors
- [ ] Syntax check passes
- [ ] Required files/dependencies exist

### ✅ For Each DAG:

- [ ] Enable DAG in UI
- [ ] Trigger manual run
- [ ] Monitor execution in Graph view
- [ ] Check task logs for expected output
- [ ] Verify task dependencies work correctly
- [ ] Test failure scenarios

### ✅ After Testing:

- [ ] Document any issues found
- [ ] Verify performance is acceptable
- [ ] Check for resource leaks
- [ ] Confirm scheduling works as expected

This testing approach ensures each DAG works correctly individually and integrates properly with the Airflow environment.

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
