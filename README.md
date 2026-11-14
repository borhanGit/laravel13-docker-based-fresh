# 🚀 Procurement API — Laravel 13 (Docker Sail & phpMyAdmin)

A fresh Laravel 13 application running completely on Docker using **Laravel Sail**, configured with **MySQL 8.4**, **phpMyAdmin**, **Redis**, and **Mailpit**.

> **Note**: This setup requires **NO local PHP or Composer** installation on your host system. Everything runs inside Docker.

---

## 📌 Table of Contents
- [Quick Start](#-quick-start-current-project)
- [Services & Ports](#-services--ports)
- [Database Credentials](#-database-credentials)
- [Useful Sail Commands](#-useful-sail-commands)
- [Manual Setup Guide from Scratch (Docker-Only)](#-manual-setup-guide-from-scratch-docker-only)

---

## ⚡ Quick Start (Current Project)

### 1. Start Containers
```bash
./vendor/bin/sail up -d
# or using docker-compose
docker-compose up -d
```

### 2. Stop Containers
```bash
./vendor/bin/sail down
```

### 3. Run Migrations
```bash
./vendor/bin/sail artisan migrate
```

---

## 🌐 Services & Ports

Ports have been configured in `.env` to avoid conflicts with existing host services:

| Service | Host URL / Port | Container Port | Description |
|---|---|---|---|
| **Laravel App** | [http://localhost](http://localhost) | `80` | Web Application |
| **phpMyAdmin** | [http://localhost:8081](http://localhost:8081) | `80` | Database GUI Manager |
| **Mailpit Web UI** | [http://localhost:8025](http://localhost:8025) | `8025` | Email Testing Dashboard |
| **Mailpit SMTP** | `localhost:1025` | `1025` | SMTP Server |
| **MySQL** | `localhost:3308` | `3306` | MySQL 8.4 Database Server |
| **Redis** | `localhost:6381` | `6379` | Cache / Queue Server |
| **Vite Dev Server** | `localhost:5173` | `5173` | Frontend Asset Server |

---

## 🔑 Database Credentials

- **phpMyAdmin Login**:
  - **Server / Host**: `mysql`
  - **Username**: `sail`
  - **Password**: `password`
- **External DB Client (e.g., DBeaver, TablePlus, Navicat)**:
  - **Host**: `127.0.0.1` (or `localhost`)
  - **Port**: `3308`
  - **Database**: `laravel`
  - **User**: `sail`
  - **Password**: `password`

---

## 💡 Useful Sail Commands

Instead of running PHP/Composer commands on your host, run them via Sail:

```bash
# Artisan commands
./vendor/bin/sail artisan <command>
./vendor/bin/sail artisan route:list
./vendor/bin/sail artisan make:controller Api/AuthController

# Composer commands
./vendor/bin/sail composer require <package-name>
./vendor/bin/sail composer update

# Node / NPM commands
./vendor/bin/sail npm install
./vendor/bin/sail npm run dev

# Testing
./vendor/bin/sail artisan test

# Open a Bash shell inside the Laravel container
./vendor/bin/sail shell

# Open a Root Bash shell
./vendor/bin/sail root-shell

# View container logs
./vendor/bin/sail logs -f
```

> **Tip (Sail Alias)**: Add the following line to your `~/.bashrc` or `~/.zshrc` so you can type `sail` instead of `./vendor/bin/sail`:
> ```bash
> alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'
> ```

---

## 📖 Manual Setup Guide from Scratch (Docker-Only)

If you want to set up a brand-new Laravel project in the future without PHP or Composer installed on your machine, follow these steps:

### Step 1: Create Folder & Clone Fresh Laravel
```bash
mkdir my-laravel-app && cd my-laravel-app
git clone https://github.com/laravel/laravel.git .
```

### Step 2: Create `.env` File
```bash
cp .env.example .env
```

### Step 3: Add `laravel/sail` to `composer.json`
Add `"laravel/sail": "^1.41"` under `"require-dev"` in `composer.json`:
```json
"require-dev": {
    "fakerphp/faker": "^1.23",
    "laravel/sail": "^1.41",
    "mockery/mockery": "^1.6",
    "nunomaduro/collision": "^8.6",
    "phpunit/phpunit": "^12.5.12"
}
```

### Step 4: Install Composer Dependencies & Generate Key (Using Docker)
Run a temporary Docker container to install composer packages and generate the app key:
```bash
# 1. Install dependencies into vendor/
docker run --rm -u $(id -u):$(id -g) -v $(pwd):/var/www/html -w /var/www/html composer:latest composer install --ignore-platform-reqs

# 2. Generate application encryption key
docker run --rm -v $(pwd):/var/www/html -w /var/www/html composer:latest php artisan key:generate
```

### Step 5: Create `docker-compose.yml`
Create `docker-compose.yml` in the root folder (remove `compose.yaml` if it exists: `rm -f compose.yaml`):

```yaml
services:
    laravel.test:
        build:
            context: ./vendor/laravel/sail/runtimes/8.5
            dockerfile: Dockerfile
            args:
                WWWGROUP: '${WWWGROUP}'
        image: sail-8.5/app
        extra_hosts:
            - 'host.docker.internal:host-gateway'
        ports:
            - '${APP_PORT:-80}:80'
            - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
        environment:
            WWWUSER: '${WWWUSER}'
            LARAVEL_SAIL: 1
            XDEBUG_MODE: '${SAIL_XDEBUG_MODE:-off}'
            XDEBUG_CONFIG: '${SAIL_XDEBUG_CONFIG:-client_host=host.docker.internal}'
            IGNITION_LOCAL_SITES_PATH: '${PWD}'
        volumes:
            - '.:/var/www/html'
        networks:
            - sail
        depends_on:
            - mysql
            - redis
            - mailpit

    mysql:
        image: 'mysql:8.4'
        ports:
            - '${FORWARD_DB_PORT:-3308}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ROOT_HOST: '%'
            MYSQL_DATABASE: '${DB_DATABASE}'
            MYSQL_USER: '${DB_USERNAME}'
            MYSQL_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ALLOW_EMPTY_PASSWORD: 1
            MYSQL_EXTRA_OPTIONS: '${MYSQL_EXTRA_OPTIONS:-}'
        volumes:
            - 'sail-mysql:/var/lib/mysql'
            - './vendor/laravel/sail/database/mysql/create-testing-database.sh:/docker-entrypoint-initdb.d/10-create-testing-database.sh'
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

    redis:
        image: 'redis:alpine'
        ports:
            - '${FORWARD_REDIS_PORT:-6381}:6379'
        volumes:
            - 'sail-redis:/data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - redis-cli
                - ping
            retries: 3
            timeout: 5s

    mailpit:
        image: 'axllent/mailpit:latest'
        ports:
            - '${FORWARD_MAILPIT_PORT:-1025}:1025'
            - '${FORWARD_MAILPIT_DASHBOARD_PORT:-8025}:8025'
        networks:
            - sail

    phpmyadmin:
        image: 'phpmyadmin:latest'
        ports:
            - '${FORWARD_PHPMYADMIN_PORT:-8081}:80'
        environment:
            PMA_ARBITRARY: 1
            PMA_HOST: mysql
            PMA_PORT: 3306
            PMA_USER: '${DB_USERNAME}'
            PMA_PASSWORD: '${DB_PASSWORD}'
            UPLOAD_LIMIT: 300M
        networks:
            - sail
        depends_on:
            - mysql

networks:
    sail:
        driver: bridge

volumes:
    sail-mysql:
        driver: local
    sail-redis:
        driver: local
```

### Step 6: Configure `.env`
Update database connection and port mappings in `.env`:

```env
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

REDIS_CLIENT=phpredis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025

# Port Mapping (Custom ports to avoid conflicts)
APP_PORT=80
FORWARD_DB_PORT=3308
FORWARD_REDIS_PORT=6381
FORWARD_PHPMYADMIN_PORT=8081
FORWARD_MAILPIT_PORT=1025
FORWARD_MAILPIT_DASHBOARD_PORT=8025
```

### Step 7: Start Sail & Migrate Database
```bash
# Start all containers in background
./vendor/bin/sail up -d

# Run database migrations
./vendor/bin/sail artisan migrate
```
