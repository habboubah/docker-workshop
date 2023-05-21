# Docker Workshop: From Local Execution to Deployment

Docker is an open-source platform that automates the deployment, scaling, and management of applications. It does this by encapsulating applications into containers, which are lightweight and portable. Docker allows developers to package an application along with all its dependencies into a standardized unit for software development.

## Docker Containers and Images

**Docker Image** - A Docker image is a lightweight, stand-alone, executable package that includes everything needed to run a piece of software, including the code, a runtime, libraries, environment variables, and config files.

**Docker Container** - A Docker container is a runtime instance of a Docker image. You can think of a Docker image as a class and a Docker container as an instance of that class. Docker containers run the actual applications. A container includes an application and all of its dependencies. It shares the kernel with other containers and runs as an isolated process in user space on the host operating system.

## Docker Images and Layering

Docker uses a Union File System to achieve its lightweight images, which consists of layers stacked on top of each other. Each Docker image references a list of read-only layers that represent filesystem differences. When a container is created from an image, Docker adds a new, thin, writable layer on top of the existing layers of the image. 

Each layer is only a set of differences from the layer below it. The layers are stacked on top of each other. When you create a new container, you add a new writable layer on top of the underlying layers. This layer is often called the "container layer," and its changes go away when the container is deleted.

Docker provides a reliable, reproducible environment that is ideal for developing, testing, and deploying applications. Its containers serve to package software with all its dependencies, making it easy to run on any system that has Docker installed, regardless of the underlying infrastructure.

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

Sure, here's a brief explanation in Markdown format:

# Docker Volume Types

In Docker, a volume is used primarily for persisting data generated by and used by Docker containers. Docker volumes are designed to be easy to use, portable, and they are created and managed by Docker. There are three primary types of volumes in Docker:

## 1. **Anonymous Volumes**

Anonymous Volumes are automatically created by Docker when a container is started and a volume is referenced but not named. These volumes have a randomly generated name that isn't easy to refer to. Due to their nature, these volumes are typically not used unless necessary.

Example:
```bash
docker run -v /data nginx
```
In this case, a new volume will be created and mounted at `/data` in the container.

## 2. **Named Volumes**

Named Volumes are explicitly created by the user and have a specific name assigned by the user. This name makes it easy to maintain persistent data and share volumes across multiple containers.

Example:
```bash
docker volume create myvolume
docker run -v myvolume:/data nginx
```
In this case, a named volume called `myvolume` is created and then used by the nginx container.

## 3. **Bind Mounts**

Bind mounts are essentially mappings to existing directories or files on the host machine. Bind mounts rely on the host machine's filesystem structure and are not as portable as Docker volumes. They can, however, be useful in situations where you need to access specific files or directories on the host machine.

Example:
```bash
docker run -v /host/data:/container/data nginx
```
In this case, the directory `/host/data` on the host machine is mapped to `/container/data` inside the nginx container.

While Docker volumes provide a level of abstraction and ease-of-use, bind mounts offer more control for the user, as you are directly managing the location of the stored data. However, with this control comes additional complexity. It's generally recommended to use Docker volumes for applications in a production environment and bind mounts for development environments where the code is being actively updated.

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

Docker Compose is a tool for defining and managing multi-container Docker applications. With Docker Compose, you use a YAML file (`docker-compose.yml`) to configure your application's services, which greatly simplifies the process of deploying and managing Dockerized applications.

## What is Docker Compose?

In essence, Docker Compose allows you to manage the entirety of your application stack — including the web server, database, backend, and any other services — all within a single file. This means you can bring up or tear down an entire stack with just a single command.

## Why Use Docker Compose?

There are several reasons you might want to use Docker Compose:

1. **Simplicity**: Docker Compose makes it easy to manage complex multi-container applications. Instead of having to manage individual containers, you can define your entire stack in one file and manage it as a single entity.

2. **Reproducibility**: Because your entire stack (and all its configuration) is defined in a single file, it's easy to recreate your application stack on a different machine or environment.

3. **Isolation**: Docker Compose allows you to create separate environments for your different applications or stages of development (e.g., development, testing, production) on the same host.

## Docker Compose Commands

Here are some common Docker Compose commands:

- **`docker-compose up`**: This command starts and runs your entire app. Docker Compose starts and runs your entire app. `-d` flag is for detached mode, which means it runs your containers in the background.

    Example: `docker-compose up -d`

- **`docker-compose down`**: This command stops and removes containers, networks, images, and volumes defined in `docker-compose.yml`. It's a good way to clean up after you're done working with your application.

    Example: `docker-compose down`

- **`docker-compose ps`**: This command lists all running containers of the app defined in `docker-compose.yml`.

    Example: `docker-compose ps`

- **`docker-compose logs`**: This command shows the logs of your services. You can follow log output by using the `-f` flag.

    Example: `docker-compose logs -f`

- **`docker-compose build`**: This command builds all services in `docker-compose.yml`.

    Example: `docker-compose build`

In summary, Docker Compose is a powerful tool in the Docker ecosystem, which makes it easier to manage multi-container applications. It simplifies the orchestration of Docker containers for development environments, and it's a great tool to know when working with Docker.


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