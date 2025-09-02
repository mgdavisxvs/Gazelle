# Gazelle Setup Guide

This guide provides step-by-step instructions to get Gazelle running on your system. Gazelle is a web framework for private BitTorrent trackers, written in PHP and MySQL.

## System Requirements

### Minimum Requirements
- **Operating System**: Linux (Ubuntu 18.04+, CentOS 7+, Debian 9+) or macOS
- **PHP**: Version 5.4 or newer (PHP 7.4+ recommended)
- **MySQL/MariaDB**: Version 5.6+ (MySQL 8.0 or MariaDB 10.3+ recommended)
- **Web Server**: Nginx (recommended) or Apache
- **Memory**: At least 2GB RAM (4GB+ recommended)
- **Storage**: 10GB+ free disk space

### Required Dependencies
- **PHP Extensions**: mysqli, memcached, curl, json, mbstring, gd
- **Memcached**: In-memory caching system
- **Sphinx Search**: Full-text search engine (version 2.0.6+)
- **Git**: Version control system
- **Ocelot**: BitTorrent tracker (included as tarball)

### Optional but Recommended
- **GCC/G++**: For compiling Ocelot (version 4.7+)
- **Boost Libraries**: For Ocelot compilation (version 1.55.0+)
- **Nginx**: High-performance web server
- **fail2ban**: Intrusion prevention
- **UFW/iptables**: Firewall

## Step 1: System Preparation

### On Ubuntu/Debian:
```bash
# Update package list
sudo apt update

# Install basic dependencies
sudo apt install -y git curl wget unzip build-essential

# Install PHP and extensions
sudo apt install -y php php-fpm php-mysql php-memcached php-curl php-json php-mbstring php-gd php-cli php-dev

# Install MySQL/MariaDB
sudo apt install -y mariadb-server mariadb-client

# Install Memcached
sudo apt install -y memcached

# Install Nginx
sudo apt install -y nginx

# Install Sphinx Search
sudo apt install -y sphinxsearch

# Install development tools for Ocelot
sudo apt install -y libboost-dev libboost-system-dev libboost-iostreams-dev libmysqlclient-dev
```

### On CentOS/RHEL:
```bash
# Enable EPEL repository
sudo yum install -y epel-release

# Install basic dependencies
sudo yum install -y git curl wget unzip gcc gcc-c++ make

# Install PHP and extensions
sudo yum install -y php php-fpm php-mysql php-memcached php-curl php-json php-mbstring php-gd php-cli php-devel

# Install MySQL/MariaDB
sudo yum install -y mariadb-server mariadb

# Install Memcached
sudo yum install -y memcached

# Install Nginx
sudo yum install -y nginx

# Install Sphinx Search
sudo yum install -y sphinx

# Install development tools for Ocelot
sudo yum install -y boost-devel mysql-devel
```

## Step 2: Download and Setup Gazelle

```bash
# Clone the Gazelle repository
git clone https://github.com/yourusername/Gazelle.git /var/www/gazelle
cd /var/www/gazelle

# Set proper permissions
sudo chown -R www-data:www-data /var/www/gazelle
sudo chmod -R 755 /var/www/gazelle
sudo chmod -R 777 /var/www/gazelle/static/styles/
sudo chmod 777 /var/www/gazelle/captcha/
```

## Step 3: Database Setup

### Start and Secure MySQL/MariaDB
```bash
# Start MariaDB service
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Secure the installation
sudo mysql_secure_installation
```

### Create Gazelle Database
```bash
# Login to MySQL as root
mysql -u root -p

# Create database and user
CREATE DATABASE gazelle CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'gazelle'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON gazelle.* TO 'gazelle'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# Import the database schema
mysql -u gazelle -p gazelle < /var/www/gazelle/gazelle.sql
```

## Step 4: Configuration Setup

### Create Configuration File
```bash
# Copy the template configuration
cd /var/www/gazelle
cp classes/config.template classes/config.php

# Edit the configuration file
nano classes/config.php
```

### Required Configuration Changes

Edit `/var/www/gazelle/classes/config.php` and set these values:

```php
// Site settings
define('SITE_NAME', 'Your Tracker Name');
define('NONSSL_SITE_URL', 'yourdomain.com');
define('SSL_SITE_URL', 'yourdomain.com'); 
define('SITE_IP', 'YOUR_SERVER_IP');
define('SERVER_ROOT', '/var/www/gazelle');

// Generate random keys (32+ characters each)
define('ENCKEY', 'your_random_encryption_key_here');
define('SITE_SALT', 'your_random_site_salt_here'); 
define('SCHEDULE_KEY', 'your_random_schedule_key_here');
define('RSS_HASH', 'your_random_rss_hash_here');

// MySQL settings
define('SQLLOGIN', 'gazelle');
define('SQLPASS', 'your_secure_password');

// Tracker settings
define('TRACKER_SECRET', 'your_32_character_tracker_secret'); 
define('TRACKER_REPORTKEY', 'your_32_character_report_key');
```

**Security Note**: Use https://grc.com/passwords.html to generate secure random passwords and keys.

## Step 5: Setup Memcached

```bash
# Start Memcached with Unix socket
sudo systemctl stop memcached

# Create memcached socket directory
sudo mkdir -p /var/run/memcached
sudo chown memcache:memcache /var/run/memcached

# Start memcached with socket
sudo -u memcache memcached -d -m 512 -s /var/run/memcached.sock -a 0777 -t4 -C

# Enable memcached to start on boot
sudo systemctl enable memcached
```

## Step 6: Setup Sphinx Search

### Configure Sphinx
```bash
# Copy Sphinx configuration
sudo cp /var/www/gazelle/sphinx.conf /etc/sphinxsearch/sphinx.conf

# Edit Sphinx configuration
sudo nano /etc/sphinxsearch/sphinx.conf
```

Update the MySQL connection details in `/etc/sphinxsearch/sphinx.conf`:
```
sql_user = gazelle
sql_pass = your_secure_password
```

### Create Sphinx directories and build indexes
```bash
# Create Sphinx directories
sudo mkdir -p /var/lib/sphinxsearch/data
sudo mkdir -p /var/log/sphinxsearch
sudo chown -R sphinxsearch:sphinxsearch /var/lib/sphinxsearch
sudo chown -R sphinxsearch:sphinxsearch /var/log/sphinxsearch

# Build initial indexes
sudo indexer -c /etc/sphinxsearch/sphinx.conf --all

# Start Sphinx daemon
sudo systemctl start sphinxsearch
sudo systemctl enable sphinxsearch
```

## Step 7: Setup Ocelot Tracker

### Extract and Compile Ocelot
```bash
cd /var/www/gazelle

# Extract the latest Ocelot version
tar -xzf ocelot-1.0.tar.gz
cd ocelot-1.0

# Configure Ocelot
cp config.cpp.example config.cpp
nano config.cpp
```

Update these settings in `config.cpp`:
```cpp
std::string mysql_username = "gazelle";
std::string mysql_password = "your_secure_password";
std::string site_password = "your_32_character_tracker_secret";
std::string report_password = "your_32_character_report_key";
```

### Compile and Install Ocelot
```bash
# Compile Ocelot
make

# Copy binary to system location
sudo cp ocelot /usr/local/bin/

# Create ocelot user
sudo useradd -r -s /bin/false ocelot

# Create systemd service
sudo tee /etc/systemd/system/ocelot.service > /dev/null <<EOF
[Unit]
Description=Ocelot BitTorrent Tracker
After=network.target mysql.service

[Service]
Type=forking
User=ocelot
WorkingDirectory=/var/www/gazelle/ocelot-1.0
ExecStart=/usr/local/bin/ocelot
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start Ocelot
sudo systemctl daemon-reload
sudo systemctl start ocelot
sudo systemctl enable ocelot
```

## Step 8: Web Server Configuration

### Nginx Configuration
Create `/etc/nginx/sites-available/gazelle`:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    root /var/www/gazelle;
    index index.php;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    # PHP handling
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Static files
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }

    location ~ /(classes|docs|templates)/ {
        deny all;
    }
}
```

### Enable the site
```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/gazelle /etc/nginx/sites-enabled/

# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

## Step 9: Setup Cron Jobs

```bash
# Edit crontab for www-data user
sudo crontab -u www-data -e

# Add these cron jobs (replace SCHEDULE_KEY with your actual key)
0,15,30,45 * * * * /usr/bin/php /var/www/gazelle/schedule.php YOUR_SCHEDULE_KEY >> /var/log/gazelle-schedule.log 2>&1
10,25,40,55 * * * * /usr/bin/php /var/www/gazelle/peerupdate.php YOUR_SCHEDULE_KEY >> /var/log/gazelle-peerupdate.log 2>&1
* * * * * /usr/bin/indexer -c /etc/sphinxsearch/sphinx.conf --rotate delta requests_delta log_delta >/dev/null 2>&1
5 0,12 * * * /usr/bin/indexer -c /etc/sphinxsearch/sphinx.conf --rotate --all >> /var/log/sphinx-indexer.log 2>&1
```

## Step 10: Final Setup and First User

### Create log directories
```bash
sudo mkdir -p /var/log/gazelle
sudo chown www-data:www-data /var/log/gazelle
```

### Access your Gazelle site
1. Open your web browser and navigate to `http://yourdomain.com`
2. You should see the Gazelle homepage
3. Click "Register" to create the first user account
4. **Important**: The first user registered will automatically become a System Operator (SysOp)

### Initial Site Configuration
1. Login with your SysOp account
2. Visit `/tools.php?action=update_geoip` to populate GeoIP data
3. Configure site settings as needed through the admin panel

## Step 11: SSL Setup (Recommended)

### Using Let's Encrypt
```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Obtain SSL certificate
sudo certbot --nginx -d yourdomain.com

# Test auto-renewal
sudo certbot renew --dry-run
```

## Troubleshooting

### Common Issues

**1. Permission Errors**
```bash
sudo chown -R www-data:www-data /var/www/gazelle
sudo chmod -R 755 /var/www/gazelle
```

**2. MySQL Connection Issues**
- Verify database credentials in `classes/config.php`
- Check if MySQL service is running: `sudo systemctl status mariadb`

**3. Memcached Connection Issues**
- Check if Memcached is running: `sudo systemctl status memcached`
- Verify socket path in config: `/var/run/memcached.sock`

**4. Sphinx Search Not Working**
- Check Sphinx status: `sudo systemctl status sphinxsearch`
- Rebuild indexes: `sudo indexer -c /etc/sphinxsearch/sphinx.conf --all`

**5. Tracker Issues**
- Check Ocelot status: `sudo systemctl status ocelot`
- Verify tracker settings match between `config.php` and Ocelot's `config.cpp`

**6. PHP Errors**
- Check PHP error logs: `tail -f /var/log/nginx/error.log`
- Verify PHP extensions are installed: `php -m`

### Log Locations
- Nginx: `/var/log/nginx/`
- PHP-FPM: `/var/log/php7.4-fpm.log`
- MySQL: `/var/log/mysql/`
- Gazelle Schedule: `/var/log/gazelle-schedule.log`
- Gazelle Peer Update: `/var/log/gazelle-peerupdate.log`
- Sphinx: `/var/log/sphinx-indexer.log`

### Getting Help
- Check the logs for detailed error messages
- Ensure all services are running: `sudo systemctl status nginx php7.4-fpm mariadb memcached sphinxsearch ocelot`
- Verify your configuration files for typos and correct paths

## Security Considerations

1. **Change Default Passwords**: Use strong, unique passwords for all services
2. **Firewall**: Configure UFW/iptables to only allow necessary ports
3. **SSL**: Use HTTPS in production
4. **Updates**: Keep your system and dependencies updated
5. **Backup**: Regularly backup your database and configuration files
6. **Monitoring**: Set up log monitoring and alerting

## Performance Optimization

1. **PHP OPcache**: Enable PHP OPcache for better performance
2. **MySQL Tuning**: Optimize MySQL configuration for your hardware
3. **Nginx Caching**: Configure Nginx caching for static content
4. **Memcached**: Increase Memcached memory allocation as needed
5. **Monitoring**: Use tools like htop, iotop, and MySQL slow query log

---

**Note**: This guide assumes a production setup. For development, consider using the VagrantGazelle mentioned in the README.md for easier setup and testing.