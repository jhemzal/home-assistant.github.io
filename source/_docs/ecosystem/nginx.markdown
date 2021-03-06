---
layout: page
title: "NGINX"
description: "Documentation about setting up Home Assistant with NGINX."
release_date: 2016-12-02 15:00:00 -0700
sidebar: true
comments: false
sharing: true
footer: true
redirect_from: /ecosystem/nginx/
---

Using nginx as a proxy for Home Assistant allows you to serve Home Assistant securely over standard ports. This configuration file and instructions will walk you through setting up Home Assistant over a secure connection.

### {% linkable_title 1. Get a domain name forwarded to your IP %}

Chances are, you have a dynamic IP Address (your ISP changes your address periodically). If this is true, you can use a Dynamic DNS service to obtain a domain and set it up to update with you IP. If you purchase your own domain name, you will be able to easily get a trusted SSL certificate later.


### {% linkable_title 2 Install nginx on your server %}

This will vary depending on your OS. Check out Google for this. After installing, ensure that nginx is not running.

### {% linkable_title 3. Obtain an SSL certificate %}

There are two ways of obtaining an SSL certificate.

#### {% linkable_title Using Let's Encrypt %}
If you purchased your own domain, you can use https://letsencrypt.org/ to obtain a free, publicly trusted SSL certificate. This will allow you to work with services like IFTTT. Download and install per the instructions online and get a certificate using the following command.

```
./letsencrypt-auto certonly --standalone -d example.com -d www.example.com
```

Instead of example.com, use your domain. You will need to renew this certificate every 90 days.

#### {% linkable_title Using openssl %}

If you do not own your own domain, you may generate a self-signed certificate. This will not work with IFTTT, but it will encrypt all of your Home Assistant traffic.

```
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 9999
sudo cp key.pem cert.pem /etc/nginx/ssl
sudo chmod 600 /etc/nginx/ssl/key.pem /etc/nginx/ssl/cert.pem
sudo chown root:root /etc/nginx/ssl/key.pem /etc/nginx/ssl/cert.pem
```

### {% linkable_title 4. Create dhparams file %}

As a fair warning, this file will take a while to generate.

```
cd /etc/nginx/ssl
sudo openssl dhparam -out dhparams.pem 2048
```

### {% linkable_title 5. Install configuration file in nginx. %}

Create a new file `/etc/nginx/sites-available/hass` and copy the configuration file at the bottom of the page into it.

### {% linkable_title 6. Enable the Home Assistant configuration. %}

```
cd /etc/nginx/sites-enabled
sudo unlink default
sudo ln ../sites-available/hass default
```

### {% linkable_title 7. Start NGINX. %}

Double check this configuration to ensure all settings are correct and start nginx.


### {% linkable_title 8. Port forwarding. %}

Forward ports 443 and 80 to your server on your router. Do not forward port 8123.

### {% linkable_title NGINX Config %}

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        # Update this line to be your domain
        server_name example.com;

        # These shouldn't need to be changed
        listen [::]:80 default_server ipv6only=off;
        return 301 https://$host$request_uri;
    }

    server {
        # Update this line to be your domain
        server_name example.com;

        # Ensure these lines point to your SSL certificate and key
        ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        # Use these lines instead if you created a self-signed certificate
        # ssl_certificate /etc/nginx/ssl/cert.pem;
        # ssl_certificate_key /etc/nginx/ssl/key.pem;

        # Ensure this line points to your dhparams file
        ssl_dhparam /etc/nginx/ssl/dhparams.pem;


        # These shouldn't need to be changed
        listen [::]:443 default_server ipv6only=off; # if your nginx version is >= 1.9.5 you can also add the "http2" flag here
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";
        ssl on;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;

        proxy_buffering off;

        location / {
            proxy_pass http://localhost:8123;
            proxy_set_header Host $host;
            proxy_redirect http:// https://;
            proxy_http_version 1.1;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
```
