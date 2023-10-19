# How to deploy a server in nginx

[USEFUL LINK](https://medium.com/@eziz-hudayberdiyev/configuring-nginx-for-frontend-and-backend-applications-on-the-single-server-41c24188c8ce)

### Backend
- A backend runs on a port, *for example 8080*, for security, you need to make the backend run on *localhost:8080* so that the port is inaccessable from the outside
- Nginx is listening on port 80, so we need to proxy all requests to port 8080 for this backend
- Create a file called `api.domain.com` in `/etc/nginx/sites-available/` directory, replace `domain.com` with your domain name:
```bash
    ~> sudo nano /etc/nginx/sites-available/api.domain.com
```
- Write the following content:
```bash
    server {
        server_name api.domain.com;
        include proxy_params;
        proxy_pass http://127.0.0.1:8080;
    }
```
- Save and exit the file
- Make a hard link (or soft link but in linux, hard links work well) of `/etc/nginx/sites-available/api.domain.com` to `/etc/nginx/sites-enabled/` directory with the following command:
```bash
    ~> sudo ln /etc/nginx/sites-available/api.domain.com /etc/nginx/sites-enabled/api.domain.com
```
- Test nginx:
```bash
    ~> sudo nginx -t
```
- If the test is successful, restart nginx:
```bash
    ~> sudo systemctl restart nginx
```
- Open your browser and go to `http://domain.com/api`, but __http__ is not secure, we need to make it **https**
- Now run the following command:
```bash
    ~> sudo certbot --nginx
```
- Select the backend server to be certified
- Follow the instructions
- Now the backend server is running on `https://api.domain.com`

### Frontend (React)
- A frontend includes production build of your application, we don't need a proxy because the frontend is served from HTML and JavaScript files, this is done directly from the root directory.
- Put the production build of your application in `/var/www/domain.com`.
- Create a file called `domain.com` in `/etc/nginx/sites-available/` directory, replace `domain.com` with your domain name:
```bash
    ~> sudo nano /etc/nginx/sites-available/domain.com
```
- Write the following content:
```bash
    server {
        server_name domain.com;
        root /var/www/domain.com;
        location / {
                try_files $uri /index.html =404;
        }
    }
```
- Save and exit the file
- Make a hard link (or soft link but in linux, hard links work well) of `/etc/nginx/sites-available/domain.com` to `/etc/nginx/sites-enabled/` directory with the following command:
```bash
    ~> sudo ln /etc/nginx/sites-available/domain.com /etc/nginx/sites-enabled/domain.com
```
- Test nginx:
```bash
    ~> sudo nginx -t
```
- If the test is successful, restart nginx:
```bash
    ~> sudo systemctl restart nginx
```
- Open your browser and go to `http://domain.com`, but __http__ is not secure, we need to make it **https**
- Now run the following command:
```bash
    ~> sudo certbot --nginx
```
- Select the frontend server to be certified
- Follow the instructions
- Now the frontend server is running on `https://domain.com`

### PHP server files
- PHP server files include production build of your application, we need to proxy to php-fpm because nginx doesn't interpret PHP files
- Create a file called `php-domain.com` in `/etc/nginx/sites-available/` directory, replace `php-domain.com` with your domain name:
```bash
    ~> sudo nano /etc/nginx/sites-available/php-domain.com
```
- Write the following content:
```bash

  root /var/www/php-domain.com;

  index index.php index.html index.htm index.nginx-debian.html;

  server_name php-domain.com;

  access_log /var/www/php-domain.com/log.log;
  error_log  /var/www/php-domain.com/error.log error;

  # pass PHP scripts on Nginx to FastCGI (PHP-FPM) server
  location ~ \.php$ {
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    # Nginx php-fpm sock config:
    fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    fastcgi_index index.php;
    include fastcgi.conf;
  }

  location / {
    try_files $uri $uri/ =404;
  }

  # deny access to Apache .htaccess on Nginx with PHP, 
  # if Apache and Nginx document roots concur
  location ~ /\.ht {
    deny all;
  }
```
- Save and exit the file
- Make a hard link (or soft link but in linux, hard links work well) of `/etc/nginx/sites-available/php-domain.com` to `/etc/nginx/sites-enabled/` directory with the following command:
```bash
    ~> sudo ln /etc/nginx/sites-available/php-domain.com /etc/nginx/sites-enabled/php-domain.com
```
- Test nginx:
```bash
    ~> sudo nginx -t
```
- If the test is successful, restart nginx:
```bash
    ~> sudo systemctl restart nginx
```
- Open your browser and go to `http://php-domain.com`, but __http__ is not secure, we need to make it **https**
- Now run the following command:
```bash
    ~> sudo certbot --nginx
```
- Select the frontend server to be certified
- Follow the instructions
- Now the frontend server is running on `https://php-domain.com`

## Known Issues
- /var/www files permission issues, we need to make sure they are owned by `www-data` user and group, if not, you can use `sudo chown -R www-data:www-data /var/www` to fix it.
- It is better to not use underscores `_` in headers and use the hyphen `-` instead.
- For php-fpm, you need to use `listen unix:/run/php/php8.1-fpm.sock` instead of `listen 127.0.0.1:9000`
- In php-fpm, confirm the version of php-fpm, for example, you might use php-fpm 7.4 instead of 8.1 or 8.3
- Never restart nginx before you make sure that the output of `sudo nginx -t` is successful, otherwise, nginx might stop working and crash all the applications that are running on nginx
