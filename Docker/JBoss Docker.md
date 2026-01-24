# JBoss Docker Deployment Guide

A comprehensive guide for deploying JBoss application server using Docker with Nexus registry.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Nexus Repository Configuration](#nexus-repository-configuration)
- [Docker Configuration](#docker-configuration)
- [Available JBoss Images](#available-jboss-images)
- [Deployment](#deployment)
- [Docker Management Commands](#docker-management-commands)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have:

- Docker Desktop installed and running
- Network access to Nexus repository at `172.16.116.24:8082`
- Appropriate permissions to pull images from the registry
- Basic understanding of Docker and JBoss

---

## Nexus Repository Configuration

### Available Images

The following JBoss images are available in the Nexus repository:

| Image | Version | Full Path |
|-------|---------|-----------|
| JBoss EAP | 8.0.6 | `172.16.116.24:8082/jboss:8.0.6` |
| JBoss EAP | 7.4.20 | `172.16.116.24:8082/jboss:7.4.20` |
| JBoss EAP | 6.4.24 | `172.16.116.24:8082/jboss:6.4.24` |
| Oracle DB | 19c (clean) | `172.16.116.24:8082/oracle-19:0.0.1-clean` |

### Nexus Credentials

Use these credentials to authenticate with the Nexus Docker registry:
```bash
Username: docker-admin
Password: UBMongolia1234
```

---

## Docker Configuration

### Step 1: Configure Insecure Registry

Since the Nexus registry uses HTTP (not HTTPS), you need to configure Docker to allow insecure connections.

1. Open **Docker Desktop**
2. Navigate to **Settings** → **Docker Engine**
3. Add the following configuration to the JSON:
```json
{
  "ipv6": false,
  "insecure-registries": ["172.16.116.24:8082"]
}
```

4. Click **Apply & Restart**

### Step 2: Login to Nexus Registry

Authenticate with the Nexus Docker registry:
```bash
docker login 172.16.116.24:8082
```

When prompted:
- **Username:** `docker-admin`
- **Password:** `UBMongolia1234`

> **Note:** Credentials are stored securely in your system's keychain.

---

## Available JBoss Images

### Pull JBoss 8.0.6
```bash
docker pull 172.16.116.24:8082/jboss:8.0.6
```

### Pull JBoss 7.4.20 (Recommended)
```bash
docker pull 172.16.116.24:8082/jboss:7.4.20
```

### Pull JBoss 6.4.24
```bash
docker pull 172.16.116.24:8082/jboss:6.4.24
```

### Verify Downloaded Images
```bash
docker images | grep jboss
```

---

## Deployment

### Directory Structure

Create the following directory structure for your deployment:
```
project-root/
├── docker-compose.yml
├── config/
│   ├── nes.properties
│   ├── log4j2_develop.xml
│   ├── standalone.conf
│   └── standalone.xml
├── drivers/
│   ├── oracle/
│   ├── clickhouse/
│   └── postgresql/
├── logs/
└── nes_log/
```

### Docker Compose Configuration

Create a `docker-compose.yml` file with the following content:
```yaml
services:
  jboss:
    image: 172.16.116.24:8082/jboss:7.4.20
    container_name: jboss7
    user: root
    ports:
      - "8787:8787"     # Debug port
      - "40351:40351"   # Application port
      - "40350:40350"   # HTTP port
      - "40354:40354"   # Management port
    environment:
      - JBOSS_ADMIN_USER=admin
      - JBOSS_ADMIN_PASSWORD=Mongolia@1
    volumes:
      # Configuration files
      - ./config/nes.properties:/opt/jboss/bin/nes.properties
      - ./config/log4j2_develop.xml:/opt/jboss/standalone/configuration/log4j2_develop.xml
      - ./config/standalone.conf:/opt/jboss/bin/standalone.conf
      - ./config/standalone.xml:/opt/jboss/standalone/configuration/standalone.xml
      
      # Database drivers
      - ./drivers/oracle:/opt/jboss/modules/oracle/jdbc/main
      - ./drivers/clickhouse:/opt/jboss/modules/com/clickhouse/main
      - ./drivers/postgresql:/opt/jboss/modules/postgre/jdbc/main
      
      # Log directories
      - ./logs:/opt/jboss/standalone/log
      - ./nes_log:/opt/jboss/bin/nes_log
    entrypoint: ["/bin/sh", "-c"]
    command: 
      - |
        /opt/jboss/bin/add-user.sh admin Mongolia@1 --silent
        /opt/jboss/bin/standalone.sh -b 0.0.0.0 -bmanagement 0.0.0.0
    networks:
      - gcm-network

networks:
  gcm-network:
    external: true
```

### Port Mapping Explanation

| Host Port | Container Port | Description |
|-----------|----------------|-------------|
| 8787 | 8787 | Remote debugging port |
| 40351 | 40351 | Application-specific port |
| 40350 | 40350 | HTTP server port |
| 40354 | 40354 | Management console port |

### Volume Mounts Explanation

**Configuration Files:**
- `nes.properties` - Application-specific properties
- `log4j2_develop.xml` - Logging configuration
- `standalone.conf` - JBoss startup configuration
- `standalone.xml` - Server configuration

**Database Drivers:**
- Oracle JDBC driver modules
- ClickHouse JDBC driver modules
- PostgreSQL JDBC driver modules

**Logs:**
- `logs/` - JBoss server logs
- `nes_log/` - Application-specific logs

---

## Docker Management Commands

### Starting the Container

Start the JBoss container in detached mode:
```bash
docker compose up -d
```

> **Tip:** The `-d` flag runs the container in the background.

### Stopping the Container

Stop the running container:
```bash
docker compose down
```

### Restarting the Container

Restart the container to apply configuration changes:
```bash
docker compose restart
```

### Viewing Container Status

Check if the container is running:
```bash
docker ps
```

View all containers (including stopped):
```bash
docker ps -a
```

### Monitoring Resource Usage

Real-time resource statistics:
```bash
docker stats jboss7
```

View statistics for all containers:
```bash
docker stats
```

### Viewing Logs

View container logs:
```bash
docker logs jboss7
```

Follow logs in real-time:
```bash
docker logs -f jboss7
```

View last 100 lines:
```bash
docker logs --tail 100 jboss7
```

View logs with timestamps:
```bash
docker logs -t jboss7
```

### Accessing the Container

Execute commands inside the container:
```bash
docker exec -it jboss7 /bin/bash
```

Run a single command:
```bash
docker exec jboss7 ls -la /opt/jboss
```

### Inspecting the Container

View detailed container information:
```bash
docker inspect jboss7
```

View container IP address:
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' jboss7
```

### Managing Images

List all Docker images:
```bash
docker images
```

Remove unused images:
```bash
docker image prune
```

Remove specific image:
```bash
docker rmi 172.16.116.24:8082/jboss:7.4.20
```

### Network Commands

List Docker networks:
```bash
docker network ls
```

Create external network (if not exists):
```bash
docker network create gcm-network
```

Inspect network:
```bash
docker network inspect gcm-network
```

### Container Cleanup

Remove stopped container:
```bash
docker rm jboss7
```

Remove container forcefully:
```bash
docker rm -f jboss7
```

Remove all stopped containers:
```bash
docker container prune
```

### System Cleanup

Remove all unused resources:
```bash
docker system prune
```

Remove all unused resources including volumes:
```bash
docker system prune -a --volumes
```

### Backup and Export

Export container filesystem:
```bash
docker export jboss7 > jboss7_backup.tar
```

Save image to tar file:
```bash
docker save 172.16.116.24:8082/jboss:7.4.20 > jboss7_image.tar
```

Load image from tar file:
```bash
docker load < jboss7_image.tar
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Container Won't Start

**Check logs:**
```bash
docker logs jboss7
```

**Verify configuration files exist:**
```bash
ls -la config/
ls -la drivers/
```

#### 2. Port Already in Use

**Find process using the port:**
```bash
# Linux/Mac
lsof -i :40350

# Windows
netstat -ano | findstr :40350
```

**Kill the process or change the port mapping in docker-compose.yml**

#### 3. Permission Denied Errors

**Check volume permissions:**
```bash
ls -la logs/
ls -la nes_log/
```

**Fix permissions:**
```bash
sudo chown -R 1000:1000 logs/ nes_log/
```

#### 4. Cannot Pull Image from Nexus

**Verify Docker login:**
```bash
docker login 172.16.116.24:8082
```

**Check insecure-registries configuration**

**Test network connectivity:**
```bash
ping 172.16.116.24
curl http://172.16.116.24:8082
```

#### 5. Container Exits Immediately

**Check exit code:**
```bash
docker ps -a
```

**Review full logs:**
```bash
docker logs jboss7
```

**Run container interactively for debugging:**
```bash
docker run -it --rm 172.16.116.24:8082/jboss:7.4.20 /bin/bash
```

#### 6. Network Not Found

**Create the external network:**
```bash
docker network create gcm-network
```

---

## Quick Reference Commands

### Essential Commands Cheat Sheet
```bash
# Start container
docker compose up -d

# Stop container
docker compose down

# View logs
docker logs -f jboss7

# Container stats
docker stats jboss7

# Access container shell
docker exec -it jboss7 /bin/bash

# Restart container
docker compose restart

# View running containers
docker ps

# Remove container
docker rm -f jboss7

# Pull latest image
docker pull 172.16.116.24:8082/jboss:7.4.20
```

---

## Access Points

After successful deployment, access JBoss through:

- **HTTP Server:** `http://localhost:40350`
- **Management Console:** `http://localhost:40354`
- **Debug Port:** `localhost:8787`

**Default Credentials:**
- Username: `admin`
- Password: `Mongolia@1`

---

## Additional Resources

### JBoss Documentation
- [JBoss EAP Documentation](https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform)

### Docker Documentation
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [Docker CLI Reference](https://docs.docker.com/engine/reference/commandline/cli/)

---

## Notes

> **Security Warning:** The credentials in this guide are for development purposes only. Always use strong, unique passwords in production environments.

> **Network Configuration:** Ensure the `gcm-network` exists before starting the container. Create it with `docker network create gcm-network` if needed.

> **Volume Persistence:** Data in mounted volumes persists even when the container is removed. To completely reset, delete the volume directories.

---

*Document Version: 1.0*  
*Last Updated: January 2026*