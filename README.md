# nginx cheatsheet - from Udemy (nginx-fundamentals)

## Location
Location matching will strictly follow this order.

#### 1. Exact match
```nginx
location = /hello {
    return 200 'Match exact /hello path';
}
```

#### 2. Preferential prefix match
```nginx
location ^~ /hello {
    return 200 'Match anything that has /hello path prefix but it has higher precedence than prefix match';
}
```

#### 3. REGEX match
```nginx
location ~ /regex[0-9] {
    return 200 'Match anything that fulfills current regex';
}

location ~* /regex[0-9] {
    return 200 'Match anything that fulfills current regex. Additionally, this one is case insensitive';
}
```

#### 4. Prefix match
```nginx
location /hello {
    return 200 'Match anything that has /hello path prefix';
}
```

## Variable
nginx default will have a default list of variables. Check out it [here](https://nginx.org/en/docs/varindex.html).

```nginx
location /inspect {
    return 200 "$host\$uri\n$args";
}
```

To set a custom variable, you can call `set` with the variable name and value.

```nginx
http {
    server {
        # ...
        set $weekend 'No';

        if ($date_local ~ 'Saturday|Sunday') {
            set $weekend 'Yes';
        }

        location /is_weekend {
            return 200 $weekend;
        }
    }
}
```

#### Args variable
For GET request, you can access query parameter in nginx by using `$arg_query_name`.

```nginx
# check statis api key for every request
if ($arg_apikey != 1234) {
    return 401 "Incorrect api key";
}

# access name query parameter via $arg_name
location /inspect {
    return 200 "Name: $arg_name";
}
```

## Rewrites & Redirect
Rewrite and redirect have the same purpose, but it will have a very different result.

#### Redirect
For redirect, browser will be the one who requests for new resources from the server again. For example:

```nginx
location /logo {
    return 307 /thumb.png;
}
```

When browser requests for /logo, nginx will return redirect code with new url and browser in the end will display server.com/thumb.png

#### Rewrite
For rewrite, it will take more system resource because the matching url will be executed again with new path. This rewrite path happens internally and will be unknown to the client. The final url will be `/user/\w+` instead of `/greet`.

```nginx
rewrite ^/user/\w+ /greet;

location /greet {
    return 200 'Hello user';
}
```

For `rewrite`, you can also use capture group to refer to the value under current expression by adding `()`. Then you can refer to it by the capture group id.

```nginx
rewrite ^/user/(\w+) /greet/$1;
```

Rewrite url can get complicated quickly. If you want your url won't be rewrite again, you can add option `last` to the rewrite directive.

```nginx
rewrite ^/user/(\w+) /greet/$1 last;
```

## try_files directive
Make nginx check for a resource to response with in any number of location relative to root directory. The last element will be considered as the rewrite.

```nginx
try_files $uri /greet /friendly_404;

location /friendly_404 {
    return 404 'Sorry, that resource could not be found.';
}
```

This config means try the current `$uri`, `/greet` that relative the root path, if nothing matches it will rewrite the uri to `/friendly_404`. Relative to the root directory means if you have `root /sites/demo;`, then `try_files` will check for file under `/sites/demo/$uri` and `/sites/demo/greet`. 

## Name location
Allow you to name a resource and refer to it by name instead of uri path. Use the `@` to start naming a location.

```nginx
try_files $uri /greet @friendly_404;

location @friendly_404 {
    return 404 'Sorry, that resource could not be found.';
}
```

## Access log and error log
To enable logging on a particular location (for example `/secure`), use `access_log` and `error_log` directives

```nginx
location /secure {
    access_log /var/log/nginx/secure.access.log

    return 200 "Welcome to secure area.";
}
```

This will only log requests to `/secure` to `secure.access.log` file, but not `access.log`. Additionally, if you want to enable logging for both, you need to explicitly specify logging to `access.log`.

```nginx
location /secure {
    access_log /var/log/nginx/secure.access.log
    access_log /var/log/nginx/access.log

    return 200 "Welcome to secure area.";
}
```

To disable logging for a given context

```nginx
location /secure {
    access_log off;

    return 200 "Welcome to secure area.";
}
```

Also, check out this [document](https://nginx.org/en/docs/http/ngx_http_log_module.html) for more configuration with the directive.

## Inheritance and directive types

There are 3 types of directives: `Standard directive`, `Array directive`, `Action directive`.

```nginx
events {}

# 1. Array directive
# Can be specified multiple times without overriding a previous setting
# Gets inherited by all child context
# Child context can override inheritance by re-delaring directive
access_log /var/log/nginx/access.log
access_log /var/log/nginx/custome.log.gz custom_format;

http {
    # include statement - non directive
    include mime.types;
    server {
        listen 80;
        sever_name site1.com;

        # inherits access_log from parent context (1)
    }

    server {
        listen 80;
        server_name site2.com;

        # 2. Standard directive
        # Can only be declared once. A second declaration overrides the first
        # Gets inherited by all child contexts
        # Child context can override inheritance by re-declaring directive

        root /sites/site2;

        # Completely overrides inheritance from (1)
        access_log off;

        location /images {
            # Uses root directive inherited from (2)
            try_files $uri /stock.png;
        }

        location /secret {
            # 3. Action directive
            # Invokes an action such as a rewrite or redirect
            # Inheritance does not apply as the request is either stopped (redirect/response) or re-evaluated (rewrite)
            return 403 "You do not have permission to view this.";
        }
    }
}
```

## Use nginx as a reverse proxy for php

```nginx
# remember to always set this
# you would need to run as the same user as php.sock for nginx to be able to read uds from php process.
user www-data;

events {}

http {
    include mime.types;

    server {
        listen      80;
        server_name 10.0.0.56;
        root /sites/demo;

        # tell nginx which file to load if the request points to a directory
        # /request_path/ for example, it will load /request_path/index.html
        index index.php index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            # pass php requests to the php-fpm service (fastcgi)
            include fastcgi.conf;
            # forward all the request to php.sock
            fastcgi_pass unix:/run/php/php7.1-fpm.sock;
        }
    }
}
```

This will forward the request to php server and serve the response back.

## nginx workers and connections
By default nginx only spawns 1 instance of nginx worker. To config it to spawn more workers, use `worker_process` on the main context.

```nginx
worker_processes 2;
```

Spawning more worker_processes by no mean provide better performance. Those nginx workers already handle request asynchronously. It means they will handle requests as fast as the hardware are capable of. Therefore, increasing worker simply doesn't improve hardware capability.

![nginx-worker](/images/nginx-worker.png)

1 nginx process will be run on a single core, so normally you would want to maximise your hardware capability by running the same number of workers as the number of cpu cores. To know how many cores your server have, run `nproc` or `lscpu`.

Luckily, nginx provide `worker_process auto;` to automate this process.

Additionally, when dealing with worker_processes, you should also care for the number of opened file descriptor (fd). To check the maximum number of fd you can open, run `ulimit`. This is equivalent on the maximum number of connections your server can handle. 

```nginx
events {
    # allow number of opened fd
    worker_connections 1024;
}
```

In theory, it would look like this.
```
worker_processes x worker_connections = max_connections
```

## nginx buffers and timeouts optimisation
Buffering means that nginx worker process reads data into memory before writing it to the next destination, or when serving static file it will load the content into the memory before responding to the client.

![nginx-buffer-request](/images/nginx-buffer-request.png)

![nginx-buffer-static-file](/images/nginx-buffer-static-file.png)

Timeouts simply mean the cut-off time for certain events.

![nginx-timeouts](/images/nginx-timeouts.png)

```nginx
user www-data;

http {
    include mime.types;

    # Buffer size for POST submissions
    client_body_buffer_size 10K; # if your form-data is large, it is better to increase this buffer_size
    client_max_body_size 8M; # Don't accept body that large than 8M

    # Buffer size for Headers
    client_header_buffer_size 1K;

    # Max time to receive client headers/body
    # This is not the time taken to transmit the entire request body, but the time between consecutive read operation that happens to the buffer
    client_body_timeout 12; # in milliseconds
    client_header_timeout 12;

    # Max time to keep a connection open for
    keepalive_timeout 15;

    # Max time for the client accept/receive a response
    send_timeout 10;

    # Skip buffering for static files and write directly bytes to the response
    sendfile on;

    # Optimise sendfile packets
    tcp_nopush on;
}
```

![nginx-buffer-directive](/images/nginx-buffer-directive.png)
![nginx-timeout-directive](/images/nginx-timeout-directive.png)

## Add nginx dynamic modules
In order to add dynamic modules to nginx, you have to rebuild nginx from source

Making sure that you don't change any default nginx configuration by running this command first.

```bash
$ nginx -V
# for example
nginx version: nginx/1.13.0
built by gcc 7.2.0 (Ubuntu 7.2.0-8ubuntu3.2)
built with OpenSSL 1.0.2g
TLS SNI support enalbed
configure arguments: --sbin-path=/usr/bin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --with-pcre --with-http_ssl_module
```

To see a list of available configuration, in nginx src repo. Run

```bash
$ ./configure --help
```

To build a dynamic module (for example, with-http_image_filter_module=dynamic) with nginx, run:

```bash
$ ./configure --sbin-path=/usr/bin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --with-pcre --with-http_ssl_module --with-http_image_filter_module=dynamic --modules-path=/etc/nginx/modules
$ make
$ make install
```

`modules-path` is to specify the install location the module and make it easier to refer to it later in the code.

```nginx
# load the dynamic module
load_module modules/ngx_http_image_filter_module.so;

http {
    # ...
    server {
        # ...
        location = /thumb.png {
            image_filter rotate 180;
        }
    }
}
```

## Headers and Expires
For specifying header, you can use `add_header` directive. Additionally, use`expires` directive if you want to control the expiry of the resource itself.

```nginx
location ~* \.(css|js|jpg|png)$ {
    access_log off;

    add_header Cache-Control public;
    add_header Pragma public;
    add_header Vary Accept-Encoding; # Specify that the response can be very based on Accept-Encoding from the client. For example (gzip)
    expires 1M;
}
```

## Compress with gzip
Compression can be used to reduce the file sizes that are sent to the client, which overall can lead to better performance in the client side. However, the trade-off of that is it would consume more resources for compressing those files.

```nginx
http {
    gzip on;
    gzip_comp_level 3;
    gzip_types text/css;
    gzip_types text/javascript;
}
```

Compress level can be from 0 to 9. However, the best utilisation of server resources and file sizes are around 3 to 4.

![nginx-compression-level](/images/nginx-compression-level.png)

## FastCGI Cache
FastCGI Cache can help reduce greatly server load by serving dynamic content from cache.

```nginx
http {
    fastcgi_cache_path /tmp/nginx_cache levels=1:2 key_zone=ZONE_1:100m inactive=60m;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";

    # allow you have more visibility into cache status
    add_header X-Cache $upstream_cache_status;

    server {
        location ~ \.php$ {
            # pass php requests to the php-fpm service (fastcgi)
            include fastcgi.conf;
            # forward all the request to php.sock
            fastcgi_pass unix:/run/php/php7.1-fpm.sock;

            # Enable cache by the name and status type
            fastcgi_cache ZONE_1;
            fastcgi_cache_valid 200 60m;
        }
    }
}
```

For certain request, if you want to skip storing request content into nginx cache. You can config fastcgi to ignore certain request using `fastcgi_cache_bypass` and `fastcgi_no_cache`.

```nginx
http {
    set $no_cache 0;

    if ($arg_skipcache = 1) {
        set $no_cache 1;
    }

    server {
        location ~ \.php$ {
            fastcgi_cache_bypass $no_cache;
            fastcgi_no_cache $no_cache;
        }
    }
}
```

## [HTTP2](https://en.wikipedia.org/wiki/HTTP/2)
To enable HTTP2 for nginx server, you have to rebuild nginx with dynamic module. Make sure you follow the built step on `Add nginx dynamic modules`, then build nginx with `--with-http_v2_module` and `--with-http_ssl_module` (You would need to enable SSL for HTTP2 to work).

You can generate temporary private and public key for testing by openssl.

```bash
openssl req -x509 -days 10 -nodes -newkey rsa:2048 -keyout /etc/nginx/ssl/self.key -out /etc/nginx/ssl/self.crt
```

Then enable SSL on the nginx config.

```nginx
http {
    # enable ssl and http2
    listen 443 ssl http2;

    ssl_certificate /etc/nginx/ssl/self.crt;
    ssl_certificate_key /etc/nginx/ssl/self.key;
}
```

## Server push

To enable requesting for the resource in a same request like index.html. You would require server push enabled.

```nginx
http {
    location = /index.html {
        http2_push /style.css;
        http2_push /thumb.png;
    }
}
```

## Optimise HTTPS

To redirect all http requests to https, you can create another `server` directive and do the redirect to https server.

```nginx
server {
    listen 80;
    server_name 10.0.0.56;
    return 301 https://$host$request_uri;
}
```

Additionally, you can fine tune TLS configuration on nginx to improve its performance.

```nginx
server {
    # Disable SSL protocol
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

    # Optimise cipher suits
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

    # Enable DH params
    # https://wiki.openssl.org/index.php/Diffie-Hellman_parameters
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # Enable HSTS (Strict transport security)
    # This is the header to tell browser not to load any thing from http
    add_header Strict-Transport-Security "max-age=31536000" always;

    # SSL sessions
    # to caching handshake data on SSL and improve connection time
    ssl_session_cache shared:SSL:40m;
    ssl_session_timeout 4h;
    ssl_session_tickets on;
}
```

To generate dhparam,

```bash
openssl dhparam 2048 -out /etc/nginx/ssl/dhparam.pem
```

## Rate limiting

To define rate limiting, you can specify it as below.

```nginx
http {
    # Define limit zone
    # limit_req_zone $binary_remote_addr zone=MYZONE:10m rate=60r/m;
    limit_req_zone $request_uri zone=MYZONE:10m rate=60r/m;

    server {
        # ...

        location / {
            # burst define the buffer for connection in nginx
            # supposing that we have a rate limiting of 1r/s,
            # when you send 6 requests at a same time, only 1
            # request get process by upstream server, and the
            # remained 5 requests will have to wait.
            # limit_req zone=MYZONE burst=5;

            # when specify nodelay, it will allow the burst of 5 requests
            # to be sent to upstream and return
            limit_req zone=MYZONE burst=5 nodelay;
            try_files $uri $uri/ =404;
        }
    }
}
```

`$binary_remote_addr` will do the rate limiting based on user ip.

## Basic Auth

For basic auth to work, you have to generate password using `httpd-tools` or `apache2-utils` first.

```bash
htpasswd -c /etc/nginx/.htpasswd user1
```

Then

```nginx
http {
    server {
        location / {
            auth_basic "Secure Area";
            auth_basic_user_file /etc/nginx/.htpasswd;
            try_files $uri /$uri =404;
        }
    }
}
```

## Hardening nginx

```nginx
http {
    # hide nginx version for security reason
    server_tokens off;

    server {
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
    }
}
```

## Let's Encrypt with nginx

Supposing we have a simple configuration like this.

```nginx
http {
    server {
        listen 80;
        # certbot will use this info when inspect your nginx.conf
        server_name yourdomain.com;

        location / {
            return 200 "Hello from NGINX";
        }
    }
}
```

For Let's Encrypt to work, firstly go to https://certbot.eff.org/ and follow the guide there
to set up auto renewal for your certificate. Sth like this as an example:
https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx

Then, execute:

```bash
certbot --nginx
```

The command will try to do some modifications to your nginx config. Sth similar like this

```nginx
http {
    server {
        listen 443 ssl;
        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssh_dhparam /etc/letsencrypt/ssh-dhparams.pem;
    }
}
```

To renew your certificate, run:

```bash
certbor renew
```
