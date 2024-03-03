## Internal Server

squaremap ships with a very basic internal web server to help host your map files. It has all the bells and whistles you'll need to get your map online and sharable.

```yaml
  # the internal web server settings
  internal-webserver:

    # set to true to use the internal web server.
    # set to false if you want to run your own
    # external web server
    enabled: true

    # the ip the internal web server will bind to
    # (leave this alone if you don't know what it does)
    bind: 0.0.0.0

    # the port the internal web server will bind to
    port: 8080
```

By default, the internal web server is enabled for your convenience. If you need a more advanced setup (such as SSL certs) you will want to use an external server instead.

The default address to bind to is `0.0.0.0` which simply means "any address the machine gets". It is only used if you have multiple network cards installed on the machine and need to bind to a specific one. If you don't understand what that means, then this option isn't for you and you should leave it alone.

The default port is 8080. You will want to ensure this port is open to outside internet connections so others can access the map. If you have access to port 80 and no other services are using it, you can set it to port 80 which will allow users to use the map without specifying a port in the address bar.

## External Server

Some users may want to disable the internal web server to use their own external web server.

If you cannot save tiles on the same machine the web server will run on and do not need to disable the internal web server, you can set up a [reverse proxy](#reverse-proxy). Otherwise, you can follow this section to route all traffic through an external web server.

Before you begin, ensure you have:
- root/sudo access on the machine
- a Linux distro with `apt` (Debian, Ubuntu, etc.)
- TCP ports `80` and `443` available and open
- already ran squaremap once with the internal web server enabled
- the Minecraft server is currently offline

#### 1 - Set up a server directory for squaremap

This will be the directory that the web server will serve the map from

```sh
$ sudo mkdir -p /var/www/YOUR_DOMAIN/html
```

Replace `YOUR_DOMAIN` with the domain you wish to use. This can be the name of your subdomain if you have one

***

#### 2 - Assign the correct permissions to the server directory

This will allow your user to read/write to the directory

```sh
# assign ownership to the user
$ sudo chown -R $USER:$USER /var/www/YOUR_DOMAIN/html

# gives the owner read/write/execute perms and only read/execute perms to other users/groups
$ sudo chmod -R 755 /var/www/YOUR_DOMAIN
```

***

#### 3 - Move the files from the default squaremap web directory (`~/plugins/squaremap/web`) to the web server's web root directory (`/var/www/YOUR_DOMAIN/html`)

This can be done in one of two ways:

##### Creating a symlink from the squaremap web directory:

```sh
$ sudo ln -s /path/to/your/server/plugins/squaremap/web /var/www/YOUR_DOMAIN/html
```

As usual, replace `YOUR_DOMAIN` with your domain, and `/path/to/your/server/` to the path where you run the server.

Updating the path in squaremap's config is not required if using a symlink. Instead, your server block configuration will require additional options.

##### Copying the web directory to the web server's root directory:

```sh
$ sudo cp /path/to/your/server/plugins/squaremap/web /var/www/YOUR_DOMAIN/html
```

As usual, replace `YOUR_DOMAIN` with your domain, and `/path/to/your/server/` to the path where you run the server.

Now that you moved the web directory to `/var/www/YOUR_DOMAIN/html`, you need to update the path in squaremap's config.

The option can be found in [`config.yml`](Default-config.yml):
```yml
  # web directory settings (where all the public files go)
  web-directory:

    # the path where the public web directory is
    # relative paths are from the plugin directory
    # absolute paths will work here, too
    path: /var/www/YOUR_DOMAIN/html
```

***

#### 4 - Configure your web server

There are two different sections for configuring Apache2 and NGINX.

- [Apache2](#apache2)
- [NGINX](#nginx)

After either of the above sections are finished, follow [Enabling SSL](#enabling-ssl).

### Apache2

This part of the guide covers how to serve the map with Apache2 without a reverse proxy.

This section assumes that you have:
- **the latest stable release** of Apache2 installed (`2.4.49` as of writing this)
- not modified The Apache config since installation (Should not be an issue if you know what you're doing)
- You've followed the first step of this guide: https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-20-04

#### 1 - Setting up the Apache2 site

```sh
$ sudo nano /etc/apache2/sites-available/YOUR_DOMAIN.conf
```

This will create a new file under `/etc/apache2/sites-available`. Paste the snippet below:

```yaml
<VirtualHost *:80>
  DocumentRoot /var/www/YOUR_DOMAIN/html # sets the root dir

  ServerName YOUR_DOMAIN # the name of the server
</VirtualHost>
```

This will create a new site config for squaremap. As usual, replace `YOUR_DOMAIN` with your domain.  
After you're done editing the file, do `Ctrl + S` to save the file and `Ctrl + X` to exit.

***

#### 2 - Activating the site

```sh
$ sudo a2ensite YOUR_DOMAIN.conf
```

This command will symlink our site file with one in `/etc/apache2/sites-enabled/`. This is the directory where Apache2 will read and enable live sites.

***

#### 3 - Test and restart Apache2

```sh
$ sudo apache2ctl configtest
```

Testing is very important. This command will check for syntax errors and any misconfigured options.

```sh
$ sudo systemctl restart apache2
```

If there were no errors, restart Apache2 with this command. Start up your Minecraft server and make sure nothing appears in `/path/to/your/server/plugins/squaremap/web`.

Navigating to `http://YOUR_DOMAIN/` should show you the map.

### NGINX

This part of the guide will show you how to serve the map with NGINX without a reverse proxy.

This section assumes that you have:
- **the latest stable release** of NGINX installed (`1.20.1` as of writing this)
- not modified The NGINX config since installation (Should not be an issue if you know what you're doing)
- You've followed this guide up to Step 4: https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04

#### 1 - Setting up the NGINX site

```sh
$ sudo nano /etc/nginx/sites-available/YOUR_DOMAIN
```

This will create a new file under `/etc/nginx/sites-available`. Paste the snippet below:

```yaml
server {
  listen 80; # listens on port 80
  listen [::]:80; # listen to all addresses on port 80

  root /var/www/YOUR_DOMAIN/html; # sets the root dir
  index index.html; # the index file name

  server_name YOUR_DOMAIN; # the name of the server

  location / { # location of your files
    try_files $uri $uri/ =404; # if location == null return 404
  }
}
```

This will create a new site config for squaremap. As usual, replace `YOUR_DOMAIN` with your domain.  
After you're done editing the file, do `Ctrl + S` to save the file and `Ctrl + X` to exit.

***

#### 2 - Activating the site

```sh
$ sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled/
```

This command will symlink our new site file with one in `/etc/ngnix/sites-enabled/`. This is the directory where NGINX will read for live sites, and enable them.

To avoid a possible hash bucket memory problem that can arise from adding additional server names, it is necessary to adjust a single value in the `/etc/nginx/nginx.conf` file:

```yml
...
http {
  server_names_hash_bucket_size 64; # uncomment this option
  ...
}
...
```

This option controls server names. You can learn more [here](https://gist.github.com/muhammadghazali/6c2b8c80d5528e3118613746e0041263).

***

#### 3 - Test and restart NGINX

```sh
$ sudo nginx -t
```

Testing is very important. This command will check for syntax errors and any misconfigured options.

```sh
$ sudo systemctl restart nginx
```

If there were no errors, restart NGINX with this command. Start up your Minecraft server and make sure nothing appears in `/path/to/your/server/plugins/squaremap/web`.

Navigating to `http://YOUR_DOMAIN/` should show you the map.

## Reverse Proxy

"A reverse proxy is a server that sits in front of one or more web servers, intercepting requests from clients."  
\- [Cloudflare](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)

This route will allow the use of a domain and existing web server without modifying squaremap's internal web server.

Before you begin, ensure you have:
- root/sudo access on the machine
- a Linux distro with `apt` (Debian, Ubuntu, etc.)
- TCP ports `80` and `443` available and open
- squaremap's internal server is enabled and functional

There are two different sections for configuring Apache2 and NGINX.

- [Apache2](#apache2-1)
- [NGINX](#nginx-1)

After either of the above sections are finished, follow [Enabling SSL](#enabling-ssl).

### Apache2

This part of the guide covers how to serve the map with Apache2 with a reverse proxy.

This section assumes that you have:
- **the latest stable release** of Apache2 installed (`2.4.49` as of writing this)
- not modified The Apache config since installation (Should not be an issue if you know what you're doing)
- You've followed the first step of this guide: https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-20-04

#### 1 - Setting up the Apache2 site

```sh
$ sudo nano /etc/apache2/sites-available/YOUR_DOMAIN.conf
```

This will create a new file under `/etc/apache2/sites-available`. Paste the snippet below:

```yaml
<VirtualHost *:80>
  # reverse proxy to squaremap instance
  ProxyPass "/" "http://MAP_IP:MAP_PORT/"
  ProxyPassReverse "/" "http://MAP_IP:MAP_PORT/"

  ServerName YOUR_DOMAIN # the name of the server
</VirtualHost>
```

This will create a new site config for squaremap. Replace `YOUR_DOMAIN` with your domain, `MAP_IP` with the IP hosting your squaremap instance, and `MAP_PORT` with the port squaremap is hosted on.  
After you're done editing the file, do `Ctrl + S` to save the file and `Ctrl + X` to exit.

***

#### 2 - Activating the site

```sh
$ sudo a2ensite YOUR_DOMAIN.conf
```

This command will symlink our site file with one in `/etc/apache2/sites-enabled/`. This is the directory where Apache2 will read and enable live sites.

***

#### 3 - Test and restart Apache2

```sh
$ sudo apache2ctl configtest
```

Testing is very important. This command will check for syntax errors and any misconfigured options.

```sh
$ sudo systemctl restart apache2
```

If there were no errors, restart Apache2 with this command. Start up your Minecraft server and make sure squaremap's internal server is running.

Navigating to `http://YOUR_DOMAIN/` should show you the map.

### NGINX

This part of the guide will show you how to serve the map with NGINX with a reverse proxy.

This section assumes that you have:
- You have **the latest stable release** of NGINX installed (`1.20.1` as of writing this)
- The NGINX config has not been modified since installation (Should not be an issue if you know what you're doing)
- You've followed this guide up to Step 4: https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04

#### 1 - Setting up the NGINX site

```sh
$ sudo nano /etc/nginx/sites-available/YOUR_DOMAIN
```

This will create a new file under `/etc/nginx/sites-available`. Paste the snippet below:

```yaml
server {
  listen 80; # listens on port 80
  listen [::]:80; # listen to all addresses on port 80

  server_name YOUR_DOMAIN; # the name of the server

  location / {
    proxy_pass http://MAP_IP:MAP_PORT; # reverse proxy to squaremap instance
  }
}
```

This will create a new site config for squaremap. Replace `YOUR_DOMAIN` with your domain, `MAP_IP` with the IP hosting your squaremap instance, and `MAP_PORT` with the port squaremap is hosted on.  
After you're done editing the file, do `Ctrl + S` to save the file and `Ctrl + X` to exit.

***

#### 2 - Activating the site

```sh
$ sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled/
```

This command will symlink our new site file with one in `/etc/ngnix/sites-enabled/`. This is the directory where NGINX will read for live sites, and enable them.

To avoid a possible hash bucket memory problem that can arise from adding additional server names, it is necessary to adjust a single value in the `/etc/nginx/nginx.conf` file:

```yml
...
http {
    ...
    server_names_hash_bucket_size 64; # uncomment this option
    ...
}
...
```

This option controls server names. You can learn more [here](https://gist.github.com/muhammadghazali/6c2b8c80d5528e3118613746e0041263).

***

#### 3 - Test and restart NGINX

```sh
$ sudo nginx -t
```

Testing is very important. This command will check for syntax errors and any misconfigured options.

```sh
$ sudo systemctl restart nginx
```

If there were no errors, restart NGINX with this command. Start up your Minecraft server and make sure squaremap's internal server is running.

Navigating to `http://YOUR_DOMAIN/` should show you the map.

## Enabling SSL

SSL is important to keep the users accessing your site secure.  
You can set up SSL pretty quickly by running the commands below:

This section assumes that you have:
- Successfully set up an [external web server](#external-server) or [reverse proxy](#reverse-proxy).

```sh
$ sudo apt-get update
$ sudo apt-get install certbot
$ sudo certbot --YOUR_WEBSERVER
```

Replace `YOUR_WEBSERVER` with either `apache` or `nginx` depending on which web server you set up.

Make sure to choose your domain/site and your preferred options during the installation. Wait for it to finish and SSL should be set up.
