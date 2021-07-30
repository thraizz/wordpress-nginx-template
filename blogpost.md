# WordPress Docker behind NGINX Reverse Proxy

## Setting up the containers

Look at the `docker-compose.yml` first and be surprised by how easy the setup is:

```yaml
version: "3"
services:
  proxy: 
    image: nginx:latest
    container_name: production-nginx
    volumes:
      - /etc/letsencrypt/:/etc/letsencrypt/
      - ./nginx/:/etc/nginx/
    ports:
      - "80:80"
      - "443:443"
	wp:
    image: wordpress:latest
    restart: always
    container_name: production-wordpress
    environment:
      WORDPRESS_DB_HOST: production-db:3306
      WORDPRESS_DB_USER: dbuser
      WORDPRESS_DB_PASSWORD: dbpassword
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - ./wordpress:/var/www/html
  db:
    image: mysql:5.7
    container_name: production-db
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: dbuser
      MYSQL_PASSWORD: dbpassword
      MYSQL_RANDOM_ROOT_PASSWORD: "1"
    volumes:
      - ./mysql:/var/lib/mysql
```

As you can see, we need three running containers for our goal. WordPress and its database plus NGINX for the reverse proxy. Unsure what that means? Well, visitors cannot reach your WordPress site directly, as it does not expose any ports. The proxy server will redirect requests over the internal docker network to the right virtual server. This allows one host to serve multiple services or websites, each resolved through our proxy. Other advantages are good caching and SSL encryption, but more on that later.

## Configuring NGINX

Once again, we start with a minimal `nginx.conf` example:

```docker
events {}
http {
  server_names_hash_bucket_size  64;
  error_log /etc/nginx/error_log.log debug;
  client_max_body_size 20m;
  proxy_cache_path /etc/nginx/cache keys_zone=one:500m max_size=1000m;

  # Redirect all http requests to https
  server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
  }

  # Redirect requests from https://www.example.com to https://example.com
  server {
    server_name www.example.com;
    listen 443 ssl;
    # Include SSL certificates managed by certbot
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # Include letsencrypt standards
    include /etc/letsencrypt/options-ssl-nginx.conf;
    # Include proxy settings for docker/wordpress
    include /etc/nginx/includes/proxy.conf;
    # Use docker DNS for proxy resolving
    resolver 127.0.0.11;
    # https://www.example.com -> https://example.com
    rewrite ^/(.*) https://example.com/$1 permanent;
  }
  server {
    server_name example.com;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    include /etc/nginx/includes/proxy.conf;
    resolver 127.0.0.11;

    index index.php;
    try_files $uri $uri/ =404;
  
    # This must match your container_name set in docker-compose.yml!
    set $wordpress http://production-wordpress;

    location / {
      proxy_set_header    X-Forwarded-Host   $host;
      proxy_set_header    X-Forwarded-Server $host;
      proxy_set_header    X-Forwarded-For    $proxy_add_x_forwarded_for;
      proxy_set_header    X-Real-IP          $remote_addr;
      proxy_set_header    Host               $host;
      proxy_set_header    X-Forwarded-Proto https;
      proxy_pass $wordpress;
    }
  }
}
```

Notable are the caching options that should work well for you as well, the SSL configuration which requires a working certificate (I manage mine through certbot, I might write another post about that), the docker-internal proxy resolver at 127.0.0.11 which allows us to use container names as valid hostnames and the `proxy_set_header` configuration in our main locaation block.
These settings are the absolute minimum to get this example to work. I recommend you to have a  look at the way `sites-available` and `sites-enabled` work, which is basically putting server blocks into their own files and linking them from `nginx/sites-available` to `ngix/sites-enabled`,  which helps with keeping a clean nginx.conf when your configuration advances and you run multiple virtual server blocks.
Also be sure to replace [example.com](http://example.com) with your domain.

## Running the whole thing

As everything is configured now (yes, really, we don't need to configure anything for WordPress or MySQL) we can start the whole unit with `docker-compose up -d`, which will pull all images for the containers, create the services, start them and detach from their output.

Normally you should see everything starting, the last lines would be something around

```bash
[...]
Starting production-nginx     ... done
Starting production-wordpress ... done
Starting production-db        ... done
```

But if you run into any errors you should check the logs of the service with `docker-compose logs` and the content of `nginx/error_log.log` which will hold a lot of important informations about your proxy setup.

You might also get errors because of your  SSL certificates/Lets Encrypt - I really recommend to setup these over certbot. Interrested in learning how? Leave a comment and I'll write a post for this.

If everything went fine - nice! Check your domain to see the WordPress initalization setup and leave a comment that everything worked.
