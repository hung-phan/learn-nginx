# nginx cheatsheet

## Location
nginx location will follow strictly this order.

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
NgINX default will have a default list of variables. Check out it [here](https://nginx.org/en/docs/varindex.html).

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
