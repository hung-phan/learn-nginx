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
