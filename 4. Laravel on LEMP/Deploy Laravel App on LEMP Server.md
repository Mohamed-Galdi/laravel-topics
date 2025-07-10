# Table of Contents

1. [1. Preparing the LEMP stack server](#1-preparing-the-lemp-stack-server)

   - [a. L for Linux](#a-l-for-linux)
   - [b. E for NginX](#b-e-for-nginx)
   - [c. M for MySQL](#c-m-for-mysql)
   - [d. P for PHP](#d-p-for-php)
   - [e. Additional Requirements](#e-additional-requirements)

2. [2. Deploy the Laravel Project](#2-deploy-the-laravel-project)

   - [a. Prepare a Domain or Subdomain](#a-prepare-a-domain-or-subdomain)
   - [b. Clone the Laravel Project](#b-clone-the-laravel-project)
   - [c. Create a MySQL Database](#c-create-a-mysql-database)
   - [d. Set Directory Ownership](#d-set-directory-ownership)
   - [e. Configure the Environment File](#e-configure-the-environment-file)
   - [f. Run Laravel Setup Commands](#f-run-laravel-setup-commands)
   - [g. Build Frontend Assets](#g-build-frontend-assets)
   - [h. Configure Laravel Queues with Supervisor](#h-configure-laravel-queues-with-supervisor)
   - [i. Create an Nginx config file for the project](#i-create-an-nginx-config-file-for-the-project)
   - [j. Generate the SSL certificate using Certbot:](#j-generate-the-ssl-certificate-using-certbot)
   - [j. Set Proper Permissions for Laravel](#j-set-proper-permissions-for-laravel)

3. [3. Setting Up CI/CD for Laravel Deployment](#3-setting-up-cicd-for-laravel-deployment)

   - [Option 1: Simple Auto-Deploy with GitHub Webhook](#option-1-simple-auto-deploy-with-github-webhook)
   - [Option 2: Laravel Envoy + GitHub Actions](#option-2-laravel-envoy--github-actions)


In this comprehensive guide, we’ll walk through the process of deploying a Laravel project on a **LEMP stack**—a powerful and popular environment made up of **Linux**, **Nginx**, **MySQL**, and **PHP**. We’ll use a fresh **Ubuntu VPS from DigitalOcean** as our server, and cover everything from initial setup to production-ready deployment.

Along the way, you’ll learn:

- What the LEMP stack is and why it’s used for Laravel
  
- How to prepare your server with Nginx, MySQL, PHP, Composer, Node.js, and other essentials
  
- How to securely deploy your Laravel codebase
  
- How to configure your environment, file permissions, and storage
  
- And finally, how to set up a basic **CI/CD pipeline** to streamline future deployments
  

## 1. Preparing the LEMP stack server

### a. L for Linux

Linux is the foundation of the LEMP stack. The most commonly used distribution for servers is **Ubuntu**, thanks to its stability, security, and community support.

If you’re using [DigitalOcean](https://m.do.co/c/ea1b6a0cfa31), you can create a VPS (called a *Droplet*) in just a few steps:

- Choose a region
  
- Select **Ubuntu 22.04 LTS** as the image
  
- Pick your plan (CPU, RAM, storage)
  
- Set **SSH** as the authentication method
  

Once your droplet is created, connect to it via SSH as the root user:

```bash
ssh root@your_server_ip
```

To use a specific private key:

```bash
ssh -i ~/.ssh/your_key root@your_server_ip
```

If your key is password-protected, you’ll be prompted to enter it.

##### ✅ Create a Non-Root User

Running everything as `root` is dangerous—it has full access and can break things with one wrong command. Let’s create a safer user:

```bash
adduser your_name
```

Now grant the new user administrative (sudo) access:

```bash
usermod -aG sudo your_name
```

To allow the new user to connect via SSH:

```bash
su - your_name               # Switch to the new user
mkdir -p ~/.ssh              # Create .ssh directory
chmod 700 ~/.ssh             # Set proper permissions
nano ~/.ssh/authorized_keys  # Paste your public key here
chmod 600 ~/.ssh/authorized_keys
```

Now you can log in as the new user:

```bash
ssh your_name@your_server_ip
```

Once your new user can connect via SSH, it's a good idea to disable direct root login for added security:

1. Open the SSH configuration file:
  
  ```bash
  sudo nano /etc/ssh/sshd_config
  ```
  
2. Find the line:
  
  ```nginx
  PermitRootLogin yes
  ```
  
  And change it to:
  
  ```nginx
  PermitRootLogin no
  ```
  
3. Save and exit (`Ctrl + O`, then `Enter`, then `Ctrl + X`).
  
4. Apply the changes:
  
  ```bash
  sudo systemctl restart ssh
  ```
  

From now on, only non-root users with valid SSH keys will be able to connect. If anything goes wrong, you can still use the DigitalOcean console to regain access.

Ubuntu includes **UFW** (Uncomplicated Firewall) to help manage firewall rules easily. Before enabling it, allow SSH connections:

```bash
ufw allow OpenSSH
ufw enable
ufw status
```

This ensures you won’t get locked out when the firewall is turned on.

## b. E for NginX

**Nginx** (pronounced "Engine-X") is a high-performance web server that also acts as a **reverse proxy**, **load balancer**, and **HTTP cache**. In the LEMP stack, it’s responsible for handling web traffic and serving your Laravel application to users.

To install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

On most systems, Nginx will start automatically after installation. If not, you can start and enable it manually:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Since **UFW** (Uncomplicated Firewall) is enabled, you must allow web traffic:

```bash
sudo ufw app list                  # Show recognized services
sudo ufw allow 'Nginx Full'        # Allow HTTP (80) and HTTPS (443)
sudo ufw status                    # Check active rules
```

Now, visit your server's IP address in your browser. You should see the default Nginx welcome page.

##### 🌐 How Nginx Works with Laravel

Nginx receives HTTP requests from the browser and **forwards them to PHP-FPM**, which processes Laravel’s logic and returns a response. Nginx then serves that response to the browser. This flow is fast, secure, and scalable.

##### 📁 Understanding Nginx Directory Structure

In Ubuntu, web projects are typically stored in `/var/www/`. Nginx uses a modular configuration approach where each website gets its own configuration file. This makes managing multiple projects much easier.

The main Nginx configuration files are located in `/etc/nginx/`:

- **nginx.conf** - The main configuration file that defines global settings like worker processes, connection limits, and includes other configuration files
- **sites-available/** - Contains configuration files for all your websites (enabled or disabled)
- **sites-enabled/** - Contains symbolic links to active site configurations from sites-available

The `nginx.conf` file includes this important line: `include /etc/nginx/sites-enabled/*;` which tells Nginx to load all configuration files from the sites-enabled directory.

The sites-available/sites-enabled pattern isn't mandatory, but it's a best practice that allows you to:

- Keep all site configurations organized in sites-available
- Enable sites by creating symbolic links to sites-enabled
- Disable sites by removing their symbolic links (without deleting the configuration)

Here's an example Laravel site configuration:

```nginx
# this config file must be at: /etc/nginx/sites-availabe
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;
    root /var/www/laravel/public;

    index index.php index.html;

    # Handle Laravel's pretty URLs
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Process PHP files
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Deny access to .htaccess files
    location ~ /\.ht {
        deny all;
    }

    # Logging
    error_log /var/log/nginx/laravel_error.log;
    access_log /var/log/nginx/laravel_access.log;
}
```

To activate your Laravel config:

```bash
sudo ln -s /etc/nginx/sites-available/your_project /etc/nginx/sites-enabled/
sudo nginx -t    # Test the configuration
sudo systemctl reload nginx
```

Laravel requires specific file permissions for its storage and cache directories. The web server needs write access to these folders:

```bash
sudo chown -R www-data:www-data /var/www/laravel/storage
sudo chown -R www-data:www-data /var/www/laravel/bootstrap/cache
```

> `www-data` is the default user group under which Nginx (and most web services) run.

### c. M for **MySQL**

**MySQL** is a powerful open-source relational database management system (RDBMS). In a Laravel project, it stores and manages all your application’s data—such as users, posts, orders, and more.

Laravel uses **Eloquent ORM**, which makes it easy to interact with the database using PHP code instead of writing raw SQL queries. But before that, we need to install and configure MySQL on the server.

To install MySQL on Ubuntu:

```bash
sudo apt update
sudo apt install mysql-server -y
```

After installation, check that the service is running:

```bash
sudo systemctl status mysql
```

##### 🔐 Securing the MySQL Installation

MySQL includes a security script to set important defaults like root password, removing test users, and disabling remote root login:

```bash
sudo mysql_secure_installation
```

Follow the prompts:

- Set a root password (if not already set)
  
- Remove anonymous users
  
- Disallow remote root login
  
- Remove the test database
  
- Reload privilege tables
  

> 📝 If you're unsure about any step, choosing the recommended defaults is usually safe.

### 👤 Creating a Database and User for Your Project

Log in to the MySQL shell:

```bash
sudo mysql
```

Then run the following commands (replace with your actual values):

```sql
CREATE DATABASE your_project CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'project_user'@'localhost' IDENTIFIED BY 'strong_password';

GRANT ALL PRIVILEGES ON your_project.* TO 'project_user'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

This creates a dedicated database and user for your Laravel project.

### d. P for **PHP**

**PHP** is the programming language that powers Laravel. It runs behind the scenes to process requests, interact with the database, and return responses through Nginx.

Laravel requires **PHP 8.1 or higher** and several essential extensions. Let’s install PHP and everything Laravel needs to run smoothly.

First, install PHP and commonly required extensions:

```shell
sudo apt update
sudo apt install php php-fpm php-mysql php-curl php-mbstring php-xml php-bcmath php-zip php-cli php-common php-tokenizer unzip -y
```

**Explanation of some key extensions:**

- `php-mysql`: Required to connect to MySQL
  
- `php-mbstring`: Handles multi-byte strings (used by Laravel)
  
- `php-xml`: Required for tasks like config caching
  
- `php-curl`, `php-bcmath`, `php-zip`: Commonly needed by Laravel packages
  

After installation, verify the PHP version:

```bash
php -v
```

##### PHP-FPM Integration with Nginx

Nginx does **not** process PHP directly. Instead, it passes PHP requests to **PHP-FPM** (FastCGI Process Manager). This is already installed via `php-fpm`.

Make sure the service is running:

```shell
sudo systemctl status php8.3-fpm
```

> ⚠️ Replace `php8.3-fpm` with your installed version if it differs.

You’ve already configured Nginx to forward `.php` requests to PHP-FPM in the earlier step. At this point, Nginx and PHP are now working together.

##### 📦 Installing Composer

**Composer** is the official PHP dependency manager. Laravel and many PHP tools rely on it to install and manage packages.

Here’s how to install Composer globally:

```bash
cd ~
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'dac665fdc30fdd8ec78b38b9800061b4150413ff2e3b6f88543c636f7cd84f6db9189d43a81e5503cda447da73c7e5b6') { echo 'Installer verified'.PHP_EOL; } else { echo 'Installer corrupt'.PHP_EOL; unlink('composer-setup.php'); exit(1); }"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```

Now move it to a global location so you can use it system-wide:

```bash
sudo mv composer.phar /usr/local/bin/composer
```

Verify installation:

```bash
composer --version
```

### e. Additional Requirements

Besides the core LEMP stack, some additional tools are commonly required for a complete Laravel deployment. This includes tools for frontend asset building and securing your application with HTTPS.

##### 📦 Node.js & NPM (via NVM)

Laravel uses tools like **Vite**, **TailwindCSS** to compile frontend assets. These tools require **Node.js** and **npm**.

We’ll use **NVM (Node Version Manager)** to install Node.js in a flexible and upgradeable way.

1. Install NVM:
  
  ```bash
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
  ```
  
2. Load NVM into your current shell:
  
  ```bash
  export NVM_DIR="$HOME/.nvm"
  source "$NVM_DIR/nvm.sh"
  ```
  
3. Install the latest LTS version of Node.js:
  
  ```bash
  nvm install --lts
  ```
  
4. Confirm installation:
  
  ```bash
  node -v
  npm -v
  ```
  

##### 🔐 Certbot (Free SSL from Let’s Encrypt)

To secure your Laravel site with **HTTPS**, we’ll use **Certbot**, a free and automated tool from Let’s Encrypt.

Install Certbot and the Nginx plugin:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Once these additional tools are installed, your server is fully prepared to deploy and run a modern Laravel application, both backend and frontend, over a secure HTTPS connection

## 2. Deploy the Laravel Project

### a. Prepare a Domain or Subdomain

On your [DigitalOcean](https://m.do.co/c/ea1b6a0cfa31) dashboard, create a new **A record** pointing your domain or subdomain to your droplet's IP address.

### b. Clone the Laravel Project

SSH into your server and move into the web root directory:

```bash
cd /var/www
```

Then clone your project:

```bash
git clone git@github.com:your-username/your-repo.git myapp
```

If you're using SSH authentication, make sure to:

- Generate an SSH key (`ssh-keygen`)
  
- Add the public key to GitHub/GitLab under your account settings
  
- Start the SSH agent (`eval "$(ssh-agent -s)"`) and add your key (`ssh-add ~/.ssh/your_key`)
  

Disable Git pull rebase mode (optional):

```bash
git config pull.rebase false
```

### c. Create a MySQL Database

Log into MySQL, Then create a new database:

```sql
mysql -u myapp_user -p
 > show databases;
 > create database myapp;
 > exit;
```

If you encounter issues with the new database permissions for **myapp_user**, you may log in as root and grant the necessary permissions:

```sql
sudo mysql
 > grant all privileges on myapp.* to 'myapp_user'@'%';
 > flush privileges;
 > exit;
```

### d. Set Directory Ownership

Make sure the current user owns the project directory:

```bash
sudo chown -R $USER:$USER /var/www/myapp
```

### e. Configure the Environment File

Navigate into your project and copy the `.env` file:

```bash
cp .env.example .env
```

Update the following key variables:

- `APP_NAME` - Your application name
- `APP_ENV=production`
- `APP_DEBUG=false`
- `APP_URL` - Your domain URL
- `DB_DATABASE=myapp`
- `DB_USERNAME=myapp_user`
- `DB_PASSWORD` - Your database password

### f. Run Laravel Setup Commands

```bash
composer install --no-dev --optimize-autoloader
php artisan key:generate
php artisan storage:link
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan migrate:fresh --seed --force
```

### g. Build Frontend Assets

Install Node.js dependencies and build assets:

```bash
npm install
npm run build
```

Running heavy commands like `npm run build` can lead to memory issues if your server has low RAM (e.g., 1GB). A simple workaround is to stop other services to free up RAM temporarily, then restart them after the heavy task is complete.

1. **Stop Unnecessary Services**
  
  Use the following commands to stop services like Nginx and MySQL temporarily:
  
  ```bash
  sudo systemctl stop nginx
  sudo systemctl stop mysql
  ```
  
2. **Run the Heavy Command**
  
  Execute your memory-intensive task, such as:
  
  ```bash
  npm run build
  ```
  
3. **Restart the Stopped Services**
  
  Once the heavy command is complete, restart the services:
  
  ```bash
  sudo systemctl start nginx
  sudo systemctl start mysql
  ```
  

### h. Configure Laravel Queues with Supervisor

Laravel queues allow you to defer time-consuming tasks, improving application performance and responsiveness. We'll use Supervisor to manage queue workers.

**<u>Install Supervisor:</u>**

```shell
sudo apt-get update
sudo apt-get install supervisor
```

**<u>Create Supervisor Configuration:</u>**

```shell
sudo nano /etc/supervisor/conf.d/myapp-queue.conf
```

Paste the following configuration:

```ini
[program:myapp-queue]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/myapp/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/myapp/storage/logs/queue.log
stopwaitsecs=3600
```

💠 Configuration breakdown:

- `numprocs=2`: Runs two queue worker processes
- `--max-time=3600`: Restart worker every hour to prevent memory leaks
- `--tries=3`: Retry failed jobs up to 3 times
- Logs stored in project's storage/logs directory

**<u>Apply the configuration:</u>**

```shell
# Reread configurations
sudo supervisorctl reread

# Update and start the queue workers
sudo supervisorctl update

# Check status
sudo supervisorctl status myapp-queue
```

**<u>Optional: Laravel Horizon (for Redis Queues)</u>**

If you're using Redis for more advanced queue management:

```shell
# Install Horizon
composer require laravel/horizon

# Publish Horizon configuration
php artisan horizon:install

# Update Supervisor config to use Horizon instead
[program:myapp-horizon]
process_name=%(program_name)s
command=php /var/www/myapp/artisan horizon
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/myapp/storage/logs/horizon.log
```

💠 Best Practices:

- Monitor queue performance
- Handle failed jobs gracefully
- Use appropriate queue connections (sync, database, redis)
- Set reasonable job timeout and retry limits

**<u>Troubleshooting:</u>**

```shell
# View queue worker logs
tail -f /var/www/myapp/storage/logs/queue.log

# Restart specific queue workers
sudo supervisorctl restart myapp-queue:*
```

🚨 **Note**: Ensure your `.env` file has the correct `QUEUE_CONNECTION` set (e.g., `QUEUE_CONNECTION=database` or `redis`)

### i. Create an Nginx config file for the project

```
nano /etc/nginx/sites-available/myapp
```

There is a starter Nginx configuration for Laravel projects in the official documentation, or you can copy the config of an existing project, but be careful not to include the SSL config part.

**This is a generale example:**

```
server {
    server_name maypp.domain.com ;
    root /var/www/myapp/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Create a symbolic link to sites-enabled:

```
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled
```

Check nginx syntax and reload it:

```
sudo nginx -t
sudo systemctl restart nginx
```

### j. Generate the SSL certificate using Certbot:

```
sudo certbot --nginx --agree-tos --redirect --hsts --staple-ocsp -d myapp.domain.com --email you@email.com
```

Where:

- **--nginx**: Use the Nginx authenticator and installer
  
- **--agree-tos**: Agree to Let’s Encrypt terms of service
  
- **--redirect:** Enforce HTTPS by 301 redirect.
  
- **--hsts:** Add the Strict-Transport-Security header to every HTTP response.
  
- **--staple-ocsp**: Enables OCSP Stapling.
  
- **-d**: flag is followed by a list of domain names, separated by a comma. You can add up to 100 domain names.
  
- **--email:** Email used for registration and recovery contact.
  

### j. Set Proper Permissions for Laravel

Ensure the web server can write to `storage` and `bootstrap/cache`:

```
sudo chown -R www-data:www-data /var/www/myapp/storage
sudo chown -R www-data:www-data /var/www/myapp/bootstrap/cache
```

Optionally restart the php-fpm:

```
sudo systemctl restart php8.3-fpm
```

## 3. Setting Up CI/CD for Laravel Deployment

Manually SSH-ing into your server and pulling the latest changes every time you update your Laravel project isn’t scalable. With **CI/CD**, you can automate that.

We’ll use **GitHub Actions** and **Laravel Envoy** to build a lightweight and flexible CI/CD pipeline.

### Option 1: Simple Auto-Deploy with GitHub Webhook

**Best for:** Personal projects, small teams, basic auto-deploy

#### Step 1: Create a deploy script on your server

Create a Bash script on your server that pulls the latest changes and runs deployment commands.

```bash
sudo nano /var/www/myapp/deploy.sh
```

Paste:

```bash
#!/bin/bash

cd /var/www/myapp || exit

# Pull latest code
git pull origin main

# Run Laravel deployment tasks
composer install --no-interaction --prefer-dist --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
npm run build
```

Make it executable:

```bash
chmod +x /var/www/myapp/deploy.sh
```

#### Step 2: Create a webhook listener

Use a lightweight Node.js tool like [Webhook](https://github.com/adnanh/webhook) or a Laravel route that triggers the script. Example (for simplicity):

1. Add a route in `routes/web.php` (you can protect it with a token):
  
  ```php
  Route::post('/deploy', function () {
      $token = request('token');
      if ($token !== 'your-secret-token') {
          abort(403);
      }
  
      exec('/var/www/myapp/deploy.sh > /dev/null 2>&1 &');
  
      return 'Deployment triggered';
  });
  ```
  
2. From your GitHub repo settings, add a **webhook** to:
  

`https://your-domain.com/deploy?token=your-secret-token`

Set it to trigger on **push events**.

Done. Every push to your `main` branch triggers deployment.

### Option 2: Laravel Envoy + GitHub Actions

**Best for:** Cleaner automation, multi-step deployments, rollback support

#### Step 1: Install Laravel Envoy

Install globally on your machine:

```bash
composer global require laravel/envoy
```

Make sure `~/.composer/vendor/bin` is in your `PATH`.

#### Step 2: Create `Envoy.blade.php` in your Laravel project root

```php
@servers(['prod' => 'your_user@your_server_ip'])

@task('deploy', ['on' => 'prod'])
cd /var/www/myapp
git pull origin main
composer install --no-interaction --prefer-dist --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
npm run build
@endtask
```

Run manually with:

```bash
envoy run deploy
```

💡 You can secure the connection using SSH keys.

#### Step 3: Automate with GitHub Actions

In your Laravel repo, create a `.github/workflows/deploy.yml` file:

```yaml
name: Deploy Laravel App

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: SSH & run Envoy
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            envoy run deploy
```

Add these GitHub **Secrets** in your repo settings:

- `SSH_HOST`
  
- `SSH_USER`
  
- `SSH_PRIVATE_KEY`
  

## Conclusion

Deploying a Laravel application on a LEMP stack might seem daunting at first, but with a structured approach, it becomes a powerful and repeatable process. In this guide, you’ve learned how to:

- Set up a secure and optimized LEMP server
  
- Install all Laravel dependencies, tools, and services
  
- Deploy your Laravel project manually or with automation
  
- Configure SSL, queues, and frontend builds
  
- Set up a CI/CD pipeline to streamline future updates
  

With your server properly configured and a reliable deployment workflow in place, you’re now ready to launch, scale, and maintain Laravel projects with confidence.

**Next steps:**

- 🔐 Harden your server (fail2ban, SSH port changes, security updates)
  
- 📈 Monitor performance and logs
  
- 🚀 Add staging environments
  
- 🧪 Integrate testing in your CI pipeline
  

If you found this guide helpful, consider bookmarking it, sharing it, or customizing it for your own team documentation. Happy deploying!