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
Rewrite and redirect have the same purpose but it will have a very different result.

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

## nginx worker
By default nginx only spawns 1 instance of nginx worker. To config it to spawn more workers, use `worker_process` on the root context.

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
