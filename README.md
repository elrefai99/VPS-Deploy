# Connection with VPS Server (AWS-EC2)

How to deploy nodejs app to AWS EC2 Ubuntu 22 Server with free SSL and Nginx reverse proxy

---
## Installation instructions
## connect with server

- In EC2 instance connect: you can choice your username ubuntu or as administrator ```(root)```
- In SSH instance connect: open terminal and write this command 
```bash
ssh -i <key.pem> ubuntu@<ip-address> -v
```

## First Configuration

### Update apt and this will take some seconds

```bash
sudo apt update && sudo apt upgrade
```
### Check GIT version
```bash
git --version
```

- if git version under ```2.42.0``` then you can update it by running 
```bash 
  apt install git 
```
### Install Node.js and npm
```bash
curl -sL https://deb.nodesource.com/setup_18.x | bash -
```
```bash
apt-get install -y nodejs
```

#### Check nodejs installed
```bash
node --version
```

#### Check npm installed

```bash
npm --version
```

### clone project from github to server
- for upload your project you can user clone the repository from github

```bash
git clone https://github.com/yourUsername/yourProject.git
```
### Test run project

- install package dependencies for your project
```bash
npm install
```
- test run
```bash
node index.js
```

- then you can take ip address and run it in your browser

### Make project still available running (Make sure everything working)

#### Install pm2

```bash
npm install -g pm2
```

#### Starting the app with pm2
```bash
pm2 start index.js
```

#### Saves the running processes, if not saved, pm2 will forget, the running apps on next boot
```bash
pm2 save
```

#### IMPORTANT: If you want pm2 to start on system boot
```bash
pm2 startup 
```

### Install Nginx web server
```bash
sudo apt install nginx
```
Delete the default config
```bash
rm /etc/nginx/sites-available/default
```
```bash
rm /etc/nginx/sites-enabled/default
```
#### Create new project config

```bash
sudo nano /etc/nginx/sites-available/project_name
```
Add the following to the location part of the server block
```bash
worker_processes auto;

events {
     worker_connections 1024;
}

http {
     include /etc/nginx/mime.types;
     default_type application/octet-stream;

     # Logging
     access_log /var/log/nginx/access.log;
     error_log /var/log/nginx/error.log;

     # Performance
     sendfile on;
     tcp_nopush on;
     keepalive_timeout 65;

     # Gzip compression
     gzip on;
     gzip_vary on;
     gzip_proxied any;
     gzip_comp_level 6;
     gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;

     # Rate limiting zone for auth endpoints
     limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;

     # Upload size limit (for prescription images)
     client_max_body_size 100M;

     # Upstream definitions
     upstream frontend {
          server frontend:3000;
     }

     upstream backend {
          server backend:5000;
     }

     # Redirect HTTP -> HTTPS
     server {
          listen 80;
          server_name _;
          return 301 https://$host$request_uri;
     }

     # HTTPS server
     server {
          listen 443 ssl;
          server_name _;

          # ssl_certificate /etc/nginx/ssl/self.crt;
          # ssl_certificate_key /etc/nginx/ssl/self.key;
          # ssl_protocols TLSv1.2 TLSv1.3;
          # ssl_ciphers HIGH:!aNULL:!MD5;
          # ssl_session_cache shared:SSL:10m;
          # ssl_session_timeout 10m;

          # Security headers
          add_header X-Frame-Options "SAMEORIGIN" always;
          add_header X-Content-Type-Options "nosniff" always;
          add_header X-XSS-Protection "1; mode=block" always;
          add_header Referrer-Policy "strict-origin-when-cross-origin" always;
          add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

          # API proxy
          location /api/ {
               proxy_pass http://backend;
               proxy_http_version 1.1;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_set_header Referer $http_referer;
               proxy_set_header Connection "";
               proxy_set_header Content-Type $content_type;
               proxy_set_header Accept-Encoding "";
               proxy_set_header Accept-Language $http_accept_language;
               proxy_set_header Accept-Charset $http_accept_charset;
               proxy_set_header Accept $http_accept;
               proxy_set_header User-Agent $http_user_agent;
               proxy_set_header Cache-Control "no-store, no-cache, must-revalidate, post-check=0, pre-check=0";
               proxy_set_header Pragma "no-cache";
               proxy_pass_request_body on;
               proxy_set_header Transfer-Encoding "";
               proxy_buffering off;
               proxy_request_buffering off;
          }

          # WebSocket proxy for Socket.io
          location /socket.io/ {
               proxy_pass http://backend;
               proxy_http_version 1.1;
               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection "upgrade";
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_cache_bypass $http_upgrade;
          }

          # Static file caching
          location ~* ^/(images|fonts)/ {
               proxy_pass http://frontend;
               expires 30d;
               add_header Cache-Control "public, immutable";
          }

          location /_next/static/ {
               proxy_pass http://frontend;
               expires 365d;
               add_header Cache-Control "public, immutable";
          }

          # Frontend proxy (default)
          location / {
               proxy_pass http://frontend;
               proxy_http_version 1.1;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection "upgrade";
          }
     }
}
```
create site-available and site-enabled to let any change make in both
```bash
ln -s /etc/nginx/sites-available/project_name /etc/nginx/sites-enabled/project_name
```

Check NGINX config
```bash
sudo nginx -t
```
```bash
systemctl start nginx
```
Restart NGINX
```bash
sudo service nginx restart
```

You should now be able to visit your IP with no port (port 80) and see your app. Now let's add a domain
## Enjoy Your Nodejs server 😁
