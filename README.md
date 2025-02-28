# WordPress + Pimcore DDEV Environment

This project provides a complete development environment for running both WordPress and Pimcore in a single DDEV setup. WordPress is accessible at the main domain, while Pimcore runs on a subdomain.

## Requirements

- [DDEV](https://ddev.readthedocs.io/en/stable/) (v1.22.0 or higher)
- [Docker](https://www.docker.com/products/docker-desktop/)
- [Composer](https://getcomposer.org/) (v2.0 or higher)

## Quick Start

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd pimcore-ddev
   ```

2. Start DDEV:
   ```bash
   ddev start
   ```

3. Install dependencies:
   ```bash
   ddev composer install
   ```

4. Access your sites:
   - WordPress: https://pimcore-ddev.ddev.site
   - Pimcore: https://pimcore.pimcore-ddev.ddev.site

## Detailed Setup

### 1. DDEV Configuration

The project uses DDEV for local development. The configuration includes:

- WordPress running on the main domain
- Pimcore running on a subdomain
- MariaDB 10.11 for the database
- PHP 8.2 for both applications

### 2. WordPress Setup

WordPress is installed in the `web` directory and is accessible at the main domain. The admin credentials are:

- Username: `admin`
- Password: `admin`

To access the WordPress admin panel, go to https://pimcore-ddev.ddev.site/wp-admin

### 3. Pimcore Setup

Pimcore is installed in the `pimcore` directory and is accessible at the subdomain. The setup includes:

#### Configuration Files

1. Docker Compose Configuration (`.ddev/docker-compose.pimcore.yaml`):
```yaml
services:
  pimcore:
    container_name: ddev-${DDEV_SITENAME}-pimcore
    image: pimcore/pimcore:php8.2-latest
    restart: always
    environment:
      - PIMCORE_INSTALL_ADMIN_USERNAME=admin
      - PIMCORE_INSTALL_ADMIN_PASSWORD=admin
      - PIMCORE_INSTALL_MYSQL_HOST_SOCKET=db:3306
      - PIMCORE_INSTALL_MYSQL_USERNAME=db
      - PIMCORE_INSTALL_MYSQL_PASSWORD=db
      - PIMCORE_INSTALL_MYSQL_DATABASE=pimcore
      - APP_ENV=dev
      - APP_DEBUG=1
    volumes:
      - type: bind
        source: ../pimcore
        target: /var/www/html
      - type: bind
        source: ./.pimcore-nginx.conf
        target: /etc/nginx/conf.d/default.conf
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${DDEV_SITENAME}-pimcore.rule=Host(`pimcore.${DDEV_SITENAME}.ddev.site`)"
      - "traefik.http.services.${DDEV_SITENAME}-pimcore.loadbalancer.server.port=80"
```

2. Nginx Configuration (`.ddev/.pimcore-nginx.conf`):
```nginx
server {
    listen 80;
    server_name pimcore.pimcore-ddev.ddev.site;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Security configurations
    location ~ ^/(\.env|composer\.(json|lock)|package(-lock)?\.json)$ {
        deny all;
        return 404;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
        access_log off;
        add_header Cache-Control "public, no-transform";
    }

    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

#### Initial Setup

The Pimcore installation is automatically configured during `ddev start`. The setup includes:

1. Database creation and configuration
2. Admin user creation with default credentials:
   - Username: `admin`
   - Password: `admin`
3. Proper file permissions setup
4. Nginx configuration for web access

#### Access Points

- Admin Interface: https://pimcore.pimcore-ddev.ddev.site/admin
- Frontend: https://pimcore.pimcore-ddev.ddev.site

#### Troubleshooting

If you encounter permission issues (500 errors related to cache), run:
```bash
ddev exec -s pimcore "chmod -R 777 /var/www/html/var"
```

To reinstall Pimcore (if needed):
```bash
ddev exec -s pimcore "cd /var/www/html && ./vendor/bin/pimcore-install --admin-username=admin --admin-password=admin --mysql-host-socket=db:3306 --mysql-username=db --mysql-password=db --mysql-database=pimcore --no-interaction"
```

### 4. Database Management

Both WordPress and Pimcore databases are automatically created and configured during `ddev start`:

- WordPress database: `db`
- Pimcore database: `pimcore`

The databases are automatically:
- Created if they don't exist
- Configured with proper permissions
- Available to both applications

You can access the databases using:

```bash
ddev mysql               # Access MySQL as the default 'db' user
ddev mysql -uroot -proot # Access MySQL as root
```

Or import a database dump:

```bash
ddev import-db --target-db=db --src=path/to/wordpress-dump.sql
ddev import-db --target-db=pimcore --src=path/to/pimcore-dump.sql
```

## Managing Dependencies

This project uses Composer to manage dependencies for both WordPress and Pimcore. The main `composer.json` file in the root directory handles all dependencies.

### Adding WordPress Plugins

```bash
ddev composer require wpackagist-plugin/plugin-name
```

### Adding Pimcore Bundles

```bash
ddev composer require pimcore/bundle-name
```

### Updating Dependencies

```bash
ddev composer update
```

## Project Structure

```
├── .ddev/                  # DDEV configuration
├── pimcore/                # Pimcore installation
├── web/                    # WordPress installation
├── vendor/                 # Shared vendor directory
├── composer.json           # Dependency management
└── README.md               # This file
```

## Common Tasks

### Restart the Environment

```bash
ddev restart
```

### SSH into Containers

```bash
ddev ssh                    # Web container (WordPress)
ddev ssh -s pimcore         # Pimcore container
ddev ssh -s db              # Database container
```

### View Logs

```bash
ddev logs
```

## Customizing the Setup

### Changing Domain Names

1. Edit `.ddev/config.yaml` to update the project name and additional FQDNs
2. Edit `.ddev/nginx_full/pimcore.conf` to update the server_name
3. Edit `.ddev/docker-compose.pimcore.yaml` to update the traefik labels
4. Run `ddev restart`

### Adding Custom PHP Extensions

Edit `.ddev/config.yaml` to add custom PHP extensions:

```yaml
webimage_extra_packages:
  - php8.2-extension-name
```

## Troubleshooting

### Fixing Permissions

If you encounter permission issues with Pimcore:

```bash
ddev exec -s pimcore "chmod -R 777 /var/www/html/var"
```

### Clearing Caches

For WordPress:
```bash
ddev wp cache flush
```

For Pimcore:
```bash
ddev exec -s pimcore "cd /var/www/html && bin/console cache:clear"
```

## License

This project is licensed under the MIT License - see the LICENSE file for details. 