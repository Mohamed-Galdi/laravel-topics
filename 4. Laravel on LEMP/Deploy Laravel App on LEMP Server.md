# Table of Contents

1. [Prerequisites](#1-prerequisites)

2. [Steps](#2-steps)
   - [1. Prepare Domain](#1%EF%B8%8F%E2%83%A3-prepare-a-domain-or-sub-domain-for-the-project)
   - [2. Create Production Branch](#2%EF%B8%8F%E2%83%A3-create-prod-branch-on-the-project-remote-repository)
   - [3. Clone Project](#3%EF%B8%8F%E2%83%A3-on-the-server-cd-to-varwww-clone-the-project-and-checkout-to-prod-branch)
   - [4. Create Database](#4%EF%B8%8F%E2%83%A3-create-a-new-database-for-that-project)
   - [5. Set Folder Ownership](#5%EF%B8%8F%E2%83%A3-assign-ownership-of-the-project-folder-to-the-current-user)
   - [6. Configure Environment](#6%EF%B8%8F%E2%83%A3-create-the-env-file-and-fill-in-the-configurations)
   - [7. Run Laravel Setup](#7%EF%B8%8F%E2%83%A3-now-navigate-to-the-project-folder-and-run-the-laravel-setup-commands)
   - [8. Build Frontend](#8%EF%B8%8F%E2%83%A3-install-front-end-dependencies-and-build-assets)
   - [9. Configure Nginx](#9%EF%B8%8F%E2%83%A3-create-an-nginx-config-file-for-the-project)
   - [10. SSL Setup](#1%EF%B8%8F⃣0%EF%B8%8F⃣--generate-the-ssl-certificate-using-certbot)
   - [11. Set Permissions](#1%EF%B8%8F%E2%83%A31%EF%B8%8F%E2%83%A3-give-the-web-server-user-write-access-to-storage-and-cache)

3. [Auto-Deploy Workflow](#3-auto-deploy-workflow)
   - [1. Add SSH Key](#1%EF%B8%8F%E2%83%A3-add-ssh-key-to-github)
   - [2. Create Deploy Script](#2%EF%B8%8F%E2%83%A3-create-the-deploy-script)
   - [3. Setup GitHub Actions](#3%EF%B8%8F%E2%83%A3-create-the-github-actions-workflow)

# 1. PREREQUISITES

To host a laravel project on an Ubuntu server you must first ensure you have the following things:

✅ Ubuntu server with a sudo-enabled non-root user and a firewall (UFW).

✅ Domain name or sub-domain pointed to the server IP.

✅ Nginx installed and NginxFull enabled on ufw. firewall.

✅ MySQL server installed, and a non-root user.

✅ PHP and PHP-FPM installed and running ( `sudo service php8.3-fpm status `)

✅ All php extensions required for laravel, ex:

    sudo apt install openssl php8.3-bcmath php8.3-curl php8.3-json php8.3-mbstring php8.3-mysql php8.3-tokenizer php8.3-xml php8.3-zip

✅ Certbot installed for ssl generation.

✅ Git user for auto-deployment

# 2. STEPS

### **1️⃣** Prepare a domain or sub-domain for the project.

### **2️⃣** Create 'prod' branch on the project remote repository.

### **3️⃣** On the server, cd to **/var/www**, clone the project and checkout to prod branch.

&emsp;&emsp; 💠To clone the project via SSH, generate an SSH key pair, add it to GitHub, then restart the SSH agent.
<br>
&emsp;&emsp; 💠Run the command ``git config pull.rebase false `` to allow merging the pull requests.

### **4️⃣** Create a new database for that project
```
mysql -u myapp_user -p
 > show databases;
 > create database myapp;
 > exit;
```
&emsp;&emsp; 💠  If you encounter issues with the new database permissions for **myapp_user**, you may log in as root and grant the necessary permissions:
```
sudo mysql
 > grant all privileges on myapp.* to 'myapp_user'@'%';
 > flush privileges;
 > exit;
```
### **5️⃣** Assign ownership of the project folder to the current user:
```
sudo chown -R $USER:$USER /var/www/myapp
```

### **6️⃣** Create the .env file and fill in the configurations:
```
cp .env.example .env
```

### **7️⃣** Now navigate to the project folder and run the Laravel setup commands:
```
composer install
php artisan key:generate
php artisan storage:link
php artisan migrate:fresh --seed --force
```

### **8️⃣** Install front-end dependencies and build assets
```
npm install
npm run build
```

### **9️⃣** Create an Nginx config file for the project
```
nano /etc/nginx/sites-available/myapp
```
&emsp;&emsp; 💠  There is a starter Nginx configuration for Laravel projects in the official documentation, or you can copy the config of an existing project, but be careful not to include the SSH config part.  <br>
&emsp;&emsp; 💠 This is a generale example:
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
&emsp;&emsp; 💠 Create a symbolic link to sites-enabled:
```
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled
```
&emsp;&emsp; 💠Check nginx syntax and reload it:
```
sudo nginx -t
sudo systemctl restart nginx
```
### **1️⃣0️⃣**  Generate the SSL certificate using Certbot:
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

### **1️⃣1️⃣** Give the web server user write access to storage and cache:
```
sudo chown -R www-data:www-data /var/www/myapp/storage
sudo chown -R www-data:www-data /var/www/myapp/bootstrap/cache
```
&emsp;&emsp; 💠 Optionally restart the php-fpm:
```
sudo systemctl restart php-fpm
```
# 3. AUTO-DEPLOY WORKFLOW
We will use GitHub Actions to create a CI/CD pipeline for our auto-deploy workflow.

### **1️⃣** Add SSH key to GitHub:
&emsp;&emsp; 💠We must first add the **private** SSH key that we use to connect to our server to GitHub so it can connect to our server and deploy the project. <br><br>
&emsp;&emsp; 💠 first copy the key from our local machine:
```
cat ~/.ssh/pair_name
``` 
&emsp;&emsp; 💠 Then in repository of the project on github, go to ``Settings > Secretes and Variables > new repository secret`` and add it there with a name like  **DO_SSH_KEY** 

### **2️⃣** Create the deploy script
```
nano /var/www/myapp/deploy.sh
```
- example script:
```
#!/bin/bash

# Navigate to the project directory
cd /var/www/myapp

# Pull the latest changes from the repository
git pull origin prod

# Install/update Composer dependencies
composer install --no-interaction --prefer-dist --optimize-autoloader

# Install/update Node.js dependencies
npm install

# Build frontend assets
npm run build

# Run database migrations
php artisan migrate --force

# Clear and cache config/routes/views
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Set correct permissions
sudo chown -R www-data:www-data /var/www/myapp/storage /var/www/myapp/bootstrap/cache
sudo chmod -R 775 /var/www/myapp/storage /var/www/myapp/bootstrap/cache

# Restart PHP-FPM (or whichever service is necessary to reload the application)
sudo systemctl restart php8.3-fpm

echo "Deployment completed successfully."
```
&emsp;&emsp; 💠Then make that file executable:
```
chmod +x /var/www/myapp/deploy.sh
```

### **3️⃣** Create the GitHub Actions WorkFlow
&emsp;&emsp; 💠create a new file on your local project at ``.github/workflow/deploy.yml``
- example script:
```
name: Deploy MyApp Project

on:
  push:
    branches:
      - prod  # Trigger only when pushing to the prod branch

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Copy SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DO_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

      - name: Add SSH key to the agent
        run: |
          eval "$(ssh-agent -s)"
          ssh-add ~/.ssh/id_rsa

      - name: Deploy to DigitalOcean
        run: |
          ssh -o StrictHostKeyChecking=no server_user@server_ip 'bash /var/www/myapp/deploy.sh'
        env:
          SSH_AUTH_SOCK: /tmp/ssh_agent.sock

```
# 🚀That's it 🚀
