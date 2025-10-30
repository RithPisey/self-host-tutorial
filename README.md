-----

# 🚀 Multi-Project Laravel Deployment: Docker, Caddy, & Shared MariaDB

This guide outlines a robust, modern deployment strategy for multiple Laravel applications using Docker, managed by the non-root user `dit`, with Caddy providing HTTPS as a reverse proxy.

## ✅ Prerequisite: User Permissions

To allow the user **`dit`** to manage Docker containers without using `sudo`, they must be added to the `docker` group.

```bash
# Run this command as root (or with sudo)
sudo usermod -aG docker dit
```

> ❗ **Action Required:** The user `dit` must **log out and log back in** for this change to take effect.

-----

## 1\. Shared MariaDB Service & Docker Network

We'll establish a persistent MariaDB container on a **shared external network** that all future applications will join.

1.  **Create Service Directory (as root):**

    ```bash
    mkdir -p /var/www/mariadb
    cd /var/www/mariadb
    ```

2.  **Create `docker-compose.yml`:**
    This defines the persistent database service and the `shared-network`.

    ```yaml
    services:
      mariadb:
        image: mariadb:latest
        container_name: mariadb
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: N8xIyxIC9QFx # CHANGE THIS!
        ports:
          - "3306:3306"
        volumes:
          - ./mariadb-data:/var/lib/mysql
        networks:
          - shared-network

    networks:
      shared-network:
        external: true
    ```

3.  **Start the Container (as root):**

    ```bash
    docker-compose up -d
    ```

    > **Result:** The `mariadb` container is running, and the **`shared-network`** is created. All apps connect using the hostname **`mariadb`**.

-----

## 2\. Dockerizing a Laravel Application 🐳 (Template)

Repeat these steps for **each** Laravel project. Navigate to your project directory (e.g., `/home/dit/my-laravel-app`).

### 2.1. The Multi-Stage `Dockerfile` (PHP-FPM App)

This file builds assets and dependencies in Stage 1 and copies only the essentials into a clean, slim production image in Stage 2.

```dockerfile
# ---- Stage 1: The "Builder" Stage (for Composer/Node/Assets) ----
FROM php:8.3-fpm AS builder
WORKDIR /var/www
# ( ... Install dependencies: git, zip, nodejs, composer, PHP extensions ... )
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl zip unzip libzip-dev libpng-dev libjpeg-dev libfreetype6-dev libonig-dev libwebp-dev \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt-get install -y nodejs \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip \
    && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY composer.json composer.lock ./
RUN composer install --no-interaction --no-scripts --prefer-dist --optimize-autoloader

COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build


# ---- Stage 2: The Final "Production" Stage (Lean Runtime) ----
FROM php:8.3-fpm
WORKDIR /var/www

# ( ... Install required RUNTIME PHP extensions ... )
RUN apt-get update && apt-get install -y --no-install-recommends \
    libzip-dev libpng-dev libjpeg-dev libfreetype6-dev libonig-dev libwebp-dev \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY --from=builder /var/www .

# Crucial Permissions Fix: Give www-data ownership
RUN mkdir -p /var/www/bootstrap/cache \
    && mkdir -p /var/www/storage/framework/views \
    && chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache \
    && chmod -R 775 /var/www/storage /var/www/bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]
```

### 2.2. Project-Specific `docker-compose.yml`

This sets up the **`app`** (PHP-FPM) and **`webserver`** (Nginx) for the project.

```yaml
services:
  app:
    build: .
    container_name: vml-erp # Use a unique name
    restart: unless-stopped
    volumes:
        - ./storage:/var/www/storage
    networks:
        - shared-network

  webserver:
    build:
        context: .
        dockerfile: Dockerfile.webserver # References the Nginx Dockerfile
    container_name: vml-erp-webserver # Use a unique name
    restart: unless-stopped
    ports:
        - '8001:80' # 👈 CRITICAL: CHANGE THIS HOST PORT FOR EVERY APP!
    networks:
        - shared-network

networks:
    shared-network:
        external: true
```

### 2.3. The Nginx Dockerfile (`Dockerfile.webserver`)

This creates the container that serves static files and proxies PHP requests.

```dockerfile
# IMPORTANT: Replace 'vml-erp-app:latest' with your final app image name if different,
# though using 'debian:bookworm-slim' or a similar base and installing Nginx is safer.
# Note: The original draft had an issue referencing a non-existent tag.
# Let's assume a clean image for Nginx.
FROM debian:bookworm-slim
 
# Install Nginx and cleanup
RUN apt-get update && apt-get install -y nginx \
    && rm -rf /var/lib/apt/lists/*

# Copy the Nginx config
COPY docker/nginx/conf.d/app.conf /etc/nginx/sites-enabled/default

# Set Nginx user to www-data (UID 33) to match PHP-FPM for file access.
RUN usermod -u 33 www-data \
    && sed -i 's/user nginx;/user www-data;/' /etc/nginx/nginx.conf

WORKDIR /var/www
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2.4. Nginx Configuration (`docker/nginx/conf.d/app.conf`)

Create the directory and the configuration file:

```bash
mkdir -p docker/nginx/conf.d
nano docker/nginx/conf.d/app.conf
```

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/public;
    # ... (standard Nginx configuration) ...

    # Pass PHP scripts to PHP-FPM
    location ~ \.php$ {
        include fastcgi_params;
        # 👈 CRITICAL: Use your app container name here!
        fastcgi_pass vml-erp:9000; 
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }
    # ... (rest of your provided static assets/logging blocks) ...
}
```

### 2.5. Configure `.env`

Update your project's **`.env`** file to connect to the shared database service:

```env
DB_CONNECTION=mysql
DB_HOST=mariadb # Use the container name of the shared service
DB_PORT=3306
DB_DATABASE=your_app_database 
DB_USERNAME=your_db_user      
DB_PASSWORD=your_db_password
```

-----

## 3\. Caddy Reverse Proxy Configuration 🔄

Caddy runs on the host, handles SSL automatically, and proxies traffic to the app's unique host port.

1.  **Edit your Caddyfile (as root):**

    ```bash
    sudo nano /etc/caddy/Caddyfile
    ```

2.  **Add the site block:**

    ```caddy
    yourapp.yourdomain.com {
        # Proxy to the Nginx container's exposed HOST port (e.g., 8001)
        reverse_proxy localhost:8001
    }

    # For the next app, change the domain and port:
    # anotherapp.yourdomain.com {
    #     reverse_proxy localhost:8002
    # }
    ```

3.  **Reload Caddy (as root):**

    ```bash
    sudo systemctl reload caddy
    ```

-----

## 4\. Full Deployment Workflow Summary ✅

This checklist summarizes the entire process for deploying a **new app**. Perform these steps as the **`dit`** user (except for Caddy configuration).

| Step | Action | Command | User |
| :--- | :--- | :--- | :--- |
| **1. Setup** | Clone project and navigate in. | `git clone <repo> my-new-app; cd my-new-app` | `dit` |
| **2. Config** | Create/Update Docker files and **`.env`** (with unique host port). | N/A | `dit` |
| **3. Build** | Build and start containers. | `docker-compose up -d --build` | `dit` |
| **4. Run** | Execute setup commands (migrations, key generation, etc.). | `docker-compose exec app php artisan migrate --seed` | `dit` |
| **5. Nginx** | Configure Caddy with the new domain and port. | `sudo nano /etc/caddy/Caddyfile` | `root` |
| **6. Finish** | Reload Caddy to apply changes. | `sudo systemctl reload caddy` | `root` |

### 💡 Running Artisan Commands

Use the `app` service name for all console tasks:

```bash
docker compose exec app php artisan [command]
```

This comprehensive, repeatable setup ensures a smooth, multi-project deployment with proper user permissions and a shared infrastructure\!
