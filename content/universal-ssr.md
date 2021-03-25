+++
title = "Universal server-side rendering for better SEO"
date = 2021-03-26T16:00:00Z

[taxonomies]
categories = []
tags = ["SEO", "SPA", "SSR", "JavaScript", "Nginx", "Docker", "Docker compose"]
+++

Universal approach to *server-side rendering (SSR)* for sites that rely on JavaScript for client-side rendering.

<!-- more -->
<hr/>

## Why server-side rendering?

In short, client-side rendered pages pose a difficulty for search engines. This can harm the discoverability and ranking of your site on search engines. Checkout [**How client-side rendering hinders SEO**](/client-side-rendering-seo) to learn more.

## Server-side rendering approaches 

Many popular frontend JavaScript frameworks come with their own ways of implementing server ide rendering:

* [React](https://reactjs.org) comes with [ReactDOMServer](https://reactjs.org/docs/react-dom-server.html).
* [Vue](https://vuejs.org) comes with [vue-server-renderer](https://ssr.vuejs.org).
* [Angular](https://angular.io) comes with [Angular Universal](https://angular.io/guide/universal).

If you are using one of these frameworks, it might be worth investigating these approaches instead of the universal approach I am going to present here.

Advantages of the universal approach:
* Works with any site, regardless of framework.
* Does not require rewriting your code to work in Node (getting rid of `document`, `window`, etc.).

Disadvantages of the universal approach:
* More resource intensive and potentially higher latency.
* No implicit support for [hydration](https://en.wikipedia.org/wiki/Hydration_(web_development)), this means that you are going to lose interactivity and as such *it's in most cases only useful for search engine crawlers, not for regular users of your website*.

## Using Prerender for universal server-side rendering

[*Prerender*](https://github.com/prerender/prerender) is a Node server that uses headless Chrome to render websites, much like search engines [such as Google](http://nginx.org/en/docs/beginners_guide.html).
For our set-up we will use [Nginx](https://www.nginx.com) as a reverse proxy and [docker-compose](https://docs.docker.com/compose/) to tie it all together.


### Nginx set-up

How does it work?

1. Inspect the request and decide whether we want to prerender the response
    * This logic is based mostly on the `User-Agent` request header.
2. *If prerender*: Pass the request to the "prerender" service, which proxies it to the "server" service.
3. *If not prerender*: Pass the request directly to the "server" service.

The Nginx config assumes that the Prerender service is running at `http://prerender:3000` and that your website's server (that you want to enable server-side rendering for) is running at `http://server:10080`.

```conf
events {
}

http {
  # Handle http://example.com
  # - Prerender for scrapers
  server {
      listen 80;
      server_name example.com;

      location / {
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header        X-Forwarded-Proto $scheme;

          # Logic that decides whether we should use pre-render or not.
          # It is based mainly on the user agent, as shown here:
          # https://gist.github.com/thoop/8165802

          set $prerender 0;
          # Prerender for bot user agents.
          if ($http_user_agent ~* "googlebot|bingbot|yandex|baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest\/0\.|pinterestbot|slackbot|vkShare|W3C_Validator|whatsapp") {
              set $prerender 1;
          }
          if ($args ~ "_escaped_fragment_") {
              set $prerender 1;
          }
          if ($http_user_agent ~ "Prerender") {
              set $prerender 0;
          }
          # Do not prerender for plain resources.
          if ($uri ~* "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff|svg|eot)") {
              set $prerender 0;
          }
         
          if ($prerender = 1) {
              rewrite .* /http://server:10080$request_uri? break;
              proxy_pass http://prerender:3000;
          }
          if ($prerender = 0) {
              proxy_pass http://server:10080;
          }
      }
  }
}
```

In most cases, you will want to also handle HTTPS traffic. For that, refer to [Handling HTTPS with Nginx](/https-nginx).
Here's what a config that does both prerendering as well as HTTPS handling could look like:


```conf
events {
}

http {
  # Handle https://example.com
  #
  # - SSL
  # - Prerender for scrapers
  server {
      listen 443 ssl;
      server_name example.com;

      ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

      ssl_session_timeout 1d;
      ssl_session_cache builtin:100 shared:SSL:10m;
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
      ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
      ssl_prefer_server_ciphers on;

      location / {
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header        X-Forwarded-Proto $scheme;

          add_header Content-Security-Policy "default-src 'none';connect-src 'self' www.google-analytics.com;manifest-src 'self';style-src 'self';script-src 'self' www.googletagmanager.com;img-src 'self' www.googletagmanager.com;frame-ancestors 'self';base-uri 'none';";
          add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
          add_header X-Content-Type-Options nosniff;
          add_header X-Frame-Options SAMEORIGIN;
          add_header X-XSS-Protection "1; mode=block";

          set $prerender 0;
          if ($http_user_agent ~* "googlebot|bingbot|yandex|baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest\/0\.|pinterestbot|slackbot|vkShare|W3C_Validator|whatsapp") {
              set $prerender 1;
          }
          if ($args ~ "_escaped_fragment_") {
              set $prerender 1;
          }
          if ($http_user_agent ~ "Prerender") {
              set $prerender 0;
          }
          if ($uri ~* "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff|svg|eot)") {
              set $prerender 0;
          }
          
          if ($prerender = 1) {
              rewrite .* /http://server:10080$request_uri? break;
              proxy_pass http://prerender:3000;
          }
          if ($prerender = 0) {
              proxy_pass http://server:10080;
          }
      }
  }
  
  # Redirect http://example.com and http://www.example.com to https://example.com
  server {
      listen 80;
      server_name www.example.com example.com;
      return 301 https://example.com$request_uri;
  }

  # Redirect https://www.example.com to https://example.com
  server {
      listen 443;
      server_name www.example.com;
      
      ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

      ssl_session_timeout 1d;
      ssl_session_cache builtin:100 shared:SSL:10m;
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
      ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
      ssl_prefer_server_ciphers on;
      
      return 301 https://example.com$request_uri;
  }
}
```

### Docker-compose set-up 

We use `docker-compose` to tie it all together. It runs the Nginx service, the Prerender service and optionally also the server you want server-side rendering for: 

```yaml
version: "3.7"
services:
  # You would put the service for your website's server here.
  # If you do not use Docker for your website's server, you should remove this. 
  server:
    # We assume that the "server" service is internally exposed at the port 10080.
    expose:
      - "10080"
  nginx:
    image: nginx:latest
    volumes:
      # This should be the location of the nginx.conf above.
      - ./nginx.conf:/etc/nginx/nginx.conf
      # This assumes you use Let's Encrypt for your SSL certificate.
      - /etc/letsencrypt/:/etc/letsencrypt
    ports:
      # The Nginx server is exposed at port 80 and 443 for HTTP and HTTPS traffic respectively.
      - "80:80"
      - "443:443"
    links:
      # Remove this if you do not use the "server" service for your website's server. 
      - "server"
    # This allows you to connect to localhost of the host machine.
    # Only needed if your website's server runs outside of this docker-compose
    # (i.e. you do not use the "server" service above).
    extra_hosts:
      - "host.docker.internal:host-gateway"
  prerender:
    image: tvanro/prerender-alpine:latest
    environment:
      MEMORY_CACHE: 0
      CACHE_MAXSIZE: 0
    expose:
      - "3000"
    # This allows you to connect to localhost of the host machine.
    # Only needed if your website's server runs outside of this docker-compose
    # (i.e. you do not use the "server" service above).
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

If you do not want to use Docker for your website's server that's fine. Assuming it's running on the localhost of your host machine, e.g. `http://localhost:10080`, you have several options, but the easiest would be changing `http://server:10080` in `nginx.conf` to `http://host.docker.internal:10080`.

You can configure the prerender service using environment variables. In the above, I disabled caching because in my case it's unlikely that the same page will be retrived by crawlers more than once in a short period of time. See the documentation of [the Docker image](https://github.com/tvanro/prerender-alpine) for details.

Now you can simply run `docker-compose up` to start up all the necessary services.

## Resources

* [Nginx guide](http://nginx.org/en/docs/beginners_guide.html)
* [Docker guide](https://docs.docker.com/get-started/)
* [Docker-compose guide](https://docs.docker.com/compose/gettingstarted/)
