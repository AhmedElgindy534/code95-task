# Task 2: LEMP Stack Setup on CentOS 8 with WordPress

This guide details the steps to set up a LEMP stack and run PHP-FPM as a user named `wptask` with `public_html` at `/home/wptask/public_html` on CentOS 8. The setup includes creating two virtual machines: one for the web server (Nginx and PHP) and the other for the database server (MySQL 8).

## Prerequisites
- Two virtual machines with CentOS 8 installed.
- Basic understanding of Linux command line.

## Virtual Machine 1: Web Server Setup

1. **Create User and Directory for Web Content:**
    ```sh
    sudo useradd -m -d /home/wptask wptask
    sudo passwd wptask
    sudo mkdir -p /home/wptask/public_html
    sudo chown -R wptask:wptask /home/wptask/public_html
    ```

2. **Install Nginx:**
    ```sh
    sudo dnf install -y epel-release
    sudo dnf install -y nginx
    sudo systemctl enable nginx
    sudo systemctl start nginx
    ```

3. **Install PHP and PHP-FPM:**
    ```sh
    sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
    sudo dnf module reset php
    sudo dnf module enable php:remi-8.2
    sudo dnf install -y php php-fpm php-mysqlnd
    sudo systemctl enable php-fpm
    sudo systemctl start php-fpm
    ```

4. **Configure PHP-FPM to Run as `wptask`:**
    Edit the PHP-FPM configuration file for the `www` pool:
    ```sh
    sudo vi /etc/php-fpm.d/www.conf
    ```
    Modify the following lines:
    ```conf
    user = wptask
    group = wptask
    listen.owner = nginx
    listen.group = nginx
    ```
    Restart PHP-FPM:
    ```sh
    sudo systemctl restart php-fpm
    ```

5. **Configure Nginx for WordPress:**
    Create a new Nginx server block:
    ```sh
    sudo vi /etc/nginx/conf.d/firstwebsite.com.conf
    ```
    Add the following configuration:
    ```conf
    server {
        listen 80;
        server_name firstwebsite.com;
        root /home/wptask/public_html;

        index index.php index.html index.htm;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/var/run/php-fpm/www.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
    ```
    Test Nginx configuration and restart:
    ```sh
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## Virtual Machine 2: Database Server Setup

1. **Install MySQL 8:**
    ```sh
    sudo dnf install -y @mysql:8.0
    sudo systemctl enable mysqld
    sudo systemctl start mysqld
    ```

2. **Secure MySQL Installation:**
    ```sh
    sudo mysql_secure_installation
    ```

3. **Create WordPress Database and User:**
    ```sql
    sudo mysql -u root -p

    CREATE DATABASE wordpress;
    CREATE USER 'wpuser'@'%' IDENTIFIED BY 'your_password';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'%';
    FLUSH PRIVILEGES;
    EXIT;
    ```

4. **Allow Remote Connections to MySQL:**
    Edit MySQL configuration:
    ```sh
    sudo vi /etc/my.cnf
    ```
    Add the following line under `[mysqld]`:
    ```conf
    bind-address = 0.0.0.0
    ```
    Restart MySQL:
    ```sh
    sudo systemctl restart mysqld
    ```

## Configure Web Server to Connect to Database Server

1. **Install WordPress:**
    Download and extract WordPress to `/home/wptask/public_html`:
    ```sh
    wget https://wordpress.org/latest.tar.gz
    tar -xzvf latest.tar.gz
    sudo mv wordpress/* /home/wptask/public_html
    sudo chown -R wptask:wptask /home/wptask/public_html
    ```

2. **Configure WordPress:**
    Copy and edit the sample configuration file:
    ```sh
    cp /home/wptask/public_html/wp-config-sample.php /home/wptask/public_html/wp-config.php
    vi /home/wptask/public_html/wp-config.php
    ```
    Modify the database settings:
    ```php
    define('DB_NAME', 'wordpress');
    define('DB_USER', 'wpuser');
    define('DB_PASSWORD', 'your_password');
    define('DB_HOST', 'IP_ADDRESS_OF_DB_SERVER');
    ```

## Final Steps

1. **Set SELinux to Permissive (if necessary):**
    ```sh
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

2. **Open Firewall Ports:**

    On the web server:
    ```sh
    sudo firewall-cmd --permanent --add-service=http
    sudo firewall-cmd --permanent --add-service=https
    sudo firewall-cmd --reload
    ```

    On the database server:
    ```sh
    sudo firewall-cmd --permanent --add-service=mysql
    sudo firewall-cmd --reload
    ```

3. **Access WordPress Setup:**
    Open a web browser and navigate to `http://firstwebsite.com` to complete the WordPress installation.

Following these steps, you will have a working LEMP stack with WordPress running under the `wptask` user, with the web server and database server on separate virtual machines.
