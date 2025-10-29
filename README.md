Of course\! Deploying a Laravel application using Docker with Caddy as a reverse proxy is a powerful and modern setup. I'll guide you through the process, focusing on your specific requirements: a shared MariaDB container and support for multiple Laravel projects owned by a non-root user.

Here’s the complete plan:

1.  **Set Up a Shared Docker Network and MariaDB:** Create a persistent, shared database container that all your Laravel apps can connect to.
2.  **Dockerize a Laravel Application:** Create a `Dockerfile` and `docker-compose.yml` for a single Laravel project. This will serve as your template.
3.  **Configure Caddy as a Reverse Proxy:** Point a domain to your Dockerized application.
4.  **Deployment Workflow:** A step-by-step guide from `git clone` to a live site.

-----

### ✅ Prerequisite: User Permissions

You mentioned that the `dit` user owns the project files. To allow `dit` to run Docker commands without using `sudo`, you need to add this user to the `docker` group.

Run this command as `root` (or with `sudo`):

```bash
sudo usermod -aG docker dit
```

**Important:** The user `dit` must **log out and log back in** for this change to take effect.

-----

### \#\# Step 1: Create the Shared MariaDB Service MariaDB

First, we'll create a central MariaDB container. This setup will live in its own directory and use its own `docker-compose.yml`. This keeps it separate from your applications.

1.  **Create a directory for the service:**
    As the `root` user, create a directory to manage this shared service.

    ```bash
    mkdir -p /var/www/mariadb
    cd /var/www/mariadb
    ```

2.  **Create the `docker-compose.yml` file:**
    Create a file named `docker-compose.yml` in this directory:

    ```bash
    nano docker-compose.yml
    ```

    Paste the following configuration into the file:

    ```yaml
    services:
          mariadb:
            image: mariadb:latest
            container_name: mariadb
            restart: always
            environment:
              MYSQL_ROOT_PASSWORD: N8xIyxIC9QFx
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

3.  **Start the MariaDB container:**
    From within the `/ver/www/mariadb` directory, run:

    ```bash
    docker-compose up -d
    ```

Now you have a MariaDB container running, and it has created a network called `shared-network`. All your future Laravel applications will connect to this network to communicate with the database using the hostname `mariadb`.

-----

### \#\# Step 2: Dockerize Your Laravel Application 🐳

Now, let's prepare a Laravel project to run in Docker. You will do these steps for **each** Laravel project you want to deploy.

Let's assume your project is located at `/home/dit/my-laravel-app`. Navigate there:

```bash
cd /home/dit/my-laravel-app
```

1.  **Create a `Dockerfile`:**
    This file defines the environment for your PHP application. It will install PHP, extensions, Composer, and Node.js.

    ```bash
    nano Dockerfile
    ```

    Paste the following. This is a robust template that works for Laravel 10-12 with Node 22 and PHP 8.3.

    ```dockerfile
    
        # ---- Stage 1: The "Builder" Stage ----
        # This stage installs all tools, downloads dependencies, and builds assets.
        FROM php:8.3-fpm AS builder
        
        WORKDIR /var/www
        
        # Install all necessary dependencies FOR BUILDING
        RUN apt-get update && apt-get install -y --no-install-recommends \
            git \
            curl \
            zip \
            unzip \
            libzip-dev \
            libpng-dev \
            libjpeg-dev \
            libfreetype6-dev \
            libonig-dev \
            libwebp-dev \
            && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
            && apt-get install -y nodejs \
            && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip \
            && curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
            && apt-get clean && rm -rf /var/lib/apt/lists/*
        
        # Copy composer files and install PHP dependencies (production only)
        COPY composer.json composer.lock ./
        RUN composer install --no-dev --no-interaction --no-scripts --prefer-dist --optimize-autoloader
        
        # Copy package files, install dependencies, and build frontend assets
        COPY package.json package-lock.json ./
        RUN npm install
        COPY . .
        RUN npm run build
        
        
        # ---- Stage 2: The Final "Production" Stage ----
        # This stage is lean and only contains what's needed to RUN the app.
        FROM php:8.3-fpm
        
        WORKDIR /var/www
        
        # Install only the required RUNTIME PHP extensions and system libraries
        RUN apt-get update && apt-get install -y --no-install-recommends \
            libzip-dev \
            libpng-dev \
            libjpeg-dev \
            libfreetype6-dev \
            libonig-dev \
            libwebp-dev \
            && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip \
            && apt-get clean && rm -rf /var/lib/apt/lists/*
        
        # Copy the built application code and dependencies from the builder stage
        COPY --from=builder /var/www .
        
        # Set correct permissions for the runtime user (www-data)
        RUN mkdir -p /var/www/bootstrap/cache \
            && mkdir -p /var/www/storage/framework/views \
            && chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache \
            && chmod -R 775 /var/www/storage /var/www/bootstrap/cache
        
        # Expose port 9000 and start the server
        EXPOSE 9000
        CMD ["php-fpm"]

    
    ```

2.  **Create a Project-Specific `docker-compose.yml`:**
    This file will define the services for *this specific application*: the PHP app itself and a webserver (Nginx) to serve it.

    ```bash
    nano docker-compose.yml
    ```

    Paste this configuration:

    ```yaml
    version: '3.8'

    services:
      # PHP-FPM Application Service
      app:
        build:
          context: .
          dockerfile: Dockerfile
        container_name: my-laravel-app
        restart: unless-stopped
        volumes:
          - ./:/var/www
        networks:
          - shared-network # Connect to the same network as MariaDB

      # Nginx Webserver Service
      webserver:
        image: nginx:alpine
        container_name: my-laravel-app-webserver
        restart: unless-stopped
        ports:
          # Map a UNIQUE host port to the container's port 80
          - "8001:80"
        volumes:
          - ./:/var/www
          - ./docker/nginx/conf.d:/etc/nginx/conf.d/
        networks:
          - shared-network

    # Define the external network
    networks:
      shared-network:
        external: true
    ```

    **Note:** For your next app, change the port from `"8001:80"` to `"8002:80"`, and so on. Each app needs a unique host port.

3.  **Create the Nginx Configuration:**
    Nginx will run in its own container and pass PHP requests to your `app` container.

    ```bash
    ```

sh
mkdir -p docker/nginx/conf.d
nano docker/nginx/conf.d/app.conf
\`\`\`
Paste this standard Laravel Nginx config:

````
```nginx
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000; # 'app' is the name of our PHP service in docker-compose.yml
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```
````

4.  **Configure Your Laravel `.env` File:**
    Make sure your project's `.env` file is configured to connect to the Docker database.
    ```env
    DB_CONNECTION=mysql
    DB_HOST=mariadb # The container name of our MariaDB service
    DB_PORT=3306
    DB_DATABASE=your_app_database # Create this DB inside MariaDB
    DB_USERNAME=your_db_user      # Create this user
    DB_PASSWORD=your_db_password
    ```

-----

### \#\# Step 3: Configure Caddy as Reverse Proxy 🔄

Now, we'll tell Caddy (running on the host machine) to forward traffic from a domain to your new Docker container.

1.  **Edit your Caddyfile:**
    As the `root` user, edit the main Caddy configuration file.

    ```bash
    sudo nano /etc/caddy/Caddyfile
    ```

2.  **Add a new site block:**
    Add the following block to the file. Caddy will automatically provision an SSL certificate for you.

    ```caddy
    yourapp.yourdomain.com {
        # Reverse proxy requests to the Nginx container's exposed port
        reverse_proxy localhost:8001
    }

    # If you deploy a second app on port 8002, you would add:
    # anotherapp.yourdomain.com {
    #     reverse_proxy localhost:8002
    # }
    ```

3.  **Reload Caddy:**
    After saving the file, apply the changes by reloading Caddy.

    ```bash
    sudo systemctl reload caddy
    ```

-----

### \#\# 🚀 To run php artisan command
```docker compose exec app php artisan [command]```

### ## Your Deployment Checklist ✅

Think of it as a standard step in your deployment process for any new Laravel app. Your workflow will look like this:

1.  `git clone <new-project-repo>`
2.  `cd <new-project-directory>`
3.  Set up your `.env` file and Docker files (`Dockerfile`, `docker-compose.yml`, etc.).
4.  Run `docker compose up -d --build` to start the containers.
5.  Run `docker compose exec app composer install` to create the `vendor` directory.
6.  **Run `sudo chown -R www-data:www-data storage bootstrap/cache` to fix permissions.**
7.  Run any other necessary commands like `docker compose exec app php artisan migrate`, `docker compose exec app php artisan key:generate`, etc.
8.  Configure Caddy to point to the new app's port.

### \#\# 🚀 Full Deployment Workflow Summary

Here is the complete process for a new app, performed as the `dit` user (except for the Caddy steps):

1.  **Clone Project:** `git clone <your-repo-url> my-new-app`
2.  **Navigate to Project:** `cd my-new-app`
3.  **Add Docker Files:** Create the `Dockerfile`, `docker-compose.yml`, and the `docker/nginx/conf.d/app.conf` files as shown in Step 2. Remember to pick a **unique host port** in `docker-compose.yml`.
4.  **Configure Environment:** Copy `.env.example` to `.env` and fill in your database credentials and other app settings.
5.  **Build and Start Containers:** `docker-compose up -d --build`
6.  **Run Database Migrations:** `docker-compose exec app php artisan migrate --seed`
7.  **Configure Caddy (as root):** Edit `/etc/caddy/Caddyfile` to add the new domain and `reverse_proxy` directive pointing to the unique port you chose.
8.  **Reload Caddy (as root):** `sudo systemctl reload caddy`

Your application is now live\! You can repeat this process for all your Laravel applications, ensuring each one uses a different host port for its `webserver` service.

