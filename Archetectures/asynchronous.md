**Synchronous**: the client requests a prediction and must wait for the model service to return a prediction. This is suitable for small models that require a small number of computations, or where the client cannot continue other processing steps without a prediction.

**Asynchronous**: instead of directly returning a prediction the model service will return a unique identifier for a task. Whilst the prediction task is being completed by the model service the client is free to continue other processing. The result can then be fetched via a results endpoint using the unique task id.

<img width="758" alt="image" src="https://github.com/jyotiyadav94/Machine-Learning-Engineer-Roadmap/assets/72126242/2ddbd9ea-676e-4edf-ad4f-37fbe3c398e6">


Here's a detailed design for a backend architecture that incorporates Celery, Redis, FastAPI, and RabbitMQ to handle multiple machine learning model requests efficiently:

Architecture Overview

1. **Client(Frontend)**: Initiates prediction requests with feature data (JSON) via a POST request to the FastAPI endpoint.

2. **FastAPI(API Gateway)**:
    - Handles incoming requests.
    - Validates request data against a predefined schema.
    - Creates a Celery task (if validation succeeds) and passes it to the message broker (RabbitMQ).
    - Returns a unique task ID to the client for tracking.
    - Provides a results endpoint for clients to poll for prediction outcomes.

3. **RabbitMQ(Message Broker)**:
    - Queues and distributes tasks to available Celery workers.
    - Ensures reliable task delivery and load balancing.

4. **Celery Workers(Prediction Engines)**:
    - Pick up tasks from the queue.
    - Execute predictions using the pre-trained ML model.
    - Store the prediction results in the Redis backend.

5. **Redis(Result Store)**:
    - Acts as a fast, in-memory data store for prediction results.
    - Allows efficient retrieval of results based on the task ID.


https://towardsdatascience.com/deploying-ml-models-in-production-with-fastapi-and-celery-7063e539a5db


For example :


# Scenario: 100 Tasks Submitted

## Client Submits Requests
The client (frontend) sends 100 POST requests to the `/predict` endpoint with the feature data for each prediction.

## FastAPI Handles and Queues

1. **Validation and Task Creation**:
   - FastAPI validates each request's data in parallel (assuming you have multiple worker processes or threads configured).
   - For each valid request, it creates a Celery task and sends it to RabbitMQ.
   - FastAPI returns 100 unique task IDs to the client.

## RabbitMQ Distributes Tasks
RabbitMQ efficiently distributes these 100 tasks across available Celery workers. If you have fewer than 100 workers, some tasks will wait in the queue until workers become available.

## Celery Workers Process

Each worker picks up a task, loads the ML model (if not already in memory), performs the prediction, and stores the result in Redis under the corresponding task ID.

## Client Polls for Results

The client can start polling the `/results/{task_id}` endpoint for each of the 100 task IDs. As results become available, they're returned to the client.

## Configuration Considerations for Scaling

### Celery Worker Configuration

- **Concurrency**: Increase the number of worker processes or threads (`-c` or `--concurrency` option) to match or exceed the expected number of concurrent tasks.
- **Prefetch Multiplier**: Adjust the `worker_prefetch_multiplier` setting in your Celery config. A value of 1 means a worker will prefetch one task at a time. Increasing this can improve throughput if task execution time is short.
- **Rate Limiting**: If your model is computationally intensive, you might want to rate-limit the tasks at the worker level to prevent overwhelming your resources.

### RabbitMQ Configuration

- **Queue Size**: Ensure your RabbitMQ queue settings can handle a large number of tasks. You might need to increase the queue length if you anticipate long bursts of requests.

### Redis Configuration

- **Memory**: Ensure you have enough Redis memory to store the results of all concurrent tasks. Redis is in-memory, so plan accordingly.

### FastAPI Configuration

- **Worker Processes/Threads**: Increase the number of worker processes (Gunicorn) or threads (Uvicorn) based on your hardware and the request validation complexity.
