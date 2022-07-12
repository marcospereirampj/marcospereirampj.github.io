---
layout: post
title:  "Using NGINX as API Gateway"
date:   2022-07-08 21:00:00 +0300
tags: [nginx,devops,api gateway,microservices]
categories: microservices
image:  
    path: /assets/img/posts/nginx.jpeg
---

Hi everyone!

If you ever need to deploy a reverse proxy, you may have heard of NGINX (engine x). In case you haven't heard it yet, let's talk a little about it here and how we can use it as an API Gateway.

# What is NGINX?

NGINX is an HTTP server and reverse proxy, a mail proxy server, and a generic TCP/UDP proxy server.

According to Netcraft, NGINX served or proxied [21.67% of the busiest sites in May 2022](https://news.netcraft.com/archives/2022/05/30/may-2022-web-server-survey.html).

Currently, NGINX has an open-source version and a commercial version (NGINX Plus). In this post, we will try to use some of the features of the free version but also available in the paid version.

# What is an API Gateway?

An API Gateway is a traffic manager that allows developers to create, publish, maintain, monitor, and secure APIs. In resume, it interfaces between external traffic and backend services.

Using an API Gateway has several advantages, such as:

- Make APIs more secure through a single interface.
- Enable you to more easily enforce access control policies, rate limits, routing, mediation, etc.
- Enables the most comprehensive collection of metrics 

That said, let's put some concepts into practice!


# What do we want?

Let's say we have two APIs: a for products, which we call the **Products API**, and other for users, which we call the **Users API**. Initially, we will publish our APIs. Our architecture diagram starts like this:

![NGINX as API Gateway - First Step](/assets/img/posts/nginx_gateway_1.png)

We want to create the following routes:

- `/api/products` -> should point to the Products API service
- `/api/users` -> should point to the Users API service

A relevant feature in our settings is that we don't want to expose our services publicly. Therefore, our only gateway should be the NGINX which we call API Gateway in the diagram above.

To simulate our API, we built two simple Python applications with fixed responses ([Products API](https://github.com/marcospereirampj/nginx-api-gateway/tree/master/backends/products) and [Users API](https://github.com/marcospereirampj/nginx-api-gateway/tree/master/backends/users)).

For that, let's create a simple rule in our NGINX configuration file (`gateway.conf`):

```nginx
server {
   listen 80 default_server;
   listen [::]:80 default_server;
 
   #
   # Products API
   #
   location /api/products {
       proxy_pass http://products_api:8001;
   }
 
   #
   # Users API
   #
   location /api/users {
       proxy_pass http://users_api:8002;
   }
}

```

In the configuration above, notice that I created an alias for the API containers (see `docker-compose.yml` - contains spoilers).

So, the expected answers are:

```
GET http://localhost/api/products

{
  "name": "Product 1",
  "description": "Detail about product 1"
}

```

```
GET http://localhost/api/users
{
  "name": "User 1",
  "email": "email@email.com"
}

```

In this way, we have NGINX serving as a reverse proxy. To improve the organization of our configurations we will define `upstream` for our API instead of directly using their address (alias). So we will have:


```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
       server users_api:8002;
}
```

And after:

```nginx
server {
   listen 80 default_server;
   listen [::]:80 default_server;
   #
   # Products API
   #
   location /api/products {
       proxy_pass http://products_api_server;
   }
 
   #
   # Users API
   #
   location /api/users {
       proxy_pass http://users_api_server;
   }
}
```

The `upstream` (`ngx_http_upstream_module`) is used to define groups of servers that can be referenced by the `proxy_pass`, `fastcgi_pass`, `uwsgi_pass`, `scgi_pass`, `memcached_pass`, and `grpc_pass directives`.

After applying our basic settings, let's increment the Gateway.

# Load Balance

In the next scenario, let's assume that our Users API got overloaded and we decided to add a new instance (`users_api_balance`) to receive requests from end-users.

![NGINX as API Gateway - First Step](/assets/img/posts/nginx_gateway_2.png)

We will configure our API Gateway to do this balancing. For that, we'll add the new server to the Users API server group.

```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
       server users_api:8002;
       server users_api_balance:8002;
}
```

Through the container logs, you can see that the requests are distributed among the servers.

```sh
api-gateway-nginx    | 172.18.0.1 - - [07/Jul/2022:22:52:49 +0000] "GET /api/users HTTP/1.1" 200 46
container_users_api  | 172.18.0.5 - - [07/Jul/2022 22:52:49] "GET /api/users HTTP/1.0" 200
api-gateway-nginx    | 172.18.0.1 - - [07/Jul/2022:22:52:49 +0000] "GET /api/users HTTP/1.1" 200 46
container_users_api_balance | 172.18.0.5 - - [07/Jul/2022 22:52:50] "GET /api/users HTTP/1.0" 200
```

With these settings, the requests are distributed evenly (Round Robin) between servers. But how about we evolve our balancing rules? Now let's define that we should direct the new request to the server with the lowest number of active connections. Again, we will only modify the Users API server group by passing the desired balancing method. In our case, it will be the `least_conn`.

```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
least_conn;
       server users_api:8002;
       server users_api_balance:8002;
}
```

We'll use this setup for now, but NGINX has a few other balancing methods:

- ip_hash:
- Generic Hash
- Random
- Least Time (NGINX Plus only)

More details in the [official documentation](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/).

However, let's assume tha the `users_api` server has a better infrastructure, and we want to prioritize requests to this server.  We can do this through the `weight` parameter.

```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
least_conn;
       server users_api:8002 weight=5;
       server users_api_balance:8002;
}

```

With that, the `users_api` server has a priority five times higher than the other servers.


# Cache

In this scenario, let's assume that the Products API is not volatile, and we want to optimize for response time. For this, we will add a basic cache in API Gateway,

Only two directives are needed to enable basic caching: `proxy_cache_path` and `proxy_cache`. The `proxy_cache_path` directive sets the path and configuration of the cache, and the `proxy_cache` directive activates it.


```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
       ip_hash;
       server users_api:8002;
       server users_api_balance:8002;
}
 
 
proxy_cache_path /tmp/products levels=1:2 keys_zone=products_cache:10m max_size=10g inactive=60m use_temp_path=off;
 
server {
   listen 80 default_server;
   listen [::]:80 default_server;
 
   #
   # Products API
   #
   location /api/products {
       proxy_cache products_cache;
       proxy_pass http://products_api_server;
   }
 
   #
   # Users API
   #
   location /api/users {
       proxy_pass http://users_api_server;
   }
}

```

The parameters to the `proxy_cache_path` directive define the following settings:

- The local disk directory for the cache is called `/tmp/products`.
- `levels `sets up a two‑level directory hierarchy under `/tmp/products`. If the `levels` parameter is not included, NGINX puts all files in the same directory.
- `keys_zone` sets up a shared memory zone for storing the cache keys and metadata such as usage timers.
- `max_size` sets the upper limit of the size of the cache (to 10 gigabytes in this example). It is optional; not specifying a value allows the cache to grow to use all available disk space. 
- `inactive` specifies how long an item can remain in the cache without being accessed. In this example, a file that has not been requested for 60 minutes is automatically deleted from the cache by the cache manager process, regardless of whether or not it has expired. The default value is 10 minutes (10m).
- NGINX first writes files that are destined for the cache to a temporary storage area, and the `use_temp_path=off` directive instructs NGINX to write them to the same directories where they will be cached.

The default form of the keys that NGINX generates is similar to an MD5 hash of the following NGINX variables: `$scheme$proxy_host$request_uri;` the actual algorithm used is slightly more complicated.

More details on caching can be seen on the NGINX Blog: [A Guide to Caching with NGINX and NGINX Plus.](https://www.nginx.com/blog/nginx-caching-guide/)

# Rate Limit

Having defined our balancing and cache rules, we can now protect our internal services from a large number of requests (DDoS attacks) by setting a rate limit for consumers.

Rate limiting is configured with two main directives, `limit_req_zone` and `limit_req`. The `limit_req_zone` directive defines the parameters for rate limiting while `limit_req` enables rate limiting within the context where it appears. 


```nginx
upstream products_api_server {
       server products_api:8001;
}
 
upstream users_api_server {
       ip_hash;
       server users_api:8002;
       server users_api_balance:8002;
}
 
 
proxy_cache_path /tmp/products levels=1:2 keys_zone=products_cache:10m max_size=10g inactive=60m use_temp_path=off;
limit_req_zone $binary_remote_addr zone=products_rate:10m rate=1r/s;
limit_req_zone $binary_remote_addr zone=user_rate:10m rate=1r/s;
 
 
server {
   listen 80 default_server;
   listen [::]:80 default_server;
 
   #
   # Products API
   #
   location /api/products {
       proxy_cache products_cache;
       limit_req zone=products_rate;
       limit_req_status 429;
       proxy_pass http://products_api_server;
   }
 
   #
   # Users API
   #
   location /api/users {
       limit_req zone=user_rate;
       limit_req_status 429;
       proxy_pass http://users_api_server;
   }
}
```

`limit_req_zone` takes the following three parameters:

- Key – Defines the request characteristic against which the limit is applied. In the example it is the NGINX variable `$binary_remote_addr`, which holds a binary representation of a client’s IP address. This means we are limiting each unique IP address to the request rate defined by the third parameter.
- Zone – Defines the shared memory zone used to store the state of each IP address and how often it has accessed a request‑limited URL. Keeping the information in shared memory means it can be shared among the NGINX worker processes.
- Rate – Sets the maximum request rate. In the example, the rate cannot exceed 1 request per second for Users API and 10 requests per second for Products API.


By default, NGINX will return 503 (Service Temporarily Unavailable) when the request limit is reached. To improve this, we use `limit_req_status` to customize the response status code. In this case we use 429 (Too Many Requests).

More details on using rate limit can be seen on the NGINX Blog: [Rate Limiting with NGINX and NGINX Plus](https://www.nginx.com/blog/rate-limiting-nginx/)

# API Key Authentication

It is unusual to publish APIs without some form of authentication to protect them. NGINX offers several approaches for protecting APIs and authenticating API clients. In our solution, we will use a simple solution to validate access to our services. We will use [API Keys Authentication](https://blog.stoplight.io/api-keys-best-practices-to-authenticate-apis).

With API key authentication, we use a map block to create an allowlist of client names that can access our services.

```nginx
map $http_apikey $api_client_name {
    default       "";
    "KrtKNkLNGcwKQ56la4jcHwxF"  "client_one";
    "sqj3Ye0vFW/CM/o7LTSMEMM+"  "client_two";
    "diXnbzglAWMMIvyEEV3rq7Kt"  "client_ten";
}
```

The map directive takes two parameters. The first defines where to find the API key, in this case in the `apikey` HTTP header of the client request as captured in the `$http_apikey` variable. The second parameter creates a new variable (`$api_client_name`) and sets it to the value of the second parameter on the line where the first parameter matches the key.

Now enable API Key authentication on our services. Just to avoid code duplication I will separate the API Key validation into another method.

```nginx
# API key validation
location = /_validate_apikey {
    internal;

    if ($http_apikey = "") {
        return 401; # Unauthorized
    }
    if ($api_client_name = "") {
        return 403; # Forbidden
    }

    return 204; # OK (no content)
}
```

```nginx
#
# Products API
#
location /api/products {
    auth_request /_validate_apikey;
    proxy_cache products_cache;
    limit_req zone=products_rate;
    limit_req_status 429;
    proxy_pass http://products_api_server;
}

#
# Users API
#
location /api/users {
    auth_request /_validate_apikey;
    limit_req zone=user_rate;
    limit_req_status 429;
    proxy_pass http://users_api_server;
}
```

With this configuration in place, the Products API and Users APIs now implements API key authentication.

```sh
curl -I 'http://localhost/api/products' --header 'apikey: INVALID_KEY'
HTTP/1.1 403 Forbidden
Server: nginx/1.21.5
Date: Sat, 09 Jul 2022 02:24:03 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive
```

```sh
curl -I 'http://localhost/api/products' --header 'apikey: diXnbzglAWMMIvyEEV3rq7Kt'
HTTP/1.1 200 OK
Server: nginx/1.21.5
Date: Sat, 09 Jul 2022 02:25:03 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 62
Connection: keep-alive
```

Note that the above solution is quite limited, but it is possible to [authenticate using JWT](https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/). However, this functionality is natively only available in NGINX Plus. For NGINX open source, an introspection endpoint is required to perform the validation.

Other more robust solutions are using that can be used for authentication, are: [OAuth Proxy Module](https://curity.io/resources/learn/nginx-oauth-proxy-module/) or [Phantom Token Module](https://curity.io/resources/learn/nginx-phantom-token-module/).

# Conclusion

Finally, our final architecture is shown in the diagram below. [Here](https://github.com/marcospereirampj/nginx-api-gateway) you can find all the codes we developed.

![NGINX as API Gateway - First Step](/assets/img/posts/nginx_gateway_3.png)

In addition to the features presented in this article, NGINX offers several others (especially in NGINX Plus). Several of them can be found in the free [ebook](https://www.nginx.com/resources/library/nginx-api-gateway-deployment/) or on the tool's [official blog](https://www.nginx.com/blog/).

Well, that's it, folks. I hope you enjoyed the article. See you soon!
