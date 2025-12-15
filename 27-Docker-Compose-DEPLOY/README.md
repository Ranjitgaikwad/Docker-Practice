---
title: "Master Docker Compose: Deploy, Scale, Load Balance, and Enable Persistence"
description: "Learn how to deploy and scale applications using Docker Compose with load balancing and session persistence."
---



---

## Step-01: Introduction

In this guide, we will explore Docker Compose's `deploy` option to achieve:

1. Scaling the **app-ums** service to 3 containers.
2. Using **web-nginx** to load balance traffic across 3 **app-ums** service containers.

---

## Step-02: Review `docker-compose.yaml`

```yaml
name: ums-stack
services:
  web-nginx:
    image: nginx:latest 
    container_name: ums-nginx
    ports:
      - "8080:8080"  # NGINX listens on port 8080 of the host
    depends_on:
      - app-ums
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf  # Custom NGINX configuration

  app-ums:
    image: ranjitgaikwad/usermgmt-webapp-v6:latest
    ports:
      - "8080"  # Only expose the container's port, let Docker choose the host port
    deploy:
      replicas: 3  # Scale the service to 3 instances       
    depends_on:
      - db-mysql
    environment:
      - DB_HOSTNAME=db-mysql
      - DB_PORT=3306
      - DB_NAME=webappdb
      - DB_USERNAME=root
      - DB_PASSWORD=dbpassword11

  db-mysql:
    container_name: ums-mysqldb
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: dbpassword11
      MYSQL_DATABASE=webappdb
    ports:
      - "3306:3306"
    volumes:
      - mydb:/var/lib/mysql

volumes:
  mydb:
```

---

## Step-03: Review `nginx.conf`

```conf
events { }

http {
  resolver 127.0.0.11 ipv6=off;  
  
  upstream app-ums {
    server app-ums:8080;  
    # Session persistence using client's IP address for stateful apps (UMS)
    # ip_hash;  # Disable this to see how load balancing works
  }

  server {
    listen 8080;

    location / {
      proxy_pass http://app-ums;
    }
  }
}
```

---

## Step-04: How to Start the UMS Stack

Start the UMS stack by following these steps:

```bash
# Change to the project directory
cd ums-stack

# Start the UMS Stack in detached mode
docker compose up -d
```

The `-d` flag runs the services in the background.

---

## Step-05: Verify Services

To verify the MySQL and UMS WebApp services:

```bash
# List running Docker containers
docker compose ps
```

To view the logs of the services:

```bash
docker compose logs db-mysql
docker compose logs app-ums 
docker compose logs web-nginx
```

To check logs of individual containers:

```bash
docker logs <CONTAINER-NAME>
docker logs ums-stack-app-ums-1
docker logs ums-stack-app-ums-2
docker logs ums-stack-app-ums-3
```

---

## Step-06: Verify the Load Balancing

```bash
# Access API that displays the container ID
http://localhost:8080/hello1

# Run a loop to check load balancing between multiple app-ums containers
while true; do curl http://localhost:8080/hello1; echo ""; sleep 1; done
```

**Observation**:

1. Requests will be distributed across the containers.

---

## Step-07: Enable Persistence in NGINX 

UMS WebApp is stateful. To enable session persistence and prevent issues like login failures when switching between containers, update the `nginx.conf`:

```conf
events { }

http {
  resolver 127.0.0.11 ipv6=off;  
  
  upstream app-ums {
    server app-ums:8080;  
    ip_hash;  # Enable session persistence using client's IP address
  }

  server {
    listen 8080;

    location / {
      proxy_pass http://app-ums;
    }
  }
}
```

---

## Step-08: Restart NGINX Container

After enabling `ip_hash` in `nginx.conf`, restart NGINX:

```bash
# Option 1: Restart NGINX service
docker compose restart web-nginx

# Option 2: Reload NGINX configuration without stopping the container
docker compose ps # Get container name
docker exec <nginx_container_name> nginx -s reload
docker exec ums-nginx nginx -s reload
```

---

## Step-09: Accessing the UMS WebApp

1. Open your browser and navigate to `http://localhost:8080`.
2. Log in with:
   - **Username**: `admin101`
   - **Password**: `password101`
3. You can:
   - View the list of users.
   - Create new users.
   - Test login functionality with the new users.

---

## Step-10: Connect to MySQL Container

To inspect or interact with the MySQL database, connect to the MySQL container:

```bash
docker exec -it ums-mysqldb mysql -u root -pdbpassword11
```

Inside the MySQL shell, run SQL queries:

```bash
mysql> show schemas;
mysql> use webappdb;
mysql> show tables;
mysql> select * from user;
```

---

## Step-11: Verify the Container to Container communication using Service Names using DNS
```bash
# List running Docker containers
docker compose ps

# Connect to ums-stack-app-ums-1 container
docker exec -it ums-stack-app-ums-1 /bin/bash

# Test DNS - web-nginx
nslookup <SERVICE-NAME>
nslookup web-nginx
dig web-nginx

# Test DNS - app-ums
nslookup <SERVICE-NAME>
nslookup app-ums
dig app-ums

# Test DNS - db-mysql
nslookup <SERVICE-NAME>
nslookup db-mysql
dig db-mysql
```
---

## Step-12: Stop and Clean Up

To stop and remove the running containers:

```bash
docker compose down
```

To remove the MySQL volume as well:

```bash
docker compose down -v
docker volume ls  # Verify that the volume was removed
```

---

## Conclusion

In this guide, you learned how to scale and deploy a multi-container stack using Docker Compose, including load balancing with NGINX and scaling the UMS WebApp. We enabled persistence for session management to ensure that the UMS WebApp operates effectively in a stateful environment. 

Docker Compose's `deploy` option and custom NGINX configurations provide powerful scaling and load balancing capabilities for microservices or multi-container applications.

---

