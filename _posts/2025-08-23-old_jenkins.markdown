---
layout: post
title: "SDET. Old Jenkins instance. Docker compose"
date:   2025-08-23 23:25:00 -0700
categories: sdet
---

# Bring up old Jenkins backup

## Objectives
I have old Jenkins backup and want to restore it to grab some Job's and their configuration.

## Docker image
Our project will have next structure:
```txt
jenkins
  |-jenkins_home # unzipped directory/content of some backup
  |-Dockerfile # Docker image with JDK1.8 for future container
  |-docker-compose.yaml # docker-compose configuration
```

Old jenkins has version [Jenkins 2.277.1] what support only JRE 1.8, 
so we need to build image with all necessary ingredients:

Dockerfile:
```text
# Use an base image with Java 8 that supports ARM64
FROM openjdk:8-jdk-alpine

# Install necessary packages for AWT headless mode (fonts)
# fontconfig is often needed for Java to find fonts
# ttf-dejavu provides common fonts
RUN apk add --no-cache fontconfig ttf-dejavu
# Set Jenkins specific environment variables
# ENV JENKINS_SLAVE_AGENT_PORT 50000
# Expose ports
EXPOSE 8080
EXPOSE 50000
# Create Jenkins user (important for permissions)
RUN addgroup -S jenkins && adduser -S -G jenkins jenkins
USER jenkins
# The actual Jenkins WAR and JENKINS_HOME will be mounted by docker-compose
# We just need to ensure the user exists and ports are exposed.
```

## Docker compose

docker-compose.yaml:
```text
# docker compose up --build -d

version: '3.8'
services:
  jenkins:
    build: . # Build from the Dockerfile in the current directory
    container_name: jenkins_old
    ports:
      - "8080:8080"
      - "50000:50000" # For Jenkins agents
    volumes:
      # Mount your existing jenkins.war file
      - "{your path here}/jenkins/jenkins.war:/usr/share/jenkins/jenkins.war"
      # Mount your existing JENKINS_HOME directory
      - "{your path here}/jenkins/jenkins_home:/var/jenkins_home"
    environment:
      - JAVA_OPTS=-Djava.awt.headless=true -Xmx4g -Xms2g
      - JENKINS_HOME=/var/jenkins_home # Explicitly set JENKINS_HOME inside container
    command: java -jar /usr/share/jenkins/jenkins.war --httpPort=8080
    restart: unless-stopped
    user: jenkins # Run the container as the 'jenkins' user created in the Dockerfile
    # Add container resource limits
    # mem_limit: 4500m # Roughly 6.5 GB (6500 * 1024 * 1024 bytes)
    # mem_reservation: 2000m # Roughly 3 GB
    # deploy:
    #   resources:
    #     limits:
    #       cpus: 2
    #       memory: 4G
```

After all configurations are ready, we can build Docker container, running command: 
```bash
$> docker compose up --build -d
```

Verifying:
```bash
$> docker ps -a
CONTAINER ID   IMAGE                   COMMAND                  CREATED      STATUS        PORTS                                                                                          NAMES
1404c18832fa   jenkins_oldv2-jenkins   "java -jar /usr/shar…"   6 days ago   Up 22 hours   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp, 0.0.0.0:50000->50000/tcp, [::]:50000->50000/tcp   jenkins_old
```
Open in browser URL `http://localhost:8080/`.
