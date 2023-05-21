# Docker Workshop: From Local Execution to Deployment

This comprehensive Docker workshop is designed to walk you through running and deploying an application composed of three main services:

1. A MongoDB database.
2. A Node.js backend application.
3. A React.js frontend application.

Our application is neatly divided into two folders, `frontend` and `backend`. We will be working primarily within Visual Studio Code's built-in terminal to execute Docker commands.

## Table of Contents

- [Local Execution](#local-execution)
- [Running Application Inside Docker Containers](#running-application-inside-docker-containers)
- [Working with Docker Networks](#working-with-docker-networks)
- [Data Persistence with Docker Volumes](#data-persistence-with-docker-volumes)
- [Docker Compose](#docker-compose)
- [Deployment](#deployment)
- [Clean Up](#clean-up)
- [Docker Commands Cheat Sheet](#docker-commands-cheat-sheet)

## Local Execution

Before we dive into Docker, let's start by running the application on our local environment.

### Step 1: Setting up MongoDB

We'll use Docker to run a MongoDB instance locally. This command pulls the MongoDB image from Docker Hub and runs it as a new container:

```bash
docker run \
    --name mongodb \
    --rm \
    -d \
    -p 27017:27017 \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=password \
    mongo
```

Check the status and logs of our MongoDB container:

```bash
docker ps
docker logs mongodb
```

### Step 2: Running the Backend Service

First, install dependencies:

```bash
npm install
```

Next, set up the necessary environment variables and update the MongoDB connection string in your `mongoose.connect()` method:

```bash
export MONGODB_USERNAME=admin
export MONGODB_PASSWORD=password
export URL=localhost
echo $URL
```

Your updated `mongoose.connect()` should look like this:

```javascript
mongoose.connect(`mongodb://${process.env.MONGODB_USERNAME}:${process.env.MONGODB_PASSWORD}@${process.env.URL}:27017/course-goals?authSource=admin`);
```

Start the application. You should see a "CONNECTED TO MONGODB!!" message:

```bash
node app.js
```

### Step 3: Running the Frontend Service

Install dependencies and start the application:

```bash
npm install
npm start
```

Navigate to `http://localhost:3000/` in your browser.

### Step 4: Testing the Application

To ensure that our application is running as expected, check the logs for the MongoDB container:

```bash
docker logs mongodb
```

You should see a "Connection accepted" message.

## Running Application Inside Docker Containers

We will now containerize our entire application. Note that the backend service will attempt to connect to the MongoDB container, so using `localhost:port_number` will not work as MongoDB is running inside a Docker container.

### Step 1: MongoDB Container

Run the MongoDB container just as we did before:

```bash
docker run \
    --name mongodb \
    --rm \
    -d \
    -p 27017:27017 \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=password \
    mongo
```

Check the status and logs of our MongoDB container:

```bash
docker ps
docker logs mongodb
```

### Step 2: Backend Container

First, we need to change the `URL` variable in the Dockerfile to `ENV "URL=host.docker.internal"`. 

Navigate to the `backend/` directory, build the Docker image, and run it

## Setting Up Frontend Container

Navigate to the `frontend` directory.

```bash
cd frontend/
```

Build the Docker image for the frontend. The tag `goals-frontend` is used for this image.

```bash
docker build -t goals-frontend .
```

The frontend requires interactive mode to run. Thus, we include the `-it` flag in the `docker run` command. Port 3000 is exposed for the frontend service. The name assigned to the container is `frontend`.

```bash
docker run --name frontend --rm -d -it -p 3000:3000 goals-frontend
```

Verify that the frontend container is running by listing the Docker processes.

```bash
docker ps 
```

You can check the logs of the frontend service by running:

```bash
docker logs frontend
```

Now, the frontend can be accessed on your local machine via the following URL:

```http
http://localhost:3000
```

After testing your applications, you should stop the running containers. You can stop the containers using the following commands:

```bash
docker stop backend
docker stop frontend
docker stop mongodb
```

# Docker Networking

In Docker, each container runs in its own network namespace. This provides a level of isolation where containers cannot communicate with each other unless you explicitly allow it. Here, we will be creating a network for our containers and add our containers to this network.

### Create Docker Network

Check the list of networks with the following command:

```bash
docker network ls
```

Create a new network named `goals`:

```bash
docker network create goals
```

Recheck the list of networks to confirm that the `goals` network has been created.

```bash
docker network ls
```

### Run MongoDB Container

```bash
docker run \
    --name mongodb \
    --rm \
    -d \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=password \
    --network goals \
    mongo
```

Check the list of running Docker processes:

```bash
docker ps
```

View the MongoDB logs:

```bash
docker logs mongodb
```

### Run Backend Container

In the backend's Dockerfile, set the URL environment variable to "mongodb".

```bash
cd backend/
docker build -t goals-backend .
docker run --name backend --rm -d --network goals -p 80:80 -e URL=mongodb goals-backend
```

Verify that the backend container is running.

```bash
docker ps 
docker logs backend
```

### Run Frontend Container

Navigate to the frontend directory:

```bash
cd frontend/
docker build -t goals-frontend .
docker run --name frontend --rm -d -it -p 3000:3000 goals-frontend
docker ps 
docker logs frontend
http://localhost:3000
```

# Docker Volumes

When Docker containers are removed, any changes made to the container's filesystem will be lost. By using Docker volumes, you can persist data across container shutdowns and restarts.

## MongoDB

Stop the MongoDB container.

```bash
docker stop mongodb
```

Create a new MongoDB container. Use the `-v` flag to add a named volume.

```bash
docker run \
    --name mongodb \
    --rm \
    -d \
    -e MONGO_INITDB_ROOT_USERNAME=admin \
    -e MONGO_INITDB_ROOT_PASSWORD=password \
    --network goals \
    -v data:/data/db \
    mongo
```

Verify that the MongoDB container is running.

```bash
docker ps
docker logs mongodb
```

## Backend

Stop the backend container.

```bash
docker stop
```markdown
backend
```

Now, we will create a new backend container. We don't need to create a volume for the backend, because it doesn't manage any persistent state.

```bash
cd backend/
docker build -t goals-backend .
docker run --name backend --rm -d --network goals -p 80:80 -e URL=mongodb goals-backend
```

Verify that the backend container is running.

```bash
docker ps
docker logs backend
```

## Frontend

For frontend, you don't need to persist data, so we will simply restart the container.

Stop the frontend container.

```bash
docker stop frontend
```

Navigate to the frontend directory:

```bash
cd frontend/
docker build -t goals-frontend .
docker run --name frontend --rm -d -it -p 3000:3000 goals-frontend
```

Verify that the frontend container is running.

```bash
docker ps
docker logs frontend
```

Now, you can access your application at `http://localhost:3000`. 

# Docker Compose

Docker Compose is a tool for defining and managing multi-container Docker applications. It uses a YAML file to configure the application’s services, and then, with a single command, you can create and start all the services.

Stop all the running containers.

```bash
docker stop frontend
docker stop backend
docker stop mongodb
```

Navigate back to the main directory:

```bash
cd ..
```

Create a file named `docker-compose.yml`.

```bash
touch docker-compose.yml
```

Edit the `docker-compose.yml` file. The file should have the following content:

```yaml
version: "3.8"
services:
  mongodb:
    image: 'mongo'
    container_name: mongodb
    volumes: 
      - data:/data/db
    environment: 
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
  backend:
    build: ./backend
    container_name: backend
    ports:
      - '80:80'
    volumes: 
      - logs:/app/logs
      - ./backend:/app
      - /app/node_modules
    environment: 
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      URL: mongodb
    depends_on:
      - mongodb
  frontend:
    container_name: frontend
    build: ./frontend
    ports: 
      - '3000:3000'
    volumes: 
      - ./frontend/src:/app/src
    stdin_open: true
    tty: true
    depends_on: 
      - backend

volumes: 
  data:
  logs:
```

Run the Docker Compose command to start the services.

```bash
docker-compose up -d
```

Verify that the services are running.

```bash
docker-compose ps
```

You can access your application at `http://localhost:3000`. 

To stop the services, run:

```bash
docker-compose down
```

With Docker Compose, you can manage your services very easily. You can start, stop and rebuild your services with a single command.
```
Please note that Docker Compose is a very powerful tool. This is a basic example of how to use Docker Compose to manage multi-container applications. There are many more features and options that you can use in your `docker-compose.yml` file, such as configuring networks, setting up dependencies between services, etc. For more information, refer to the official Docker Compose documentation.
```

# Deploying Containers

Once you have your containers running locally, the next step is to deploy them. For this, we will use Docker Hub, a cloud-based registry service which allows you to link to code repositories, build your images and test them, stores manually pushed images, and links to Docker Cloud so you can deploy images to your hosts.

First, create an account on [Docker Hub](https://hub.docker.com/). Once you have an account, navigate to the Repositories tab and create a new repository for "goals-backend" and "goals-frontend".

## Docker Images and Docker Hub

Let's build the backend and frontend Docker images and push them to Docker Hub.

### Backend

```bash
cd backend/
docker build -t goals-backend .
docker images
docker tag goals-backend <your_dockerhub_username>/goals-backend
docker images
docker login -u <your_dockerhub_username>
docker push <your_dockerhub_username>/goals-backend
```

### Frontend

```bash
cd frontend/
docker build -t goals-frontend .
docker images
docker tag goals-frontend <your_dockerhub_username>/goals-frontend
docker images
docker login -u <your_dockerhub_username>
docker push <your_dockerhub_username>/goals-frontend
```

## Running the Deployed Containers

To test the containers that were pushed to Docker Hub:

```bash
docker run \
-d \
--rm \
--name backend \
-p 80:80 \
-e URL=mongodb \
--network goals \
<your_dockerhub_username>/goals-backend
```

```bash
docker run \
-d \
-it \
--rm \
--name frontend \
-p 3000:3000 \
--network goals \
<your_dockerhub_username>/goals-frontend
```

## Docker Compose with Docker Hub

You can also use Docker Compose with the images that were pushed to Docker Hub. Create a `docker-compose-dockerhub.yaml` file.

Change  "build: ./backend" with "image: '<your_dockerhub_username>/goals-backend'" and "build: ./frontend" with "image: '<your_dockerhub_username>/goals-frontend'" in the `docker-compose-dockerhub.yaml` file.

Then run Docker Compose:

```bash
docker-compose -f docker-compose-dockerhub.yaml up
```

Access the application at `http://localhost:3000`.

To stop the services, run:

```bash
docker-compose -f docker-compose-dockerhub.yaml down
```

# Cleaning Up

To remove all the Docker resources:

```bash
docker stop $(docker ps -aq) && docker system prune -af --volumes
```

# Docker Cheat Sheet

Here is a basic Docker command cheat sheet:

* `docker run <image>`: run a container
* `docker ps`: list running containers
* `docker stop <name>`: stop a container
* `docker logs <name>`: show logs from a container
* `docker rm <name>`: remove a container
* `docker images`: list Docker images
* `docker rmi <image>`: remove Docker image
* `docker build`: build a Docker image
* `docker tag <image> <tag>`: tag a Docker image
* `docker push <image>`: push a Docker image to a registry
* `docker login`: log in to a Docker registry
* `docker-compose up`: start all services in `docker-compose.yml`
* `docker-compose down`: stop all services in `docker-compose.yml`
* `docker-compose ps`: list all services in `docker-compose.yml`

That's all for this workshop. I hope it was helpful, and good luck