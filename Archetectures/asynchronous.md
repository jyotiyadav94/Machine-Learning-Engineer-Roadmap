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
