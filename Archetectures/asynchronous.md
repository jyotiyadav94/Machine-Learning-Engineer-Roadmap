# Synchronous vs Asynchronous Architecture

## Synchronous Architecture

### How it Works:
1. **Request:** The client sends a prediction request to the API endpoint.
2. **Wait:** The API endpoint blocks and waits for the ML model to process the request and generate the prediction.
3. **Response:** The API endpoint sends the prediction back to the client as soon as it's available.

### Pros:
- **Simple:** Easier to understand and implement.
- **Direct Response:** The client gets the result immediately, making it suitable for real-time applications.
- **Easier Error Handling:** Error handling is simpler since the whole process happens in a single request-response cycle.

### Cons:
- **Latency:** If the model takes a long time to process the request, the client is blocked until it completes, leading to potentially long wait times.
- **Limited Scalability:** The API can only handle a limited number of concurrent requests based on the available resources (CPU, memory, etc.). Adding more resources may not always solve the problem due to the blocking nature.
- **Inefficient Resource Usage:** Resources can be underutilized while the model is processing the request.

### Example Use Cases:
- Simple, low-latency models.
- Applications where an immediate response is critical, like fraud detection systems or real-time decision-making tools.

## Asynchronous Architecture

### How it Works:
1. **Request:** The client sends a prediction request to the API endpoint.
2. **Acknowledgement:** The API endpoint immediately acknowledges the request, often returning a task ID or status.
3. **Background Processing:** The prediction task is placed in a queue and processed in the background (e.g., by Celery workers).
4. **Result Retrieval:** The client polls the API endpoint (or receives a notification) to get the prediction results once they are ready.

### Pros:
- **Non-Blocking:** The client is not blocked while the model processes the request, allowing for better user experience and concurrent requests.
- **High Scalability:** You can easily scale the system by adding more workers to process prediction tasks concurrently.
- **Efficient Resource Usage:** Resources are used more efficiently since the API doesn't have to wait for model processing.

### Cons:
- **Complexity:** Slightly more complex to implement due to the asynchronous nature and the need for task queues, result storage, and polling mechanisms.
- **Delayed Response:** The client doesn't get the results immediately, which might not be suitable for some real-time applications.
- **Error Handling:** Error handling can be more challenging due to the distributed nature of the architecture.

### Example Use Cases:
- Complex, computationally intensive models.
- Applications where real-time responses aren't critical, like image or text analysis services, recommendation engines, etc.

## Key Points to Remember
- Choose the right architecture based on your specific requirements and constraints.
- If you need low latency and immediate responses, synchronous might be better.
- If you need high scalability and efficient resource usage, asynchronous is a better fit.
- Hybrid approaches are possible, where you use a combination of synchronous and asynchronous communication depending on the nature of the request.

![Architecture Diagram](https://github.com/jyotiyadav94/Machine-Learning-Engineer-Roadmap/assets/72126242/2ddbd9ea-676e-4edf-ad4f-37fbe3c398e6)

## Detailed Design for a Backend Architecture

### Architecture Overview

1. **Client (Frontend):** Initiates prediction requests with feature data (JSON) via a POST request to the FastAPI endpoint.
2. **FastAPI (API Gateway):**
    - Handles incoming requests.
    - Validates request data against a predefined schema.
    - Creates a Celery task (if validation succeeds) and passes it to the message broker (RabbitMQ).
    - Returns a unique task ID to the client for tracking.
    - Provides a results endpoint for clients to poll for prediction outcomes.
3. **RabbitMQ (Message Broker):**
    - Queues and distributes tasks to available Celery workers.
    - Ensures reliable task delivery and load balancing.
4. **Celery Workers (Prediction Engines):**
    - Pick up tasks from the queue.
    - Execute predictions using the pre-trained ML model.
    - Store the prediction results in the Redis backend.
5. **Redis (Result Store):**
    - Acts as a fast, in-memory data store for prediction results.
    - Allows efficient retrieval of results based on the task ID.

### Example: Scenario of 100 Tasks Submitted

#### Client Submits Requests
The client (frontend) sends 100 POST requests to the `/predict` endpoint with the feature data for each prediction.

#### FastAPI Handles and Queues

1. **Validation and Task Creation:**
    - FastAPI validates each request's data in parallel (assuming you have multiple worker processes or threads configured).
    - For each valid request, it creates a Celery task and sends it to RabbitMQ.
    - FastAPI returns 100 unique task IDs to the client.

#### RabbitMQ Distributes Tasks
RabbitMQ efficiently distributes these 100 tasks across available Celery workers. If you have fewer than 100 workers, some tasks will wait in the queue until workers become available.

#### Celery Workers Process
Each worker picks up a task, loads the ML model (if not already in memory), performs the prediction, and stores the result in Redis under the corresponding task ID.

#### Client Polls for Results
The client can start polling the `/results/{task_id}` endpoint for each of the 100 task IDs. As results become available, they're returned to the client.

### Configuration Considerations for Scaling

#### Celery Worker Configuration
- **Concurrency:** Increase the number of worker processes or threads (`-c` or `--concurrency` option) to match or exceed the expected number of concurrent tasks.
- **Prefetch Multiplier:** Adjust the `worker_prefetch_multiplier` setting in your Celery config. A value of 1 means a worker will prefetch one task at a time. Increasing this can improve throughput if task execution time is short.
- **Rate Limiting:** If your model is computationally intensive, you might want to rate-limit the tasks at the worker level to prevent overwhelming your resources.

#### RabbitMQ Configuration
- **Queue Size:** Ensure your RabbitMQ queue settings can handle a large number of tasks. You might need to increase the queue length if you anticipate long bursts of requests.

#### Redis Configuration
- **Memory:** Ensure you have enough Redis memory to store the results of all concurrent tasks. Redis is in-memory, so plan accordingly.

#### FastAPI Configuration
- **Worker Processes/Threads:** Increase the number of worker processes (Gunicorn) or threads (Uvicorn) based on your hardware and the request validation complexity.

### Redis
- Redis is an in-memory data store.
- Uses key-value data structures.
- Traditionally used as a caching layer.
- Data stored in RAM (very fast).

### Further Reading
[Deploying ML Models in Production with FastAPI and Celery](https://towardsdatascience.com/deploying-ml-models-in-production-with-fastapi-and-celery-7063e539a5db)



# Celery Overview

## What is a Task Queue?

Task queues serve as a mechanism to distribute work across threads or machines. A task queue's fundamental unit of work is called a task. Dedicated worker processes continually monitor these task queues for new work to execute.

Celery communicates through messages, typically using a broker to mediate between clients and workers. When a client initiates a task, it adds a message to the queue, which the broker then delivers to an available worker.

A Celery system can scale horizontally and ensure high availability by employing multiple workers and brokers.

Celery is primarily implemented in Python, but the underlying protocol can be implemented in any programming language. Besides Python, there are also libraries like node-celery and node-celery-ts for Node.js, and a PHP client.

Language interoperability can be achieved by exposing an HTTP endpoint and creating tasks that request it (webhooks).

## Version Requirements

Celery version 5.3 is compatible with:

- Python versions 3.8, 3.9, 3.10, 3.11
- PyPy3.8 and newer (v7.3.11+)

Celery 4.x was the last version to support Python 2.7. Celery 5.x requires Python 3.6 or newer, with specific requirements for each minor version.

For older Python versions:

- Python 2.7 or Python 3.5: Use Celery series 4.4 or earlier.
- Python 2.6: Use Celery series 3.1 or earlier.
- Python 2.5: Use Celery series 3.0 or earlier.
- Python 2.4: Use Celery series 2.2 or earlier.

Note: Celery does not officially support Microsoft Windows due to minimal funding.

## Message Transport Requirements

Celery requires a message transport system to send and receive messages. While RabbitMQ and Redis are feature-complete broker transports, there are experimental solutions, including SQLite for local development.

Celery can operate on a single machine, across multiple machines, or even span across data centers.

## Getting Started

If you're new to Celery or upgrading from a previous version, start with these tutorials:

- [First Steps with Celery](https://docs.celeryproject.org/en/stable/getting-started/first-steps-with-celery.html)
- [Next Steps](https://docs.celeryproject.org/en/stable/getting-started/next-steps.html)

## Features of Celery

### Simple

Celery is straightforward to use and maintain, requiring minimal configuration.

### Highly Available

Workers and clients automatically retry tasks in case of connection loss or failures, and some brokers support high availability through replication strategies.

### Fast

A single Celery process can handle millions of tasks per minute with sub-millisecond round-trip latency, especially when using RabbitMQ, librabbitmq, and optimized settings.

### Flexible

Nearly every aspect of Celery is customizable and extensible, including pool implementations, serializers, compression schemes, logging, schedulers, consumers, producers, broker transports, and more.

## Example

Here's a simple Celery application:

```python
from celery import Celery

app = Celery('hello', broker='amqp://guest@localhost//')

@app.task
def hello():
    return 'hello world'
```


# Continuous Integration (CI):

## Focus
The core of CI is to ensure that code changes from multiple developers can be integrated into a shared repository frequently and smoothly.

## Process
- Developers regularly commit code changes to a version control system (like Git).
- Automated build processes compile the code and run unit tests to catch errors early.
- If any tests fail, the team is notified immediately so they can fix the issues.

## Benefits
- Faster feedback on code quality.
- Reduced risk of integration conflicts.
- Improved collaboration among developers.

# Continuous Delivery (CD):

## Focus
CD builds on CI by automating the release process up to the point where the software is ready for deployment to production.

## Process
- After passing CI, code changes are automatically deployed to a staging or testing environment.
- Additional tests, like integration tests and user acceptance tests, are performed in this environment.
- The software is packaged and prepared for production deployment, but the final deployment step is usually manual (e.g., requiring a button click or approval).

## Benefits
- Increased confidence in the release process.
- Ability to release software more frequently.
- Reduced manual effort in preparing releases.

# Continuous Deployment (CD):

## Focus
Continuous Deployment takes CD a step further by automating the entire release process, including the deployment to production.

## Process
- Follows the same steps as Continuous Delivery, but the final deployment to production is automated.
- Every change that passes the automated tests is immediately released to users.

## Benefits
- Fastest time to market for new features and bug fixes.
- Minimized risk associated with releases (smaller, more frequent changes).
- Increased customer satisfaction through rapid updates.

| Feature                 | Continuous Integration (CI) | Continuous Delivery (CD) | Continuous Deployment (CD) |
|-------------------------|------------------------------|--------------------------|----------------------------|
| Automation scope        | Build and unit testing       | Build, testing, deployment| Build, testing, release    |
| Deployment to prod      | No                           | Manual                   | Automatic                  |
| Release frequency       | High                         | High                     | Highest                    |
| Risk with each release  | Low                          | Low                      | Lowest                     |

## Which one to choose?
The choice depends on your team's maturity, risk tolerance, and business needs.

- **CI:** A great starting point for most teams to improve code quality and collaboration.
- **CD:** Suitable for teams that want faster release cycles and greater confidence in their releases.
- **Continuous Deployment:** Best for mature teams with high confidence in their automated tests and a culture of rapid iteration.
