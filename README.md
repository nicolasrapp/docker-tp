# devops-livecoding

## 1 Docker 

### 1-1 Document your database container essentials: commands and Dockerfile.

```docker
# The base image upon which we build :
FROM postgres:14.1-alpine 

# We define our database environment with the username, password, and the name of the database:
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

```

We build the image: 

```bash
docker build -t nicorapp/database .
```

A network is created to allow our database to communicate with Adminer:

```bash
docker network create app-network
```

We run adminer:

```bash
docker run \
    -p "8090:8080" \
    --net=app-network \
    --name=adminer \
    -d \
    adminer
```

Then we run our image:

```bash
docker run -p 8888:5000 --name database  --network app-network nicorapp/database
```

To add our SQL tables and data, we copy our SQL files to our container:
```docker
FROM postgres:14.1-alpine

COPY /initdb /docker-entrypoint-initdb.d

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```

We rebuild our image and then rerun it. 

Finally, to preserve our data, we add a volume:

```bash
docker run -p 8888:5000 --name database  --network app-network -v /initdb:/var/lib/postgresql/data nicorapp/database
```



### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

A multistage build is needed in order to optimize Docker images by separating the build environment from
the final production image. It also reduces the size of the final image and eliminates unnecessary build
dependencies.

```bash

# Build

# Use the maven:3.8.6-amazoncorretto-17 image as the build stage and name it myapp-build.
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

# Set an environment variable MYAPP_HOME to /opt/myapp.
ENV MYAPP_HOME /opt/myapp

# Set the working directory in the container to MYAPP_HOME.
WORKDIR $MYAPP_HOME

# Copy the Maven Project Object Model (POM) file (pom.xml) from the host to the container.
COPY pom.xml .

# Copy the source code from the host to the container in the src directory.
COPY src ./src

# Run the Maven build to package the application, skipping the tests.
RUN mvn package -DskipTests

# Run

# Switch to a new stage using the amazoncorretto:17 base image.
FROM amazoncorretto:17

# Set the same MYAPP_HOME environment variable as in the build stage.
ENV MYAPP_HOME /opt/myapp

# Set the working directory in the container to MYAPP_HOME.
WORKDIR $MYAPP_HOME

# Copy the JAR file(s) from the myapp-build stage to the MYAPP_HOME directory in this stage.
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Define the entry point for the container, which will run the Java application with the JAR file.
ENTRYPOINT java -jar myapp.jar


```

### 1-3 Document docker-compose most important commands. 

Start the containers defined in the docker-compose.yml file:
```bash
docker compose up
```
Options include:

-d to start all the containers as detached,
--build to make sure all the image that have been built for the orchestration are rebuilt,
--force-recreate to force Compose to stop and recreate all containers

Stop and remove the containers defined in the docker-compose.yml file:
```bash
docker compose down
```

### 1-4 Document your docker-compose file.

```yaml
version: '3.7'  # Define the Docker Compose file version.

services:  # Define the services (containers) you want to run.

    backend:  # Define a service named "backend."
        build:  # Build the Docker image for this service.
          context: ./simple-api  # Specify the build context directory for the Dockerfile.
          dockerfile: Dockerfile  # Specify the Dockerfile to use for building the image.
        container_name: backendtp  # Set the name of the container.
        networks:  # Connect the service to a specific network.
        - app-network  # Use the "app-network" network.
        depends_on:  # Specify that this service depends on another service.
        - database  # This service depends on the "database" service.

    database:  # Define a service named "database."
        build:  # Build the Docker image for this service.
          context: .  # Specify the build context directory for the Dockerfile (current directory).
          dockerfile: Dockerfile  # Specify the Dockerfile to use for building the image.
        container_name: database  # Set the name of the container.
        networks:  # Connect the service to a specific network.
        - app-network  # Use the "app-network" network.

    httpd:  # Define a service named "httpd."
        build:  # Build the Docker image for this service.
          context: ./httpserver  # Specify the build context directory for the Dockerfile.
          dockerfile: Dockerfile  # Specify the Dockerfile to use for building the image.
        container_name: fronttp  # Set the name of the container.
        ports:  # Expose and map container ports to host ports.
        - "80:80"  # Map host port 80 to container port 80.
        networks:  # Connect the service to a specific network.
        - app-network  # Use the "app-network" network.
        depends_on:  # Specify that this service depends on another service.
        - backend  # This service depends on the "backend" service.

    front:  # Define a service named "front."
        build:  # Build the Docker image for this service.
          context: ./front  # Specify the build context directory for the Dockerfile.
          dockerfile: Dockerfile  # Specify the Dockerfile to use for building the image.
        container_name: front  # Set the name of the container.
        ports:  # Expose and map container ports to host ports.
        - "8081:80"  # Map host port 8081 to container port 80.
        networks:  # Connect the service to a specific network.
        - app-network  # Use the "app-network" network.
        depends_on:  # Specify that this service depends on another service.
        - httpd  # This service depends on the "httpd" service.

networks:  # Define the networks used by the services.

    app-network:  # Define a network named "app-network."
```


### 1-5 Document your publication commands and published images in dockerhub.

We will need a Docker Hub account.

Log in to the Docker Hub account:
```bash
docker login
```

Tag the image:
```bash
docker tag nicorapp/backendtp nicorapp/backendtp:1.0
```

Push the image to dockerhub:
```bash
docker push nicorapp/backendtp:1.0 
```

 ## 2 Github Actions

 
### 2-1 What are testcontainers?

Testcontainers are Java libraries that allow running multiple Docker containers during testing.

### 2-2 Document your Github Actions configurations.

Creation of the folder .github/workflows 

and of the file main.yml:

```yaml
#This sets the name of the GitHub Actions workflow to "CI devops 2023."
name: CI devops 2023
# The workflow is triggered on two events:
on:
# When there are pushes to the "main" branch.
  push:
    branches: 
      - main
# When pull requests are opened or updated.
  pull_request:

# Defines a single job named "test-backend":
jobs:
  test-backend:
    # Specifies that this job should run on an Ubuntu 22.04 virtual machine:
    runs-on: ubuntu-22.04
    # Lists the individual steps that should be executed within this job:
    steps:
     # The first step uses the actions/checkout action to check out the GitHub repository code (Version 2.5.0 of this action is used):
      - uses: actions/checkout@v2.5.0
     # The second step sets up JDK 17 for the job using the actions/setup-java action:
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        # Specifies that the desired Java version is 17, and the distribution to use is "adopt."
        with:
          java-version: '17'
          distribution: 'adopt'

      # The third step, named "Build and test with Maven": 
      - name: Build and test with Maven
        # Executed within the "simple-api" directory:
        working-directory: simple-api
        # Runs the command mvn clean verify to build and test the Java application using Maven:
        run: mvn clean verify
      
      # define job to build and publish docker image
        build-and-push-docker-image:
         needs: test-backend
         # run only when code is compiling and tests are passing
         runs-on: ubuntu-22.04
        
      # Steps to perform in job
      steps:

        # Log in to the Docker Hub account:
        - name: Login to DockerHub
          run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
   
        - name: Checkout code
          uses: actions/checkout@v2.5.0
     
        - name: Build image and push backend
          uses: docker/build-push-action@v3
          with:
            # relative path to the place where source code with Dockerfile is located
            context: ./simple-api
            # Tag the image:
            tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
            # Push the image to dockerhub:
            push: ${{ github.ref == 'refs/heads/main' }}
     
        - name: Build image and push database
          uses: docker/build-push-action@v3
          with:
           context: .
           tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-database:latest
           push: ${{ github.ref == 'refs/heads/main' }}
     
        - name: Build image and push httpd
          uses: docker/build-push-action@v3
          with:
            context: ./httpserver
            tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-httpserver:latest
            push: ${{ github.ref == 'refs/heads/main' }}
   
        - name: Build image and push front
          uses: docker/build-push-action@v3
          with:
            context: ./front
            tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-front:latest
            push: ${{ github.ref == 'refs/heads/main' }}
```


### 2-3 Document your quality gate configuration.

First we have to create a SonarCloud account and generate an access token with appropriate permissions for integration


We can update the main.yml and replace the mvn clean verify:
```yaml

      - name: Build and test with Maven
        working-directory: simple-api
        # This run will build the project and send the analysis results to SonarCloud:
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=lucielong_tpDevops -Dsonar.organization=lucielong -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```



Dsonar.projectKey: This is the key of the SonarCloud project.

Dsonar.organization: This is the SonarCloud organization. 

Dsonar.host.url: This is the URL of the SonarCloud service.

Dsonar.token: This is a reference to a GitHub secret called "SONAR_TOKEN," which is used for authentication when pushing the analysis results to SonarCloud and is stored as a secret in GitHub.



## 3 Ansible

### 3-1 Document your inventory and base commands

```yaml

# Define variables that apply to all hosts and groups in this inventory:
all:
  vars:
    # Set the SSH user for all hosts to 'centos':
    ansible_user: centos

    # Set the path to the SSH private key file to '../../rsa/id_rsa':
    ansible_ssh_private_key_file: ../../rsa/id_rsa

# Define host groups and their associated hosts:
children:
  # The 'prod' group contains production hosts:
  prod:
    hosts:
      # Define a single host 'lucie.long.takima.cloud' in the 'prod' group:
      lucie.long.takima.cloud:

```

We can test the inventory with the ping command:

```bash
ansible all -i inventories/setup.yml -m ping
```


We can request the server to get the OS distribution, thanks to the setup module:
```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

We installed Apache httpd server on your machine, we can remove it:
```bash
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```



### 3-2 Document your playbook

```yaml

# Define the target hosts for the playbook:
- hosts: all

  # Disable gathering facts to improve playbook execution speed. Facts are information about the target hosts and are not needed in this playbook.
  gather_facts: false

  # Use privilege escalation (sudo) to become the superuser (root) on the target hosts, if required. This is often necessary for tasks like installing packages.
  become: yes

  # Define a list of roles to be executed in this playbook. Roles are sets of tasks and variables organized into a reusable component.
  roles:
    - docker
    - network
    - database
    - app
    - proxy
    - front

```
 We created each role and put its tasks there.

 Here is an exemple for the Docker role:
```bash
ansible-galaxy init roles/docker
```
Initialized role has a couple of directories, we only keep the one we will need:
 * tasks - contains the main list of tasks to be executed by the role.
 * handlers - contains handlers, which may be used by this role or outside.

In the main.yml of the tasks folder, we put tasks to install docker on the server:

```yaml
 # This is a task named "Clean packages." It runs the "yum clean" command to remove cached package data.
- name: Clean packages
  command:
    cmd: yum clean -y packages

# This task is named "Install device-mapper-persistent-data." It uses the "yum" module to ensure that the "device-mapper-persistent-data" package is installed and up to date.
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

# This task is named "Install lvm2." It uses the "yum" module to ensure that the "lvm2" package is installed and up to date.
- name: Install lvm2
  yum:
    name: lvm2
    state: latest

# This task is named "add repo docker." It runs the "yum-config-manager" command to add the Docker repository.
- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

# This task is named "Install Docker." It uses the "yum" module to ensure that the "docker-ce" package is installed and in the "present" state.
- name: Install Docker
  yum:
    name: docker-ce
    state: present

# This task is named "Make sure Docker is running." It uses the "service" module to start the Docker service and tags it with "docker."
- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```


### 3-3 Document your docker_container tasks configuration.

We already document the docker installation.

For the creation of a network:

```yaml
# Define the task name: Create a network
- name: Create a network

  # Use the community.docker.docker_network module to create a Docker network.
  community.docker.docker_network:

    # Specify the name of the Docker network we want to create.
    name: app-network
```

For launching the database: 

```yaml
# Define the task name: Launch Database
- name: Launch Database

  # Use the docker_container module to create a Docker container.
  docker_container:

    # Specify the name of the Docker container.
    name: my-db

    # Specify the Docker image to use for the container.
    image: nicorapp/database:latest

    # Define the networks that the container should be connected to.
    networks:
      - name: app-network

    # Set environment variables for the container.
    env:
      POSTGRES_DB: "db"
      POSTGRES_USER: "user"
      POSTGRES_PASSWORD: "pwd"
```
For launching the app: 

```yaml
# Define the task name: Launch App
- name: Launch App

  # Use the docker_container module to create a Docker container.
  docker_container:

    # Specify the name of the Docker container.
    name: my-api

    # Specify the Docker image to use for the container.
    image: nicorapp/simple-api:latest

    # Define the networks that the container should be connected to.
    networks:
      - name: app-network
```

For launching the proxy:

```yaml
# Define the task name: Launch Proxy
- name: Launch Proxy

  # Use the docker_container module to create a Docker container.
  docker_container:

    # Specify the name of the Docker container.
    name: fronttp

    # Specify the Docker image to use for the container.
    image: nicorapp/http-server:latest

    # Define the networks that the container should be connected to.
    networks:
      - name: app-network

    # Expose and map container ports to host ports.
    ports:
      - 80:80
```
