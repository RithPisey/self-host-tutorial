Of course\! Deploying Laravel with Docker and using Caddy as a reverse proxy is a powerful and modern setup. Here's a step-by-step guide to get you up and running.

This approach will create a shared database container and then a separate, self-contained environment for each of your Laravel applications.

-----

### \#\# 1. Setting Up the Shared MariaDB Service  DATABASE

First, let's create a shared Docker network and launch the MariaDB container. This ensures all your Laravel applications can communicate with the same database instance securely.

1.  **Create a Docker Network:** This network will be used by MariaDB and all your future Laravel app containers.

    ```bash
    docker network create shared-services
    ```

2.  **Create a Directory for MariaDB Data:** We need to store the database data on the host machine so it persists even if the container is removed.

    ```bash
    # Create a directory to hold all your docker volumes
    sudo mkdir -p /opt/docker/mariadb/data
    ```

3.  **Create a `docker-compose.yml` for MariaDB:** In a convenient location (like `/opt/docker/mariadb/`), create a file named `docker-compose.yml`:

    ```yaml
    version: '3.8'

    services:
      mariadb:
        image: mariadb:11.2 # Using a specific version is good practice
        container_name: global-mariadb
        restart: unless-stopped
        environment:
          # IMPORTANT: Change these values!
          MARIADB_ROOT_PASSWORD: 'your_strong_root_password'
          MARIADB_DATABASE: 'your_first_laravel_db'
          MARIADB_USER: 'your_laravel_user'
          MARIADB_PASSWORD: 'your_strong_user_password'
        volumes:
          - ./data:/var/lib/mysql
        networks:
          - shared-services
        # DO NOT expose ports to the public internet unless you have a specific need.
        # Containers on the same network can communicate directly.

    networks:
      shared-services:
        external: true
    ```

4.  **Launch MariaDB:** Navigate to the directory containing this `docker-compose.yml` file and run:

    ```bash
    cd /opt/docker/mariadb
    docker compose up -d
    ```

Your shared MariaDB database is now running\! You can connect to it from any other container attached to the `shared-services` network using the hostname **`global-mariadb`**.

-----

### \#\# 2. Dockerizing Your Laravel Application 🚀

Now, for each Laravel project, you'll need to add a couple of files to "dockerize" it. Navigate to the root directory of one of your Laravel projects.

1.  **Create a `Dockerfile`:** This file defines the steps to build your application's image. It will use a multi-stage build to keep the final image lean, handling both the Node.js asset compilation and the PHP environment.

    Create a file named `Dockerfile` in your Laravel project's root:

    ```dockerfile
    # Stage 1: Build Node.js assets
    FROM node:22-alpine AS builder
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    # This command compiles your Vue/Inertia assets for production
    RUN npm run build

    # Stage 2: Create the final PHP production image
    FROM php:8.3-fpm-alpine AS final
    WORKDIR /var/www/html

    # Install system dependencies
    RUN apk add --no-cache \
        libzip-dev \
        zip \
        oniguruma-dev \
        libxml2-dev

    # Install common PHP extensions for Laravel
    RUN docker-php-ext-install \
        pdo_mysql \
        bcmath \
        pcntl \
        exif \
        zip \
        mbstring \
        gd \
        xml \
        sockets

    # Install Composer
    COPY --from=composer/latest /usr/bin/composer /usr/bin/composer

    # Copy application code and compiled assets from the builder stage
    COPY --from=builder /app /var/www/html

    # Install Composer dependencies
    RUN composer install --no-interaction --optimize-autoloader --no-dev

    # Set correct permissions for storage and cache
    RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
    RUN chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache

    # Expose port 9000 for PHP-FPM
    EXPOSE 9000

    # Start PHP-FPM
    CMD ["php-fpm"]
    ```

2.  **Create a `.dockerignore` file:** This prevents unnecessary files from being copied into your Docker image, making the build process faster and the image smaller.

    ```
    .git
    .github
    .env
    .env.example
    node_modules
    vendor
    storage
    public/storage
    docker-compose.yml
    Dockerfile
    README.md
    ```

3.  **Create a `docker-compose.yml` for the App:** This file will manage your application's service. Create a `docker-compose.yml` in the project root:

    ```yaml
    version: '3.8'

    services:
      app:
        build: . # Tells Docker to build the Dockerfile in the current directory
        container_name: my-first-app # Give each app a unique container name
        restart: unless-stopped
        volumes:
          # Mount the .env file from the host into the container
          - ./.env:/var/www/html/.env
        networks:
          - shared-services
        # Map container's port 9000 to host's port 9001 (use a different host port for each app)
        ports:
          - "127.0.0.1:9001:9000"

    networks:
      shared-services:
        external: true
    ```

    **Note:** For your second Laravel app, you would change `container_name` to `my-second-app` and the port mapping to `"127.0.0.1:9002:9000"`, and so on.

-----

### \#\# 3. Configuring Caddy as a Reverse Proxy 🔗

Caddy, which is already installed on your host OS, will act as the web server. It will serve static files directly and forward PHP requests to the correct Docker container.

1.  **Edit your `Caddyfile`:** This is typically located at `/etc/caddy/Caddyfile`.

2.  **Add a new site block for your application:**

    ```caddy
    # Add this block for your first Laravel application
    your-domain.com {
        # Set the web root to your project's public directory on the host
        root * /path/to/your/laravel/project/public

        # Enable compression
        encode zstd gzip

        # Handle PHP requests by forwarding them to the app container's mapped port
        # This points to localhost:9001, which we mapped in the app's docker-compose.yml
        php_fastcgi 127.0.0.1:9001

        # Serve static files directly
        file_server

        # Rewrite all other requests to index.php for Laravel's front-controller
        try_files {path} {path}/ /index.php?{query}
    }

    # For a second application, you would add another block:
    # another-domain.com {
    #     root * /path/to/your/second/laravel/project/public
    #     php_fastcgi 127.0.0.1:9002 # Points to the second app's mapped port
    #     # ... same file_server and try_files config
    # }
    ```

    Caddy will automatically handle provisioning and renewing SSL certificates for `your-domain.com`.

3.  **Reload Caddy:** After saving your `Caddyfile`, apply the changes.

    ```bash
    sudo systemctl reload caddy
    ```

-----

### \#\# 4. Deployment Workflow Summary ✅

Here is the complete workflow to deploy a new Laravel application:

1.  **Clone Your Project:** Clone your Laravel application from your git repository onto your VPS (e.g., into `/var/www/my-first-app`).

    ```bash
    cd /var/www
    git clone your-repository-url.git my-first-app
    cd my-first-app
    ```

2.  **Add Docker Files:** Add the `Dockerfile`, `.dockerignore`, and `docker-compose.yml` files as described in Step 2.

3.  **Configure `.env` file:** Copy `.env.example` to `.env` and configure it for production.

    ```bash
    cp .env.example .env
    nano .env
    ```

    **Most importantly**, set your database connection details:

    ```env
    DB_CONNECTION=mysql
    DB_HOST=global-mariadb  # <-- Use the container name
    DB_PORT=3306
    DB_DATABASE=your_first_laravel_db # The DB you created in Step 1
    DB_USERNAME=your_laravel_user   # The user you created in Step 1
    DB_PASSWORD=your_strong_user_password # The password you set in Step 1
    ```

4.  **Build and Run the Container:**

    ```bash
    docker compose up -d --build
    ```

5.  **Run Final Commands:** Execute database migrations and other necessary Artisan commands inside the container.

    ```bash
    docker compose exec app php artisan key:generate
    docker compose exec app php artisan storage:link
    docker compose exec app php artisan migrate --seed # optional
    docker compose exec app php artisan config:cache
    docker compose exec app php artisan route:cache
    docker compose exec app php artisan view:cache
    ```

6.  **Configure and Reload Caddy:** Add the site block to your `Caddyfile` and reload the Caddy service as shown in Step 3.

That's it\! Your Laravel application is now running in Docker, served securely by Caddy. You can repeat steps 2 through 4 for each additional Laravel application you want to deploy.
