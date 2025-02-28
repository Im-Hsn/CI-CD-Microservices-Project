## Application Architecture

Your application consists of:
- **Redis**: Database using RedisJSON
- **Inventory Service**: FastAPI backend on port 8000
- **Payment Service**: FastAPI backend on port 8001  
- **Frontend**: React application on port 3000

## Approach 1: Building and Running Containers Individually

### Step 1: Build Individual Images

```bash
# Navigate to Backend directory
cd Backend

# Build inventory service image
# -t: Tag the image with a name
docker build -t inventory-service -f Dockerfile.inventory .
# Explanation:
# - 'docker build': Command to build a Docker image
# - '-t inventory-service': Tag/name for the image
# - '-f Dockerfile.inventory': Specify which Dockerfile to use
# - '.': Use current directory as build context (source files)

# Build payment service image
docker build -t payment-service -f Dockerfile.payment .

# Navigate to Frontend directory
cd ../Frontend

# Build frontend service image
docker build -t frontend-service .
# Explanation:
# - No '-f' flag needed as we're using the default Dockerfile name
```

### Step 2: Run Individual Containers

```bash
# Run Redis container
# -d: Run in detached mode (background)
# -p: Map host port to container port
# --name: Assign a name to the container
docker run -d --name redis -p 6379:6379 redislabs/rejson:latest
# Explanation:
# - 'docker run': Command to create and start a container
# - '-d': Run in detached mode (in background)
# - '--name redis': Name the container 'redis'
# - '-p 6379:6379': Map host port 6379 to container port 6379
# - 'redislabs/rejson:latest': Image to use (pulled from Docker Hub)

# Run inventory service container
docker run -d --name inventory -p 8000:8000 \
  -e REDIS_HOST=localhost \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  inventory-service
# Explanation:
# - '-e REDIS_HOST=localhost': Set environment variable for Redis host
# - Note: This won't work properly as 'localhost' in container context
#   refers to the container itself, not the host machine

# Run payment service container
docker run -d --name payment -p 8001:8001 \
  -e REDIS_HOST=localhost \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  payment-service

# Run frontend container
docker run -d --name frontend -p 3000:3000 frontend-service
```

**Problems with this approach:**
- Containers can't easily communicate with each other
- Environment variables like REDIS_HOST=localhost don't work as expected
- No automatic dependency management (services requiring Redis start simultaneously)

## Approach 2: Building Images and Connecting with Docker Network

This approach solves the communication issues by creating a custom Docker network.

### Step 1: Build Individual Images (same as Approach 1)

```bash
# Build images as in Approach 1
cd Backend
docker build -t inventory-service -f Dockerfile.inventory .
docker build -t payment-service -f Dockerfile.payment .
cd ../Frontend
docker build -t frontend-service .
```

### Step 2: Create a Docker Network

```bash
# Create a custom bridge network
docker network create microservices-net
# Explanation:
# - 'docker network create': Command to create a new network
# - 'microservices-net': Name of the network
# - Default network type is 'bridge' which is suitable for containers
#   on the same Docker host
```

### Step 3: Run Containers on the Network

```bash
# Run Redis on the network
docker run -d --name redis --network microservices-net -p 6379:6379 redislabs/rejson:latest
# Explanation:
# - '--network microservices-net': Connect to our custom network
# - Other containers on the same network can now reach this container
#   using its name 'redis' as the hostname

# Run inventory service on the network
docker run -d --name inventory --network microservices-net -p 8000:8000 \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  inventory-service
# Explanation:
# - 'REDIS_HOST=redis': Now refers to the redis container by name
#   (Docker DNS resolution within the network)

# Run inventory consumer (for background tasks)
docker run -d --name inventory-consumer --network microservices-net \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  inventory-service python consumer.py
# Explanation:
# - Override the default command with 'python consumer.py'
# - This runs the consumer script instead of the API server

# Run payment service on the network
docker run -d --name payment --network microservices-net -p 8001:8001 \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  payment-service

# Run payment consumer
docker run -d --name payment-consumer --network microservices-net \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -e REDIS_PASSWORD= \
  payment-service python consumer.py

# Run frontend on the network
docker run -d --name frontend --network microservices-net -p 3000:3000 frontend-service


# OR we can use seperation of concerns logic for security reasons


# Create separate networks for different concerns
docker network create frontend-backend-net  # For frontend to talk to APIs
docker network create backend-db-net        # For backends to talk to Redis

# Run Redis only on the database network
docker run -d --name redis --network backend-db-net -p 6379:6379 redislabs/rejson:latest

# Run inventory service on both networks
docker run -d --name inventory --network backend-db-net -p 8000:8000 \
  -e REDIS_HOST=redis -e REDIS_PORT=6379 -e REDIS_PASSWORD= \
  inventory-service
docker network connect frontend-backend-net inventory

# Run payment service on both networks
docker run -d --name payment --network backend-db-net -p 8001:8001 \
  -e REDIS_HOST=redis -e REDIS_PORT=6379 -e REDIS_PASSWORD= \
  payment-service
docker network connect frontend-backend-net payment

# Run frontend only on the frontend network
docker run -d --name frontend --network frontend-backend-net -p 3000:3000 frontend-service
```

**Benefits of this approach:**
- Containers can communicate using service names (e.g., "redis", "inventory")
- Environment variables work correctly
- Custom network provides isolation from other Docker containers

**Limitations:**
- Still requires manual ordering of container startup
- Multi-step process that's error-prone
- No built-in dependency management

### Step 4: Managing Containers

```bash
# View all running containers
docker ps

# Stop a specific container
docker stop inventory

# Start a stopped container
docker start inventory

# View container logs
docker logs inventory

# Stop all containers
docker stop $(docker ps -q)

# Remove a container (must be stopped first)
docker rm inventory

# Remove all stopped containers
docker container prune
```

## Approach 3: Using Docker Compose

Docker Compose automates everything you did manually in Approach 2.

### Step 1: Create docker-compose.yml File

Your docker-compose.yml file defines all services, networks, and volumes in one place.

### Step 2: Build and Run with Docker Compose

```bash
# Build all services defined in docker-compose.yml
docker-compose build
# Explanation:
# - Reads the docker-compose.yml file
# - Builds any service with a 'build' directive
# - Services using existing images (like redis) are not built

# Start all services in the background
docker-compose up -d
# Explanation:
# - Creates a default network for the project
# - Pulls images if needed
# - Builds images if needed and not already built
# - Creates and starts containers in the right order
# - '-d' runs in detached mode (background)

# Combined build and start (what you've been using)
docker-compose up --build
# Explanation:
# - '--build': Forces rebuilding of images even if they exist
# - Useful during development when you're making changes
```

### Step 3: Managing Services with Docker Compose

```bash
# View running services
docker-compose ps

# View logs from all services
docker-compose logs

# View logs from specific service
docker-compose logs inventory

# Follow logs (continuous output)
docker-compose logs -f

# Stop services without removing containers
docker-compose stop
# Explanation:
# - Stops containers but preserves data and configuration
# - Containers can be restarted later

# Start stopped services
docker-compose start
# Explanation:
# - Restarts previously stopped containers
# - No rebuilding or recreation occurs

# Stop and remove containers, networks
docker-compose down
# Explanation:
# - Stops containers
# - Removes containers and networks
# - Does NOT remove volumes by default (data is preserved)

# Stop and remove everything including volumes
docker-compose down -v
# Explanation:
# - '-v' also removes volumes, deleting persistent data
```

## Comparison of Approaches

| Feature                     | Approach 1 | Approach 2 | Approach 3 |
|-----------------------------|------------|------------|------------|
| Inter-service communication  | ❌        | ✅        | ✅        |
| Dependency management        | ❌        | ❌        | ✅        |
| Network creation             | ❌        | Manual     | Automatic |
| Environment variables        | Limited    | Manual    | Declarative|
| Deployment complexity        | High       | Medium    | Low        |
| Commands required            | Many       | Many      | Few        |
| Learning value               | High       | High      | Medium     |
| Production suitability       | Low        | Medium    | High       |
























# Simple-Microservices-with-FastAPI-and-React
Simple Microservices App with FastAPI and React

Introduction
This project is a simple microservices app that is built using Python FastAPI for the backend and React for the frontend. The app utilizes RedisJSON as a database and uses Redis Streams for event dispatch.
Here we have two repositories. Frontend and Backend. The Frontend repo contains the code for the Frontend and the Backend repo contains the code for the Backend.

Prerequisites
Before starting this project, make sure that you have the following tools installed:

For Backend:- 
1. First we need to install python 3.10 
2. Install the packages listed in the requirements.txt file inside the Backend repo. To install the packages we can directly use the command  pip install -r            requirements.txt 
  
Setting up the Environment:-
Create a new Python virtual environment and activate it:
python3 -m venv venv
source venv/bin/activate

For Frontend:-
Create a new React project and install the required npm packages:
npx create-react-app my-app
cd my-app
npm install


Running the App:-
1. Start the Redis JSON and Redis Streams databases.
2. Start the FastAPI server by running the following command: uvicorn main:app --reload
3. Start the React app by running the following command: npm start

Features:-
1. User-friendly interface built with React.
2. Backend built with Python FastAPI, a modern and fast framework for building APIs.
3. Database powered by RedisJSON, a NoSQL database.
4. Event dispatch using Redis Streams, an event bus similar to RabbitMQ or Apache Kafka.

Conclusion:-
This project provides a simple but powerful way to build microservices apps with Python and React. The use of RedisJSON and Redis Streams allows for easy data storage and event dispatch, making it a great choice for microservices apps.
