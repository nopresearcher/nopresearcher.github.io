---
title: Deploying CTFd
---

I had a recent requirement to deploy CTFd.  It was a simple deployment, but then I noticed some performance issues.  This is where I spent too much time trying to figure out how to increase the performance of the site.

This is the deployment guide for CTFd 2.0.3.  If you have any suggestions, let me know so I can incorporate it.


# Linode 

I used a $10 a month node to deploy this. After performance tweaking this was overkill for 70 people.  I could have handled that load with a $5 node.

Nanode 2GB

50GB storage

1CPU

2TB xfer


## Linode configuration:

Ubuntu 18.04 LTS

512 MB swap


## Basic Configuration

```bash
# Configure User
adduser ctfd

usermod -aG sudo ctfd

# UFW Firewall
ufw allow openssh
ufw allow http
ufw allow https
ufw enable

# update and install required software
apt update
apt upgrade -y
apt install -y python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools nginx git
pip3 install pipenv

# install CTFd
cd /var/www
git clone https://github.com/CTFd/CTFd.git

su ctfd
sudo chown -R ctfd:www-data /var/www/CTFd
cd /var/www/CTFd

# Create a pipenv to run CTFd in
pipenv install --python 3.6
pipenv shell
./prepare.sh
```

At this point if you need to test the deployment you can with the following.  If not, continue with the configuration.

## Testing
```bash
sudo ufw allow 5000
gunicorn --bind 0.0.0.0:5000 'CTFd:create_app()'
http://www.yourdomain.com:5000
```

## CTFd Configuration

I had to tweak the systemd unit file to adjust workers and worker types.  For the single core deployment, I used 3 workers with a worker-class of gevent.  I also tweaked the keep-alive and changed it to 2.

```bash
# identify the pipenv virtual environment for use in unit file
pipenv --venv
/home/ctfd/.local/share/virtualenvs/CTFd-rOJbThUf

# Create unit file
sudo vim /etc/systemd/system/ctfd.service

[Unit]
Description=Gunicorn instance to serve ctfd
After=network.target

[Service]
User=ctfd
Group=www-data
WorkingDirectory=/var/www/CTFd
Environment="PATH=/home/ctfd/.local/share/virtualenvs/CTFd-rOJbThUf/bin"
ExecStart=/home/ctfd/.local/share/virtualenvs/CTFd-rOJbThUf/bin/gunicorn --bind unix:app.sock --keep-alive 2 --workers 3 --worker-class gevent 'CTFd:create_app()' --access-logfile '/var/log/CTFd/CTFd/logs/access.log' --error-logfile '/var/log/CTFd/CTFd/logs/error.log'


[Install]
WantedBy=multi-user.target
```

```bash
# Create log directories
sudo mkdir -p /var/log/CTFd/CTFd/logs/
sudo chown -R ctfd:www-data /var/log/CTFd/CTFd/logs/

# Start CTFd service
sudo systemctl enable ctfd
sudo systemctl start ctfd
sudo systemctl status ctfd

# Create nginx site, let's encrypt will handle the https later
sudo vim /etc/nginx/sites-available/ctfd

# Nginx config
# the client_max_body_size enables file uploads over the default of 1MB
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com your.ip.add.ress;
    client_max_body_size 75M;
    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/CTFd/app.sock;

    }
}

# Link config file
sudo ln -s /etc/nginx/sites-available/ctfd /etc/nginx/sites-enabled

# Remove defaults
rm /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default

# Test nginx configuration
sudo nginx -t

# Restart nginx if test wasw good
sudo systemctl restart nginx

# For troubleshooting
tail /var/log/CTFd/CTFd/logs/access.log
tail /var/log/CTFd/CTFd/logs/error.log


# SSL Certs
sudo add-apt-repository ppa:certbot/certbot
sudo apt install python-certbot-nginx

sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
youremail@domain.com

# certificate locations
/etc/letsencrypt/live/yourdomain.com/fullchain.pem
/etc/letsencrypt/live/yourdomain.com/privkey.pem

# renew certificates
certbot renew
```

## DNS

Configure DNS with your hosting/registrar


A 1h your.ip.add.ress

www A 1h your.ip.add.ress


## CTFd Setup

Browse to the site and setup your admin account.
