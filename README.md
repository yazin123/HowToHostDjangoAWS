# Django Project Deployment on AWS EC2

This README provides step-by-step instructions for deploying a Django project from GitHub to an AWS EC2 Ubuntu instance, including necessary permissions, custom domain setup, and ensuring continuous operation.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [EC2 Instance Setup](#ec2-instance-setup)
3. [Project Setup](#project-setup)
4. [Gunicorn Setup](#gunicorn-setup)
5. [Nginx Setup](#nginx-setup)
6. [SSL Configuration](#ssl-configuration)
7. [Custom Domain Setup](#custom-domain-setup)
8. [File Permissions](#file-permissions)
9. [Continuous Operation](#continuous-operation)

## Prerequisites

- An AWS account
- A GitHub repository with your Django project
- A registered domain name (for custom domain setup)

## EC2 Instance Setup

1. Launch an EC2 instance:
   - Choose Ubuntu Server 22.04 LTS
   - Select an appropriate instance type (t2.micro for testing)
   - Configure instance details as needed
   - Add storage (8GB is usually sufficient for small projects)
   - Configure security group:
     - Allow SSH (port 22)
     - Allow HTTP (port 80)
     - Allow HTTPS (port 443)
   - Review and launch the instance
   - Create or select an existing key pair

2. Connect to your instance:
   ```
   ssh -i /path/to/your-key.pem ubuntu@your-instance-public-dns
   ```

3. Update the system:
   ```
   sudo apt update && sudo apt upgrade -y
   ```

## Project Setup

1. Install required packages:
   ```
   sudo apt install python3-pip python3-venv nginx git -y
   ```

2. Clone your project:
   ```
   git clone https://github.com/yourusername/your-repo.git
   cd your-repo
   ```

3. Create and activate a virtual environment:
   ```
   python3 -m venv venv
   source venv/bin/activate
   ```

4. Install project dependencies:
   ```
   pip install -r requirements.txt
   ```

5. Set up environment variables:
   ```
   sudo nano /etc/environment
   ```
   Add your environment variables:
   ```
   export SECRET_KEY="your_secret_key"
   export DEBUG="False"
   export ALLOWED_HOSTS="your_domain.com,www.your_domain.com"
   ```
   Save and exit. Then, reload the environment:
   ```
   source /etc/environment
   ```

6. Run migrations:
   ```
   python manage.py migrate
   ```

7. Collect static files:
   ```
   python manage.py collectstatic
   ```

## Gunicorn Setup

1. Install Gunicorn:
   ```
   pip install gunicorn
   ```

2. Create a Gunicorn service file:
   ```
   sudo nano /etc/systemd/system/gunicorn.service
   ```
   Add the following content:
   ```
   [Unit]
   Description=gunicorn daemon
   After=network.target

   [Service]
   User=ubuntu
   Group=www-data
   WorkingDirectory=/home/ubuntu/your-repo
   ExecStart=/home/ubuntu/your-repo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/your-repo/your_project.sock your_project.wsgi:application

   [Install]
   WantedBy=multi-user.target
   ```

3. Start and enable the Gunicorn service:
   ```
   sudo systemctl start gunicorn
   sudo systemctl enable gunicorn
   ```

## Nginx Setup

1. Create an Nginx configuration file:
   ```
   sudo nano /etc/nginx/sites-available/your_project
   ```
   Add the following content:
   ```
   server {
       listen 80;
       server_name your_domain.com www.your_domain.com;

       location = /favicon.ico { access_log off; log_not_found off; }
       location /static/ {
           root /home/ubuntu/your-repo;
       }

       location / {
           include proxy_params;
           proxy_pass http://unix:/home/ubuntu/your-repo/your_project.sock;
       }
   }
   ```

2. Create a symbolic link:
   ```
   sudo ln -s /etc/nginx/sites-available/your_project /etc/nginx/sites-enabled
   ```

3. Test Nginx configuration:
   ```
   sudo nginx -t
   ```

4. Restart Nginx:
   ```
   sudo systemctl restart nginx
   ```

## SSL Configuration

1. Install Certbot:
   ```
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. Obtain and install SSL certificate:
   ```
   sudo certbot --nginx -d your_domain.com -d www.your_domain.com
   ```

## Custom Domain Setup

1. In your domain registrar's DNS settings, create an A record:
   - Host: @ or your domain name
   - Points to: Your EC2 instance's public IP address

2. If you want to use www subdomain, create a CNAME record:
   - Host: www
   - Points to: your_domain.com

3. Wait for DNS propagation (can take up to 48 hours)

## File Permissions

1. Set correct ownership:
   ```
   sudo chown -R ubuntu:www-data /home/ubuntu/your-repo
   ```

2. Set correct permissions:
   ```
   sudo chmod -R 755 /home/ubuntu/your-repo
   ```

3. Make manage.py executable:
   ```
   chmod +x /home/ubuntu/your-repo/manage.py
   ```

## Continuous Operation

To ensure your project runs even when the terminal is closed:

1. Make sure the Gunicorn service is enabled:
   ```
   sudo systemctl enable gunicorn
   ```

2. You can use `tmux` or `screen` for long-running processes:
   ```
   sudo apt install tmux -y
   tmux new -s django_session
   ```

3. To detach from the tmux session, press `Ctrl+B`, then `D`.

4. To reattach to the session:
   ```
   tmux attach -t django_session
   ```

Your Django project should now be successfully deployed on AWS EC2, accessible via your custom domain with HTTPS, and running continuously.
