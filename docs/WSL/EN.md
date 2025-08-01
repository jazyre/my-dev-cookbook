# Laravel Sail MySQL Connection Troubleshooting Guide

## Problem
Error `SQLSTATE[HY000] [2002] php_network_getaddresses: getaddrinfo for mysql failed: Name or service not known` when running Laravel Artisan commands like `migrate`, `db:seed`, etc.

## Root Causes

### 1. Missing or Incorrect .env Configuration
- `.env` file doesn't exist or has incomplete database settings
- `APP_KEY` is not set
- `DB_PASSWORD` is empty
- `DB_USERNAME` is inappropriate

### 2. MySQL Container Issues
- MySQL container stopped or crashed
- Problematic MySQL image
- Socket file conflicts
- Docker volume issues

### 3. Docker Configuration Problems
- Incorrect settings in `docker-compose.yml`
- Network configuration issues
- Port conflicts

## Step-by-Step Solutions

### Step 1: Check Container Status

```bash
# Check running containers
docker ps

# Check all containers (including stopped ones)
docker ps -a | grep mysql

# Check MySQL container logs
docker logs karokia-mysql-1
```

### Step 2: Configure .env File

#### a) Generate APP_KEY
```bash
# Change .env file permissions
chmod 666 .env

# Generate APP_KEY
./vendor/bin/sail artisan key:generate
```

#### b) Set Database Password
```bash
# Edit .env file and set password
sed -i 's/DB_PASSWORD=/DB_PASSWORD=password/' .env

# Or manually edit .env file:
# DB_PASSWORD=password
```

#### c) Set Appropriate Username
```bash
# Change username from root to sail
sed -i 's/DB_USERNAME=root/DB_USERNAME=sail/' .env

# Or manually:
# DB_USERNAME=sail
```

### Step 3: Fix docker-compose.yml

If you're using a problematic MySQL image, change it:

```yaml
# Before (problematic)
mysql:
    image: 'mysql/mysql-server:8.0'
    # ...

# After (better)
mysql:
    image: 'mysql:8.0'
    ports:
        - '${FORWARD_DB_PORT:-3306}:3306'
    environment:
        MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
        MYSQL_DATABASE: '${DB_DATABASE}'
        MYSQL_USER: '${DB_USERNAME}'
        MYSQL_PASSWORD: '${DB_PASSWORD}'
        MYSQL_ALLOW_EMPTY_PASSWORD: 1
    volumes:
        - 'sail-mysql:/var/lib/mysql'
    networks:
        - sail
    healthcheck:
        test:
            - CMD
            - mysqladmin
            - ping
            - '-p${DB_PASSWORD}'
        retries: 3
        timeout: 5s
```

### Step 4: Complete Restart

```bash
# Stop all containers and remove volumes
./vendor/bin/sail down -v

# Restart
./vendor/bin/sail up -d

# Wait for MySQL to be ready (at least 30 seconds)
sleep 30
```

### Step 5: Test Connection

```bash
# Run migrations
./vendor/bin/sail artisan migrate

# Or test connection with Tinker
./vendor/bin/sail artisan tinker
# Then in Tinker: DB::connection()->getPdo();
```

## Recommended .env Settings

```env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:YOUR_GENERATED_KEY_HERE
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

# Other settings...
```

## Important Notes

### 1. Wait Time
- MySQL container needs time to fully initialize
- Wait at least 30 seconds before running database commands

### 2. File Permissions
- Ensure `.env` file is writable
- If needed: `chmod 666 .env`

### 3. Port Conflicts
- Ensure port 3306 is free on host system
- Change port in `docker-compose.yml` if there's a conflict

### 4. Docker Volumes
- If problems persist, clear volumes: `./vendor/bin/sail down -v`
- This will cause loss of existing data

## Useful Debugging Commands

```bash
# Check all container statuses
docker ps -a

# Check all container logs
docker logs karokia-mysql-1
docker logs karokia-laravel.test-1

# Check Docker networks
docker network ls
docker network inspect karokia_sail

# Check Docker volumes
docker volume ls
docker volume inspect karokia_sail-mysql

# Test direct MySQL connection
./vendor/bin/sail mysql

# Check Laravel configuration
./vendor/bin/sail artisan config:cache
./vendor/bin/sail artisan config:clear
```

## Common Issues and Solutions

### Issue: "Another process with pid X is using unix socket file"
**Solution**: Clear volumes and restart
```bash
./vendor/bin/sail down -v
./vendor/bin/sail up -d
```

### Issue: "MYSQL_USER and MYSQL_PASSWORD are for configuring a regular user"
**Solution**: Change DB_USERNAME from `root` to `sail`

### Issue: "Permission denied" in .env file
**Solution**: Change file permissions
```bash
chmod 666 .env
```

## Useful Resources

- [Laravel Sail Documentation](https://laravel.com/docs/sail)
- [Docker MySQL Official Image](https://hub.docker.com/_/mysql)
- [Laravel Database Configuration](https://laravel.com/docs/database)

## Conclusion

This issue usually occurs due to incomplete or incorrect settings in `.env` and `docker-compose.yml` files. Following the steps above should resolve the problem. If the issue persists, checking container logs and Docker network settings can be helpful.

---

**Note**: This guide is designed for Laravel Sail and Docker. If you're using other environments, you may need different configurations.

## Troubleshooting Checklist

- [ ] Check if all containers are running
- [ ] Verify .env file exists and is properly configured
- [ ] Ensure APP_KEY is generated
- [ ] Set DB_PASSWORD and DB_USERNAME correctly
- [ ] Check MySQL container logs for errors
- [ ] Verify Docker network configuration
- [ ] Test database connection
- [ ] Run migrations successfully

## Quick Fix Commands

```bash
# Complete reset and restart
chmod 666 .env
./vendor/bin/sail artisan key:generate
sed -i 's/DB_PASSWORD=/DB_PASSWORD=password/' .env
sed -i 's/DB_USERNAME=root/DB_USERNAME=sail/' .env
./vendor/bin/sail down -v
./vendor/bin/sail up -d
sleep 30
./vendor/bin/sail artisan migrate
``` 

---

## Full Troubleshooting Journey & Initial Analysis

*This section details the original problem, all the troubleshooting steps that were attempted but failed, and the initial conclusion before the final solution was found. This information is highly valuable for a deeper understanding of the issue.*

### 1. Executive Summary
This report details a persistent database connection error encountered when executing PHP Artisan commands from the command-line interface (CLI) within a Laravel Sail (Docker) container on a Windows/WSL2 host. The primary error is `Name or service not known` or `Temporary failure in name resolution`, indicating that the PHP process cannot resolve the `mysql` hostname to its corresponding container IP address.

The critical paradox is that network-level hostname resolution from within the same container is successful using standard Linux tools. Executing `curl mysql:3306` from the Laravel application container's shell confirms that the Docker internal network and DNS service are functioning correctly. This isolates the problem specifically to the **PHP-CLI runtime environment**, which appears unable to utilize Docker's DNS resolution mechanism, while other processes in the same container can.

### 2. Environment Details
* **Framework**: Laravel
* **Development Environment**: Laravel Sail
* **Orchestration**: Docker Compose
* **Host OS**: Windows 10/11
* **Virtualization Layer**: Windows Subsystem for Linux v2 (WSL2)
* **Container Engine**: Docker Desktop for Windows

### 3. Systematic Troubleshooting and Diagnostics Performed (Unsuccessful)
A comprehensive set of diagnostic and remediation steps was executed to isolate the root cause. All standard solutions have failed to resolve the issue.

* **Environment and Service Lifecycle Management**:
    * Container Restart: Graceful shutdown and startup of all Sail containers (`sail down` followed by `sail up -d`).
    * WSL Restart: A full shutdown of the WSL2 virtual machine (`wsl --shutdown` from PowerShell).
    * Docker Daemon Restart: Complete restart of the Docker Desktop application.
    * Full Image Rebuild: Forced recreation of all Docker images without using cache (`sail build --no-cache`).
    * Complete Reinstallation: A full factory reset of Docker Desktop, followed by a complete reinstallation of the WSL2 distribution and Docker.
* **Configuration Verification**:
    * Laravel Environment (`.env`): The `DB_HOST` variable is correctly set to `mysql`, matching the service name in the Docker Compose file.
    * Docker Compose (`docker-compose.yml`): The MySQL service is correctly named `mysql`, and all services are correctly attached to the default `sail` bridge network.
* **Application-Level Cache**:
    * Configuration Cache: Cleared using `php artisan config:clear`.
    * All Caches: All application-level caches were purged using `php artisan optimize:clear`.
* **Network-Level Diagnostics**:
    * Container Status: All containers, including the `mysql` service, are verified to be running and healthy (`sail ps`).
    * Direct Hostname Resolution Test: As noted, `curl mysql:3306` from within the application container's shell resolves the hostname successfully.
* **IP Address vs. Hostname**:
    * Replacing `DB_HOST=mysql` with the container's direct IP address changed the error from `Name resolution` to `No route to host`, indicating a deeper routing problem.

### 4. Overall Conclusion from Initial Analysis
Given that all application-level and container-level troubleshooting was exhausted, the evidence points towards a rare, low-level anomaly within the virtual networking stack of the Windows host, likely involving the Hyper-V Virtual Switch or how WSL2's network interfaces with the Docker engine's networking layer.