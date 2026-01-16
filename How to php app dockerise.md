Project Structure

project-root/
 ├── docker-compose.yml
 ├── nginx/
 │    ├── Dockerfile
 │    └── default.conf
 ├── php/
 │    └── Dockerfile
 └── ccc/   (your PHP application code)

Need to copy CCC(application) folder php and nginx folder first
sudo cp ccc -r nginx
sudo cp ccc -r php

Unify the app path to /app in both containers


✅ Nginx Dockerfile (nginx/Dockerfile)

FROM nginx:latest
COPY ../ccc /app
COPY default.conf /etc/nginx/conf.d/default.conf

✅ PHP-FPM Dockerfile (php/Dockerfile)

FROM php:8.2-fpm
RUN docker-php-ext-install mysqli pdo_mysql
WORKDIR /app
COPY ../ccc /app


✅ Nginx Config (nginx/default.conf)

server {
    listen 80;
    server_name localhost;

    root /app;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME /app$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}


✅ Docker Compose File (docker-compose.yml) for SWARM

services:
  nginx:
    build: ./nginx
    image: payeldiu/my-nginx:latest
    ports:
      - "80:80"
    depends_on:
      - php
      - mysqldb
    networks:
      - quickops_network
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      placement:
        constraints: [node.role == worker]

  php:
    build: ./php
    image: payeldiu/my-php-fpm:latest
    networks:
      - quickops_network
    deploy:
      restart_policy:
        condition: on-failure

  mysqldb:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - quickops_network
    deploy:
      restart_policy:
        condition: on-failure

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    environment:
      PMA_HOST: mysqldb
      PMA_USER: appuser
      PMA_PASSWORD: apppass
    ports:
      - "8080:80"   # Access phpMyAdmin at http://localhost:8080
    depends_on:
      - mysqldb
    networks:
      - quickops_network
    deploy:
      restart_policy:
        condition: on-failure

networks:
  quickops_network:
    driver: overlay # overlay networks require Swarm 

volumes:
  mysql_data:

✅ Docker Compose File (docker-compose.yml) for  DOCKER
services:
  nginx:
    build: ./nginx
    image: payeldiu/my-nginx:latest
    ports:
      - "80:80"
    depends_on:
      - php
      - mysqldb
    networks:
      - quickops_network
    
  php:
    build: ./php
    image: payeldiu/my-php-fpm:latest
    networks:
      - quickops_network
    

  mysqldb:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - quickops_network
  

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    environment:
      PMA_HOST: mysqldb
      PMA_USER: appuser
      PMA_PASSWORD: apppass
    ports:
      - "8080:80"   # Access phpMyAdmin at http://localhost:8080
    depends_on:
      - mysqldb
    networks:
      - quickops_network
    

networks:
  quickops_network:
    driver: bridge #Compose (non Swarm) only supports bridge (default), host, or none


volumes:
  mysql_data:

🚀 How to Run
1.	Build and start everything:
bash
docker compose up --build -d
2.	Access:
o	Your app → http:// 35.226.14.85
o	phpMyAdmin → http://localhost:8080
3.	Stop everything:
bash
docker-compose down

🚀 How to Deploy using DOCKER SWARM
docker swarm init
Swarm initialized: current node (abc123) is now a manager.
To add a worker to this swarm, run:
    docker swarm join --token <token> <manager-ip>:2377
docker swarm join --token SWMTKN-1-2q94h5xy3kv8l8okmgnven6qr9jj3to24op2784oyqjrn3zhiy-4y8v41rttg0luw0qey7ze5bj0 10.128.0.2:2377
1.	Build and push images:
docker build -t payeldiu/my-nginx:latest ./nginx
docker build -t payeldiu/my-php-fpm:latest ./php
docker push payeldiu/my-nginx:latest
docker push payeldiu/my-php-fpm:latest

2.	Deploy stack in Swarm:
docker stack deploy -c docker-compose.yml phpapp

3.	Verify:
docker stack services phpapp


Access:
•	Your app → http://localhost
•	phpMyAdmin → http://localhost:8080

