# **Migration to the Cloud with Containerization (Docker & Docker Compose) - Project 101**

## **📋 Project Overview**

This project demonstrates the complete migration of the Tooling web application from a VM-based solution to a containerized one using Docker. You'll learn to run MySQL in a container, containerize the PHP application, and orchestrate multi-container applications with Docker Compose. This guide provides step-by-step instructions with comprehensive explanations and screenshots.

Your repositories:

- **Tooling App**: [https://github.com/Munyat/tooling-2](https://github.com/Munyat/tooling-2)
- **PHP-Todo App**: [https://github.com/Munyat/tooling3](https://github.com/Munyat/tooling3)

---

## **🎯 Why Migrate from VMs to Containers?**

Before diving into the technical steps, it's crucial to understand why containerization revolutionizes application deployment:

| Challenge with VMs                       | Solution with Containers          |
| ---------------------------------------- | --------------------------------- |
| Each VM requires full OS                 | Containers share host OS kernel   |
| Slow to start (minutes)                  | Instant startup (seconds)         |
| Resource heavy (GBs)                     | Lightweight (MBs)                 |
| Configuration drift between environments | Consistent environment everywhere |
| Dependency conflicts between apps        | Isolated environments per app     |
| "It works on my machine" problem         | Same environment everywhere       |

> **"It works on my machine!"** – This problem disappears with Docker, as containers package the application with its entire environment, ensuring it runs identically on any system with Docker engine.

---

## **🏗️ Architecture Overview**

```
           +--------------------------+
           |  Docker Network:         |
           |  tooling_app_network     |
           +-----------+--------------+
                       |
         +-------------+-------------+
         |                           |
+--------------------+       +--------------------+
| Container:         |       | Container:         |
| tooling-app        |       | musing_swanson     |
| (PHP / Apache)     |       | (MySQL Server)     |
|--------------------|       |--------------------|
| Connects to MySQL  |       | Runs MySQL Server  |
| Hostname:          |       | Hostname:          |
| musing_swanson     |       | mysqlserverhost    |
| Port: 80 inside    |       | Port: 3306         |
| Mapped: 8085 host  |       | Exposed: 3306/33060|
+--------------------+       +--------------------+
```

**Key Components:**

| Component                | Description              | Purpose                                |
| ------------------------ | ------------------------ | -------------------------------------- |
| **Docker Network**       | Isolated virtual network | Enables secure container communication |
| **MySQL Container**      | Database server          | Stores application data                |
| **PHP/Apache Container** | Web server               | Serves the Tooling application         |
| **Port Mapping**         | Host:Container mapping   | Exposes container ports to host        |

---

## **Part 1: MySQL in Container**

### **Step 1: Pull MySQL Docker Image**

Start by pulling the MySQL image from Docker Hub:

```bash
docker pull mysql/mysql-server:latest
```

**What this does**: Downloads the MySQL server image from Docker Hub registry to your local machine.

**Screenshot**: ![mysql_container_intsalled_and_working.png](screenshots/mysql_container_intsalled_and_working.png)
_Figure 1: MySQL container successfully installed and running_

### **Step 2: List Downloaded Images**

```bash
docker images
```

This command shows all Docker images stored locally, including the newly pulled MySQL image.

### **Step 3: Deploy MySQL Container**

Run the MySQL container with a root password:

```bash
docker run --name mysql-server -e MYSQL_ROOT_PASSWORD=admin123 -d mysql/mysql-server:latest
```

**Flag explanations:**

- `--name mysql-server`: Assigns a friendly name to the container
- `-e MYSQL_ROOT_PASSWORD=admin123`: Sets environment variable for root password
- `-d`: Runs container in detached mode (background)
- `mysql/mysql-server:latest`: The image to use

### **Step 4: Verify Container is Running**

```bash
docker ps -a
```

**Screenshot**: ![docker_ps_command_showing_containers.png](screenshots/docker_ps_command_showing_containers.png)
_Figure 2: Docker ps command showing running containers_

Expected output:

```
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS                             PORTS                       NAMES
7141da183562   mysql/mysql-server:latest            "/entrypoint.sh mysq…"   12 seconds ago   Up 11 seconds (health: starting)   3306/tcp, 33060-33061/tcp   mysql-server
```

### **Step 5: Connect to MySQL Container Directly**

```bash
docker exec -it mysql-server mysql -uroot -p
# Enter password: admin123 when prompted
```

**What this does**: Executes the MySQL client inside the running container, allowing direct database access.

### **Step 6: Create a Custom Network**

```bash
docker network create --subnet=172.18.0.0/24 tooling_app_network
docker network ls
```

**Why create a custom network?** Containers on the same network can communicate using container names as hostnames, providing service discovery.

**Screenshot**: ![docker_network_command.png](screenshots/docker_network_command.png)
_Figure 3: Creating a custom Docker network for container communication_

### **Step 7: Run MySQL Container on Custom Network**

```bash
export MYSQL_PW=admin123
docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW -d mysql/mysql-server:latest
```

**Flag explanations:**

- `--network tooling_app_network`: Connects container to custom network
- `-h mysqlserverhost`: Sets container hostname for easier reference

### **Step 8: Create Database User**

Create a file named `create_user.sql`:

```sql
CREATE USER 'tooling_user'@'%' IDENTIFIED BY 'tooling_pass';
GRANT ALL PRIVILEGES ON * . * TO 'tooling_user'@'%';
```

Run the script:

```bash
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < create_user.sql
```

**Screenshot**: ![docker_mysql_executing_create_user.sql.png](screenshots/docker_mysql_executing_create_user.sql.png)
_Figure 4: Executing create_user.sql to create database user_

### **Step 9: Connect Using MySQL Client Container**

```bash
docker run --network tooling_app_network --name mysql-client -it --rm mysql mysql -h mysqlserverhost -u tooling_user -p
```

**Flag explanations:**

- `--rm`: Automatically removes container when it exits
- `-it`: Interactive mode with pseudo-TTY

### **Step 10: Load Database Schema**

Create `tooling-db.sql`:

```sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (username, password, email) VALUES
('test', '12345', 'test@gmail.com'),
('demo', 'password123', 'demo@example.com');
```

Load the schema:

```bash
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < tooling-db.sql
```

**Screenshot**: ![mysql_created_tables.png](screenshots/mysql_created_tables.png)
_Figure 5: MySQL tables successfully created_

---

## **Part 2: Containerize Tooling Application**

### **Step 1: Clone Tooling Application Repository**

```bash
git clone https://github.com/Munyat/tooling-2.git tooling-02
cd tooling-02
```

Your repository structure:

```
tooling-02/
├── Dockerfile
├── Jenkinsfile
├── Jenkinsfile-Sample
├── Jenkinsfile-sample2
├── Jenkinsfile_tmp
├── README.md
├── apache-config.conf
├── default.conf
├── html/
├── start.sh
└── supervisord.conf
```

### **Step 2: Examine the Dockerfile**

View your **[Dockerfile](https://github.com/Munyat/tooling-2/blob/main/Dockerfile)**:

```dockerfile
FROM php:8-apache

ENV MYSQL_IP=$MYSQL_IP
ENV MYSQL_USER=$MYSQL_USER
ENV MYSQL_PASS=$MYSQL_PASS
ENV MYSQL_DBNAME=$MYSQL_DBNAME
EXPOSE 80

RUN <<-EOF
 docker-php-ext-install mysqli
 echo "ServerName localhost" >> /etc/apache2/apache2.conf
 curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
EOF

COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
RUN a2enmod rewrite

# Copy application source
COPY html /var/www/html
RUN chown -R www-data:www-data /var/www/html

CMD ["apache2-foreground"]
```

**Dockerfile Breakdown:**

| Instruction               | Purpose                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `FROM php:8-apache`       | Base image with PHP and Apache pre-installed                                        |
| `ENV`                     | Sets environment variables for database connection                                  |
| `RUN`                     | Executes commands during build: installs mysqli, sets ServerName, installs Composer |
| `COPY apache-config.conf` | Copies custom Apache configuration                                                  |
| `RUN a2enmod rewrite`     | Enables Apache rewrite module                                                       |
| `COPY html /var/www/html` | Copies application code to web root                                                 |
| `RUN chown`               | Sets proper file permissions                                                        |
| `CMD`                     | Defines the command to run when container starts                                    |

### **Step 3: Build Docker Image**

```bash
docker build -t tooling:0.0.1 .
```

**What this does**: Builds a Docker image using the Dockerfile in the current directory (`.`), tagging it as `tooling:0.0.1`.

### **Step 4: Run the Tooling Container**

```bash
docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1
```

**Flag explanations:**

- `--network tooling_app_network`: Connects to same network as MySQL
- `-p 8085:80`: Maps host port 8085 to container port 80
- `-it`: Interactive mode to see logs

### **Step 5: Fix Apache ServerName Error**

If you see the error:

```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name...
```

Create a fix:

```bash
echo "ServerName localhost" > apache-config.conf
docker cp apache-config.conf tooling:/etc/apache2/conf-available/servername.conf
docker exec tooling a2enconf servername
docker restart tooling
```

### **Step 6: Update Database Connection**

Edit `html/functions.php` to use the correct database hostname:

```php
$servername = "mysqlserverhost";  // Use the hostname set in MySQL container
$username = "tooling_user";
$password = "tooling_pass";
$dbname = "toolingdb";
```

### **Step 7: Access the Application**

Open browser: `http://localhost:8085`

**Screenshot**: ![localHoast_5000_login_page.png](screenshots/localHoast_5000_login_page.png)
_Figure 6: Tooling application login page at localhost:5000_

Login with:

- Email: `test@gmail.com`
- Password: `12345`

**Screenshot**: ![toolong_dashboard.png](screenshots/toolong_dashboard.png)
_Figure 7: Tooling application dashboard after successful login_

---

## **Part 3: Docker Compose for Multi-Container Orchestration**

### **Why Docker Compose?**

Running multiple containers with individual `docker run` commands becomes cumbersome. Docker Compose allows you to define all services in a YAML file and start everything with one command.

### **Step 1: Create Docker Compose File**

Create `tooling.yaml` based on your **[tooling.yaml](https://github.com/Munyat/tooling3/blob/main/tooling.yaml)**:

```yaml
version: "3.9"

services:
  tooling_frontend:
    build: .
    container_name: tooling-app
    ports:
      - "5000:80"
    volumes:
      - tooling_frontend:/var/www/html
    networks:
      - tooling_network
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DB_HOST=db
      - DB_USER=tooling_user
      - DB_PASSWORD=tooling_pass
      - DB_NAME=toolingdb
    restart: always

  db:
    image: mysql:5.7
    container_name: tooling-db
    restart: always
    environment:
      MYSQL_DATABASE: toolingdb
      MYSQL_USER: tooling_user
      MYSQL_PASSWORD: tooling_pass
      MYSQL_ROOT_PASSWORD: rootpassword
    volumes:
      - db_data:/var/lib/mysql
      - ./tooling-db.sql:/docker-entrypoint-initdb.d/schema.sql
    networks:
      - tooling_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  tooling_frontend:
  db_data:

networks:
  tooling_network:
    driver: bridge
```

**Docker Compose Fields Explained:**

| Field            | Purpose                       | Example                               |
| ---------------- | ----------------------------- | ------------------------------------- |
| `version`        | Docker Compose API version    | `"3.9"`                               |
| `services`       | Defines the containers        | `tooling_frontend:`, `db:`            |
| `build`          | Build context for Dockerfile  | `build: .`                            |
| `image`          | Docker image to use           | `image: mysql:5.7`                    |
| `container_name` | Custom container name         | `container_name: tooling-app`         |
| `ports`          | Port mapping (host:container) | `- "5000:80"`                         |
| `volumes`        | Persistent data storage       | `- db_data:/var/lib/mysql`            |
| `networks`       | Container network             | `- tooling_network`                   |
| `depends_on`     | Service startup order         | `depends_on: - db`                    |
| `environment`    | Environment variables         | `- DB_HOST=db`                        |
| `restart`        | Restart policy                | `restart: always`                     |
| `healthcheck`    | Service health verification   | `test: ["CMD", "mysqladmin", "ping"]` |

### **Step 2: Start with Docker Compose**

```bash
docker-compose -f tooling.yaml up -d
docker-compose -f tooling.yaml ps
```

**Screenshot**: ![docker_compose_ps_all_containers_running.png](screenshots/docker_compose_ps_all_containers_running.png)
_Figure 8: Docker Compose showing all containers running_

Access the app: `http://localhost:5000`

---

## **Practice Task 1: Push Images to Docker Hub**

### **Why Push to Docker Hub?**

Docker Hub serves as a centralized registry to store and share Docker images, making them accessible from any system for deployment.

### **Step 1: Login to Docker Hub**

```bash
docker login
```

**Screenshot**: ![docker_login.png](screenshots/docker_login.png)
_Figure 9: Docker Hub login successful_

### **Step 2: Create Docker Hub Repository**

**Screenshot**: ![docker_repo_created.png](screenshots/docker_repo_created.png)
_Figure 10: Docker Hub repository created_

**Screenshot**: ![all_repositories_dockerhub_dashoard.png](screenshots/all_repositories_dockerhub_dashoard.png)
_Figure 11: All repositories in Docker Hub dashboard_

### **Step 3: Tag and Push Images**

```bash
# Tag the image with your Docker Hub username and repository
docker tag tooling:0.0.1 yourusername/tooling-frontend:0.0.1

# Push to Docker Hub
docker push yourusername/tooling-frontend:0.0.1
```

**Screenshot**: ![docker_image_tagging_for_later_push.png](screenshots/docker_image_tagging_for_later_push.png)
_Figure 12: Docker image tagging for later push_

**Screenshot**: ![docker_pushed_images.png](screenshots/docker_pushed_images.png)
_Figure 13: Docker images successfully pushed_

**Screenshot**: ![after_pushing_image_dockerhub_image_showing updated_version.png](screenshots/after_pushing_image_dockerhub_image_showing%20updated_version.png)
_Figure 14: Docker Hub showing updated image version_

---

## **Practice Task 2: CI/CD Pipeline with Jenkins**

### **Why CI/CD for Docker?**

Continuous Integration ensures that every code change is automatically built, tested, and packaged as a Docker image, maintaining quality and consistency.

### **Step 1: Jenkinsfile with Test Stage**

Your **[Jenkinsfile](https://github.com/Munyat/tooling3/blob/main/Jenkinsfile)** implements a complete CI/CD pipeline:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub')
        IMAGE_NAME = 'yourusername/tooling-frontend'
        GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    BRANCH_NAME = env.BRANCH_NAME ?: 'main'
                    IMAGE_TAG = "${BRANCH_NAME}-${GIT_COMMIT_SHORT}"

                    sh """
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Integration Test') {
            steps {
                sh '''
                    echo "🧪 Running Integration Tests..."

                    # Create test network
                    docker network create test-network || true

                    # Start MySQL test container
                    docker run -d --network test-network --name test-mysql \
                      -e MYSQL_ROOT_PASSWORD=testpass \
                      -e MYSQL_DATABASE=testdb \
                      -e MYSQL_USER=testuser \
                      -e MYSQL_PASSWORD=testpass \
                      mysql:5.7

                    # Wait for MySQL to be ready
                    sleep 20

                    # Run the app container
                    docker run -d --network test-network --name test-app \
                      -p 8085:80 ${IMAGE_NAME}:latest

                    # Wait for app to start
                    sleep 10

                    # Test HTTP endpoint - should return 200
                    echo "Testing application endpoint..."
                    STATUS_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8085)

                    if [ "$STATUS_CODE" -eq 200 ] || [ "$STATUS_CODE" -eq 302 ]; then
                        echo "✅ Test passed! HTTP Status: $STATUS_CODE"
                    else
                        echo "❌ Test failed! HTTP Status: $STATUS_CODE"
                        exit 1
                    fi
                '''
            }
        }

        stage('Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin
                        docker push ${IMAGE_NAME}:${GIT_COMMIT_SHORT}
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up test containers and images
            sh '''
                echo "🧹 Cleaning up test containers and images..."

                # Stop and remove test containers
                docker stop test-app test-mysql 2>/dev/null || true
                docker rm test-app test-mysql 2>/dev/null || true

                # Remove test network
                docker network rm test-network 2>/dev/null || true

                # Remove unused images
                docker image prune -f

                echo "✅ Cleanup complete!"
            '''
            cleanWs()
        }
        success {
            echo '🎉 Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
```

**Pipeline Stages Explained:**

| Stage                  | Purpose                  | Key Actions                              |
| ---------------------- | ------------------------ | ---------------------------------------- |
| **Checkout**           | Get code from repository | `checkout scm`                           |
| **Build Docker Image** | Create container image   | `docker build`, `docker tag`             |
| **Integration Test**   | Validate the build       | Start MySQL, run app, test HTTP endpoint |
| **Push to Docker Hub** | Store image in registry  | `docker login`, `docker push`            |
| **Cleanup**            | Remove test artifacts    | Stop containers, prune images            |

### **Step 2: Create Feature Branch**

```bash
# Create and switch to feature branch
git checkout -b feature/add-logging
git push origin feature/add-logging
```

**Screenshot**: ![creating_git_feature_branch_practice1.png](screenshots/creating_git_feature_branch_practice1.png)
_Figure 15: Creating a feature branch in Git_

### **Step 3: Jenkins Multi-Branch Pipeline Setup**

In Jenkins:

1. Create a new "Multibranch Pipeline" item
2. Add your GitHub repository URL
3. Jenkins automatically scans branches and creates pipelines

**Screenshot**: ![jenkins_blue_ocean_showing diffrent_branches_main_feature.png](screenshots/jenkins_blue_ocean_showing%20diffrent_branches_main_feature.png)
_Figure 16: Jenkins Blue Ocean showing different branches (main and feature)_

### **Step 4: Feature Branch Build Success**

**Screenshot**: ![feature_branch_jenkins_multibuild_success.png](screenshots/feature_branch_jenkins_multibuild_success.png)
_Figure 17: Feature branch multi-build successful in Jenkins_

### **Step 5: Main Branch Build Success**

**Screenshot**: ![jenkins_main_multibuild_success.png](screenshots/jenkins_main_multibuild_success.png)
_Figure 18: Main branch multi-build successful in Jenkins_

### **Step 6: Docker Hub Images from Jenkins**

**Screenshot**: ![docker_hub_showing_images_pushed_from_jenkins.png](screenshots/docker_hub_showing_images_pushed_from_jenkins.png)
_Figure 19: Docker Hub showing images pushed from Jenkins CI pipeline_

---

## **📸 Complete Screenshot Reference Table**

| #   | Screenshot Name                                                 | Description                            | Key Insights                                                      | Figure    |
| --- | --------------------------------------------------------------- | -------------------------------------- | ----------------------------------------------------------------- | --------- |
| 1   | mysql_container_intsalled_and_working.png                       | MySQL container successfully installed | Shows container with "healthy" status after pull                  | Figure 1  |
| 2   | docker_ps_command_showing_containers.png                        | Docker ps showing running containers   | Lists both MySQL and app containers with their status             | Figure 2  |
| 3   | docker_network_command.png                                      | Creating custom Docker network         | Demonstrates network isolation for secure container communication | Figure 3  |
| 4   | docker_mysql_executing_create_user.sql.png                      | Executing create_user.sql              | Creating database user with proper permissions                    | Figure 4  |
| 5   | mysql_created_tables.png                                        | MySQL tables successfully created      | Verifies database schema loaded correctly                         | Figure 5  |
| 6   | localHoast_5000_login_page.png                                  | Tooling app login page                 | Application accessible at localhost:5000                          | Figure 6  |
| 7   | toolong_dashboard.png                                           | Tooling app dashboard                  | Successful login with test credentials                            | Figure 7  |
| 8   | docker_compose_ps_all_containers_running.png                    | Docker Compose containers running      | Both services healthy and running                                 | Figure 8  |
| 9   | docker_login.png                                                | Docker Hub login                       | Authenticated to push images                                      | Figure 9  |
| 10  | docker_repo_created.png                                         | Docker Hub repository created          | Repository ready for images                                       | Figure 10 |
| 11  | all_repositories_dockerhub_dashoard.png                         | All repositories dashboard             | Overview of all Docker Hub repositories                           | Figure 11 |
| 12  | docker_image_tagging_for_later_push.png                         | Tagging images for push                | Proper image tagging with version                                 | Figure 12 |
| 13  | docker_pushed_images.png                                        | Images successfully pushed             | Push operation completed successfully                             | Figure 13 |
| 14  | after_pushing_image_dockerhub_image_showing updated_version.png | Updated image version                  | New image version visible in Docker Hub                           | Figure 14 |
| 15  | creating_git_feature_branch_practice1.png                       | Creating feature branch                | Git branch workflow demonstration                                 | Figure 15 |
| 16  | jenkins_blue_ocean_showing diffrent_branches_main_feature.png   | Jenkins Blue Ocean branches            | Multi-branch pipeline visualization                               | Figure 16 |
| 17  | feature_branch_jenkins_multibuild_success.png                   | Feature branch build success           | CI pipeline passing for feature branch                            | Figure 17 |
| 18  | jenkins_main_multibuild_success.png                             | Main branch build success              | CI pipeline passing for main branch                               | Figure 18 |
| 19  | docker_hub_showing_images_pushed_from_jenkins.png               | Docker Hub images from Jenkins         | Automated image pushes from CI pipeline                           | Figure 19 |

---

## **Repository References**

| File         | Repository | Link                                                              | Purpose                                     |
| ------------ | ---------- | ----------------------------------------------------------------- | ------------------------------------------- |
| Dockerfile   | tooling-2  | [View](https://github.com/Munyat/tooling-2/blob/main/Dockerfile)  | Defines the container image for Tooling app |
| Jenkinsfile  | tooling-2  | [View](https://github.com/Munyat/tooling-2/blob/main/Jenkinsfile) | CI/CD pipeline configuration                |
| html/        | tooling-2  | [View](https://github.com/Munyat/tooling-2/tree/main/html)        | Application source code                     |
| Dockerfile   | tooling3   | [View](https://github.com/Munyat/tooling3/blob/main/Dockerfile)   | PHP-Todo app container definition           |
| Jenkinsfile  | tooling3   | [View](https://github.com/Munyat/tooling3/blob/main/Jenkinsfile)  | PHP-Todo CI/CD pipeline                     |
| tooling.yaml | tooling3   | [View](https://github.com/Munyat/tooling3/blob/main/tooling.yaml) | Docker Compose configuration                |

---

## **📊 Key Concepts Summary**

| Concept              | Description                    | Why It Matters                              |
| -------------------- | ------------------------------ | ------------------------------------------- |
| **Docker Image**     | Blueprint for containers       | Portable, versioned application package     |
| **Docker Container** | Running instance of an image   | Isolated, lightweight execution environment |
| **Docker Network**   | Virtual network for containers | Enables secure service discovery            |
| **Port Mapping**     | Host:container port binding    | Exposes container services to outside world |
| **Volume**           | Persistent data storage        | Preserves data beyond container lifecycle   |
| **Docker Compose**   | Multi-container orchestration  | Declarative service definition              |
| **Docker Hub**       | Image registry                 | Centralized image storage and sharing       |
| **CI/CD Pipeline**   | Automated build-test-deploy    | Ensures code quality and consistency        |

---

## **🎯 Troubleshooting Guide**

| Issue                       | Symptoms                      | Solution                                                             |
| --------------------------- | ----------------------------- | -------------------------------------------------------------------- |
| **Connection refused**      | App can't connect to database | Ensure both containers on same network; use service name as hostname |
| **Apache ServerName error** | Warning in logs               | Add `ServerName localhost` to Apache config                          |
| **Port already in use**     | Error when starting container | Change host port mapping (e.g., `8086:80`)                           |
| **Permission denied**       | Can't access files            | Run `chown -R www-data:www-data /var/www/html`                       |
| **MySQL not healthy**       | Container shows "unhealthy"   | Check logs; ensure enough resources                                  |
| **Image push fails**        | Authentication error          | Run `docker login` first                                             |

---

## **📝 Summary of Achievements**

✅ **MySQL Containerized** - Successfully running with custom network
✅ **Tooling App Containerized** - Built and running with Docker
✅ **Apache Error Fixed** - ServerName configured properly
✅ **Docker Compose Implemented** - Multi-container orchestration
✅ **Docker Hub Integration** - Images pushed to registry
✅ **Git Feature Branch Workflow** - Branch created and managed
✅ **Jenkins CI/CD Pipeline** - Multi-branch pipeline with test stage
✅ **Main & Feature Branch Builds** - Both branches building successfully
✅ **Automated Image Pushing** - Images pushed from Jenkins to Docker Hub
